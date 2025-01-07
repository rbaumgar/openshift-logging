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

```shell
$ oc rsh deployments/minio-server mkdir /data/lokistack
sh-5.1$ 

mc alias set myminio http://localhost:9000 minio minio123
Added `myminio` successfully.

mc admin accesskey ls myminio
User: minio
  Access Keys:
    loki, expires: never, sts: false
    tempo, expires: never, sts: false

### Create a bucket for the lokistack

For the lokistack a bucket has to be created on the MinIO server. This can be done very simple by creating a directory within the /data folder.

```shell
$ oc rsh deployments/minio-server mkdir /data/lokistack
```

### Create an accesskey for the lokistack

```shell
# create loki user secret
LOKI_USER_SECRET=`openssl rand -base64 12`

# create an alias for the connection
oc rsh deployments/minio-server \
       mc alias set myminio http://localhost:9000 minio $MINIO_ADMIN_PWD
Added `myminio` successfully.

oc rsh deployments/minio-server \
       mc admin accesskey create myminio/ --access-key loki --secret-key $LOKI_USER_SECRET
Access Key: loki
Secret Key: ...
Expiration: NONE
Name: 
Description: 

# (optional) if you want to list all available accesskeys
oc rsh deployments/minio-server \
       mc admin accesskey ls myminio               
User: minio
  Access Keys:
    loki, expires: never, sts: false

# (optional) if you want to delete an accesskey
oc rsh deployments/minio-server \
       mc admin accesskey rm myminio loki              
Successfully removed access key `loki`.
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
oc apply -f operators/logging/operator-logging.yaml

oc adm policy add-cluster-role-to-user cluster-logging-write-application-logs -z collector 
oc adm policy add-cluster-role-to-user cluster-logging-write-audit-logs -z collector
oc adm policy add-cluster-role-to-user cluster-logging-write-infrastructure-logs -z collector

oc adm policy add-cluster-role-to-user collect-application-logs -z collector
oc adm policy add-cluster-role-to-user collect-audit-logs -z collector
oc adm policy add-cluster-role-to-user collect-infrastructure-logs -z collector

oc apply -f operators/logging/clusterlogforwarder.yaml
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

$ oc get pod -l app.kubernetes.io/instance=collector
NAME              READY   STATUS    RESTARTS   AGE
collector-588b9   1/1     Running   0          3d19h
collector-88vc5   1/1     Running   0          3d19h
collector-bpj84   1/1     Running   0          3d19h
collector-fnbgr   1/1     Running   0          3d19h
collector-s858j   1/1     Running   0          3d19h
collector-scv4z   1/1     Running   0          3d19h
```

## Install Cluster Observability Operator

```sh
oc apply -f operators/coo/operator-coo.yaml

oc apply -f operators/coo/uiplugin-logging.yaml
```

## View Logs as Admin

Administrator - Observe - Logs

- application
- infrastructure
- audit

## View Logs as User

- Administrator - Workloads/Pods -> select Pod - Aggregated Logs

Keep in mind that the Loki stack has a retention period!

- Developer - Observe - Logs

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