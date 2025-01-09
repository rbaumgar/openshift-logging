# OpenShift Logging QuickStart Guide with Loki

![](images/monitor.jpg)

*By Robert Baumgartner, Red Hat Austria, Janurary 2025 (OpenShift 4.17, OpenShift Logging 6.1)*

In this blog, I will guide you on

- How to install Loki as storage for OpenShift Logging using MinIO as object store

- How to install OpenShift Logging using the Loki stack

- How to install OpenShift console UIPlugin and use it

This document is based on OpenShift 4.17. See [Configuring and using logging in OpenShift Container Platform](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/logging/index).

## Install MinIO (optional)
Loki requires an object storage. So if you have already an object storage available you can skip this chapter.

### Create a New MinIO Project

Create a new project (for example minio) :

```shell
$ oc new-project minio
Now using project "minio" on server "https://api.ocp4.openshift.freeddns.org:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app rails-postgresql-example

to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=registry.k8s.io/e2e-test-images/agnhost:2.43 -- /agnhost serve-hostname
```

### Create MinIO Objects

```shell
# create minio admin password
MINIO_ADMIN_PWD=`openssl rand -base64 12`
# cat minio/secret-minio.yaml
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  annotations: {}
  name: minio-access-secrets
stringData:
  minioSecret: $MINIO_ADMIN_PWD
  minioAccessKey: minio
  minioConsoleAddress: ":44645"
EOF

oc apply -f minio/pvc-minio.yaml

oc apply -f minio/svc-minio.yaml

oc apply -f minio/deployment-minio.yaml

oc apply -f minio/route-minio.yaml
```

### Create a bucket and user for the lokistack


```shell
# create loki user secret
LOKI_USER_SECRET=`openssl rand -base64 12`

# create an alias for the connection
$ oc rsh deployments/minio-server \
         mc alias set myminio http://localhost:9000 minio $MINIO_ADMIN_PWD
Added `myminio` successfully.

# copy policy file to pod
$ cat minio/openshift-logging-access-policy.json | \
  oc exec deployments/minio-server -i -- sh -c "cat /dev/stdin > /tmp/openshift-logging-access-policy.json"

# create new policy
$ oc rsh deployments/minio-server \
         mc admin policy create myminio openshift-logging-access-policy /tmp/openshift-logging-access-policy.json

# cleanup: remove policy file
$ oc rsh deployments/minio-server rm /tmp/openshift-logging-access-policy.json

# (optional) create list all policies
$ oc rsh deployments/minio-server \
         mc admin policy ls myminio
readonly
readwrite
writeonly
consoleAdmin
diagnostics
openshift-logging-access-policy

# create openshift-logging bucket
$ oc rsh deployments/minio-server \
         mc mb myminio/openshift-logging
Bucket created successfully `myminio/openshift-logging`.

# (optional) list all buckets
$ oc rsh deployments/minio-server \
         mc ls myminio
[2025-01-09 11:28:51 UTC]     0B openshift-logging/

# create user
$ oc rsh deployments/minio-server \
         mc admin user add myminio openshift-logging $LOKI_USER_SECRET
Added user `openshft-logging` successfully.

# (optional) list all users
$ oc rsh deployments/minio-server \
         mc admin user ls myminio
Added user `openshft-logging` successfully.
enabled    openshift-logging

# attach openshift-logging-acccess-policy to user openshift-logging
$ oc rsh deployments/minio-server \
         mc admin policy attach myminio openshift-logging-access-policy --user openshift-logging

# (optional) if you want to delete an user
$ oc rsh deployments/minio-server \
         mc admin user rm myminio user-to-delete        
Removed user `user-to-delete` successfully.

# (optional) if you want displays information on a MinIO server
$ oc rsh deployments/minio-server \
         mc admin info myminio
●  localhost:9000
   Uptime: 1 week 
   Version: 2024-12-18T13:15:44Z
   Network: 1/1 OK 
   Drives: 1/1 OK 
   Pool: 1

┌──────┬────────────────────────┬─────────────────────┬──────────────┐
│ Pool │ Drives Usage           │ Erasure stripe size │ Erasure sets │
│ 1st  │ 95.3% (total: 750 GiB) │ 1                   │ 1            │
└──────┴────────────────────────┴─────────────────────┴──────────────┘

16 GiB Used, 2 Buckets, 18,610 Objects
1 drive online, 0 drives offline, EC:0 

# cleanup: remove the alias for the connection
$ oc rsh deployments/minio-server \
         mc alias remove myminio
Removed `myminio` successfully.
```

You can login to the MinIO console with minio/$MINIO_ADMIN_PWD

```shell
# get MinIO console URL
$ oc get route minio-console -o jsonpath='{.spec.host}'
minio-console-minio.apps.rbaumgar.demo.net
```

Keep in mind if you have networkpolicies in use, allow the project openshift-logging access to the project minio on port 9000.

## Install Loki Operator

Loki Operator supports AWS S3, Azure, GCS, Minio, OpenShift Data Foundation and Swift for LokiStack object storage.

If you are using a different object store you might need to define the secret in a different way.
See https://github.com/grafana/loki/blob/main/operator/docs/lokistack/object_storage.md

```sh
kubectl create secret generic lokistack-minio -n openshift-logging\
  --from-literal=bucketnames="lokistack" \
  --from-literal=endpoint="http://minio.minio.svc:9000" \
  --from-literal=access_key_id="loki" \
  --from-literal=access_key_secret="$LOKI_USER_SECRET"
```

The endpoint consists= <svc>.<project>.svc:9000.

Loki supports different preconfigured sizes.

The 1x.demo configuration defines a single Loki deployment with minimal resource and limit requirements, no high availability (HA) support for all Loki components. This configuration is suited for demo environmnt.

The 1x.pico configuration defines a single Loki deployment with minimal resource and limit requirements, offering high availability (HA) support for all Loki components. This configuration is suited for deployments that do not require a single replication factor or auto-compaction.

Other available sizes are 1x.extra-small, 1x.small, 1x.medium.
See https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html-single/logging/index#log6x-loki-sizing_log6x-loki-6.1

In the operators/loki/lokistack.yaml the spec.size is defined as 1x.demo. 

To save space on the lokistack retention period is set to 3 days. Maximum supported retention period is 30 days.

```sh
oc apply -f operators/loki/operator-loki.yaml

oc apply -f operators/loki/lokistack.yaml
```

```sh
$ oc get lokistacks.loki.grafana.com -o custom-columns=Status:status.conditions[0].message
Status
All components ready

$ oc get pod -l app.kubernetes.io/instance=logging-loki
NAME                                           READY   STATUS    RESTARTS   AGE
logging-loki-compactor-0                       1/1     Running   0          3d20h
logging-loki-distributor-55f9475cc6-jdhjb      1/1     Running   0          3d20h
logging-loki-gateway-778c49b997-dfzjc          2/2     Running   0          3d20h
logging-loki-gateway-778c49b997-k58v8          2/2     Running   0          4d3h
logging-loki-index-gateway-0                   1/1     Running   0          3d20h
logging-loki-ingester-0                        1/1     Running   0          3d20h
logging-loki-querier-55d7fbb758-222dw          1/1     Running   0          3d20h
logging-loki-query-frontend-754c4594c5-rmgww   1/1     Running   0          3d20h
```

When you select another size than 1x.demo multiple pods for the lokistack will be created.

## Install OpenShift Logging Operator

```sh
$ oc apply -f operators/logging/operator-logging.yaml

$ oc create sa collector

$ oc adm policy add-cluster-role-to-user cluster-logging-write-application-logs -z collector 
$ oc adm policy add-cluster-role-to-user cluster-logging-write-audit-logs -z collector
$ oc adm policy add-cluster-role-to-user cluster-logging-write-infrastructure-logs -z collector

$ oc adm policy add-cluster-role-to-user collect-application-logs -z collector
$ oc adm policy add-cluster-role-to-user collect-audit-logs -z collector
$ oc adm policy add-cluster-role-to-user collect-infrastructure-logs -z collector

$ oc apply -f operators/logging/clusterlogforwarder.yaml
```

```sh
$ oc get clusterlogforwarders.observability.openshift.io collector -o jsonpath='{.status.conditions}'|jq
[
  {
    "lastTransitionTime": "2025-01-03T09:41:48Z",
    "message": "",
    "reason": "ValidationSuccess",
    "status": "True",
    "type": "observability.openshift.io/ValidLokistackOTLPOutputs"
  },
  {
    "lastTransitionTime": "2025-01-03T14:57:32Z",
    "message": "permitted to collect log types: [application audit infrastructure]",
    "reason": "ClusterRolesExist",
    "status": "True",
    "type": "observability.openshift.io/Authorized"
  },
  {
    "lastTransitionTime": "2025-01-03T14:57:32Z",
    "message": "",
    "reason": "ValidationSuccess",
    "status": "True",
    "type": "observability.openshift.io/Valid"
  },
  {
    "lastTransitionTime": "2025-01-03T14:57:32Z",
    "message": "",
    "reason": "ReconciliationComplete",
    "status": "True",
    "type": "Ready"
  }
]

$ oc get clusterlogforwarders.observability.openshift.io collector --template='{{range .status.conditions}}{{printf "%s=%s, reason=%s, message=%s\n\n" .type .status .reason .message}}{{end}}'
observability.openshift.io/ValidLokistackOTLPOutputs=True, reason=ValidationSuccess, message=

observability.openshift.io/Authorized=True, reason=ClusterRolesExist, message=permitted to collect log types: [application audit infrastructure]

observability.openshift.io/Valid=True, reason=ValidationSuccess, message=

Ready=True, reason=ReconciliationComplete, message=


$ oc get pod -l app.kubernetes.io/instance=collector
NAME              READY   STATUS    RESTARTS   AGE
collector-588b9   1/1     Running   0          3d19h
collector-88vc5   1/1     Running   0          3d19h
collector-bpj84   1/1     Running   0          3d19h
collector-fnbgr   1/1     Running   0          3d19h
collector-s858j   1/1     Running   0          3d19h
collector-scv4z   1/1     Running   0          3d19h
```

In this example six collectors are running becuase one on each node. 3 control plain nodes and 3 worker nodes.

## Install Cluster Observability Operator

```sh
oc apply -f operators/coo/operator-coo.yaml

oc apply -f operators/coo/uiplugin-logging.yaml
```

## Create Event Router

The OpenShift Container Platform Event Router is a pod that watches Kubernetes events and logs them for collection by the logging. You must manually deploy the Event Router.

The Event Router collects events from all projects and writes them to STDOUT. The collector then forwards those events to the store defined in the ClusterLogForwarder custom resource (CR).

```sh
$ oc process -f eventrouter/eventrouter.yaml | oc apply -n openshift-logging -f -
serviceaccount/eventrouter created
clusterrole.rbac.authorization.k8s.io/event-reader created
clusterrolebinding.rbac.authorization.k8s.io/event-reader-binding created
configmap/eventrouter created
deployment.apps/eventrouter created
```

View the new Event Router pod:

```sh
$ oc get pods --selector  component=eventrouter -o name -n openshift-logging
pod/eventrouter-549f95c7d7-6fqqx
```

View the events collected by the Event Router:

```sh
$ oc logs deploy/eventrouter -n openshift-logging
{"verb":"ADDED","event":{"metadata":{"name":"redhat-operators-w8svm.1818dc6c594723df","namespace":"openshift-marketplace","uid":"6ba1e49b-082d-4e3d-a700-a9bcba9689b6","resourceVersion":"2810786987","creationTimestamp":"2025-01-08T23:46:54Z","managedFields":[{"manager":"kubelet","operation":"Update","apiVersion":"v1","time":"2025-01-08T23:46:54Z","fieldsType":"FieldsV1","fieldsV1":{"f:count":{},"f:firstTimestamp":{},"f:involvedObject":{},"f:lastTimestamp":{},"f:message":{},"f:reason":{},"f:reportingComponent":{},"f:reportingInstance":{},"f:source":{"f:component":{},"f:host":{}},"f:type":{}}}]},"involvedObject":{"kind":"Pod","namespace":"openshift-marketplace","name":"redhat-operators-w8svm","uid":"5f618733-bea2-422d-985f-00cd0afe18a1","apiVersion":"v1","resourceVersion":"2810786706","fieldPath":"spec.containers{registry-server}"},"reason":"Created","message":"Created container registry-server","source":{"component":"kubelet","host":"master-0"},"firstTimestamp":"2025-01-08T23:46:54Z","lastTimestamp":"2025-01-08T23:46:54Z","count":1,"type":"Normal","eventTime":null,"reportingComponent":"kubelet","reportingInstance":"master-0"}}

```

select infra pod eventrouter-xxxx

-- Uninstall Event Router

```sh
$ oc delete deployment, sa, cm eventrouter
deployment.apps "eventrouter" deleted
serviceaccount "eventrouter" deleted
configmap "eventrouter" deleted

$ oc delete clusterrolebinding event-reader-binding
clusterrolebinding.rbac.authorization.k8s.io "event-reader-binding" deleted
$ oc delete clusterrole event-reader 
clusterrole.rbac.authorization.k8s.io "event-reader" deleted
```

## View Logs as Admin

Administrator - Observe - Logs

- Application logs
Container logs generated by user applications running in the cluster, except infrastructure container applications.

- Infrastructure logs
Container logs generated by infrastructure namespaces: openshift*, kube*, or default, as well as journald messages from nodes.

- Audit logs
Logs generated by auditd, the node audit system, which are stored in the /var/log/audit/audit.log file, and logs from the auditd, kube-apiserver, openshift-apiserver services, as well as the ovn project if enabled.

{ log_type="infrastructure"} |= `Started crio` | json | log_source="node" | hostname="master-0": all entries from master-0

## View Logs as User

- Administrator - Workloads/Pods -> select Pod - Aggregated Logs

Keep in mind that the Loki stack has a retention period!

- Developer - Observe - Logs

All messages in this namespace.

## Uninstall Logging Stack

## PodDisruptionBudget

```sh
$ oc get pdb 
NAME                          MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
logging-loki-compactor        1               N/A               0                     4d17h
logging-loki-distributor      1               N/A               0                     4d17h
logging-loki-gateway          1               N/A               1                     4d17h
logging-loki-index-gateway    1               N/A               0                     4d17h
logging-loki-ingester         1               N/A               0                     4d17h
logging-loki-querier          1               N/A               0                     4d17h
logging-loki-query-frontend   1               N/A               0                     4d17h
```

see **Not able to drain a node when running LokiStack with size 1x.demo in RHOCP 4**, https://access.redhat.com/solutions/7058851