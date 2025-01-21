# OpenShift Logging QuickStart Guide with Loki

![](images/logging.jpg)

*By Robert Baumgartner, Red Hat Austria, Janurary 2025 (OpenShift 4.17, OpenShift Logging 6.1)*

In this blog, I will guide you on

- How to install Loki as storage for OpenShift Logging using MinIO as object store

- How to install OpenShift Logging using the Loki stack

- How to install OpenShift console UIPlugin and use it

You need to be cluster admin to follow this guide.

This document is based on OpenShift 4.17. See [Configuring and using logging in OpenShift Container Platform](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/logging/index).

## Install MinIO (optional)
Loki requires an object storage. So if you have already an object storage available you can skip this chapter.

### Create a New MinIO Project

Create a new project (for example minio) :

```shell
$ oc new-project minio
```

### Create the MinIO Objects

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
secret/minio-access-secrets created

$ oc apply -f minio/pvc-minio.yaml
persistentvolumeclaim/minio-home-claim created
persistentvolumeclaim/minio-config-claim created

$ oc apply -f minio/svc-minio.yaml
service/minio created

$ oc apply -f minio/deployment-minio.yaml
deployment.apps/minio-server created

$ oc apply -f minio/route-minio.yaml
route.route.openshift.io/minio-console created
route.route.openshift.io/minio-server created

$ oc get pod -n minio
NAME                           READY   STATUS    RESTARTS   AGE
minio-server-cf8c6c9c4-cvwwv   1/1     Running   0          3d2h

$ oc logs deployments/minio-server -n minio
INFO: WARNING: MINIO_ACCESS_KEY and MINIO_SECRET_KEY are deprecated.
         Please use MINIO_ROOT_USER and MINIO_ROOT_PASSWORD
MinIO Object Storage Server
Copyright: 2015-2025 MinIO, Inc.
License: GNU AGPLv3 - https://www.gnu.org/licenses/agpl-3.0.html
Version: RELEASE.2024-12-18T13-15-44Z (go1.23.4 linux/amd64)

API: http://10.128.4.14:9000  http://127.0.0.1:9000 
WebUI: http://10.128.4.14:44645 http://127.0.0.1:44645   

Docs: https://docs.min.io
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
Created policy `openshift-logging-access-policy` successfully.

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
         mc admin user add myminio loki-user $LOKI_USER_SECRET
Added user `loki-user` successfully.

# (optional) list all users
$ oc rsh deployments/minio-server \
         mc admin user ls myminio
enabled    loki-user

# attach openshift-logging-acccess-policy to user openshift-logging
$ oc rsh deployments/minio-server \
         mc admin policy attach myminio openshift-logging-access-policy --user loki-user
Attached Policies: [openshift-logging-access-policy]
To User: openshift-logging

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

When you use the MinIO server as a general purpose object storage (s3) you should keep in mind, that you assign to the other users similar policies (openshift-logging-access-policy).
If you are using the default polcies (readonly, readwrite and writeonly) those users are able to access all buckets on the server.

The configuration of the MinIO server is not HA. Their is also an MinIO operator available.

Keep in mind if you have networkpolicies in use, allow the project openshift-logging access to the project minio on port 9000.

## Install Operators: OpenShift Logging, Loki, Observability Operator

Install the required operators

```sh
$ oc create -f operators/logging/operator-logging.yaml
namespace/openshift-logging created
operatorgroup.operators.coreos.com/openshift-logging-cqwll created
subscription.operators.coreos.com/cluster-logging created
```

```sh
$ oc create -f operators/loki/operator-loki.yaml
namespace/openshift-operators-redhat created
operatorgroup.operators.coreos.com/openshift-operators-redhat-mhrbh created
subscription.operators.coreos.com/loki-operator created
```

```sh
$ oc create -f operators/coo/operator-coo.yaml
subscription.operators.coreos.com/cluster-observability-operator created
```

Check that all operators are running and phase is *Succeeded*. This may take some minutes.

```sh
$ oc get csv -n openshift-logging cluster-logging.v6.1.0
NAME                      DISPLAY                     VERSION   REPLACES   PHASE
cluster-logging.v6.1.0    Red Hat OpenShift Logging   6.1.0                Succeeded

$ oc get csv -n openshift-operators-redhat loki-operator.v6.1.0
NAME                      DISPLAY                 VERSION   REPLACES   PHASE
loki-operator.v6.1.0      Loki Operator           6.1.0                Succeeded

$ oc get csv -n openshift-operators cluster-observability-operator.0.4.1 
NAME                                   DISPLAY                          VERSION   REPLACES                               PHASE
cluster-observability-operator.0.4.1   Cluster Observability Operator   0.4.1     cluster-observability-operator.0.3.2   Succeeded
```

## Configure the LokiStack 

Loki Operator supports AWS S3, Azure, GCS, MinIO, OpenShift Data Foundation and Swift for the LokiStack object storage.

If you are using a different object store you might need to define the secret in a different way.
See https://github.com/grafana/loki/blob/main/operator/docs/lokistack/object_storage.md

```sh
$ kubectl create secret generic lokistack-minio -n openshift-logging\
          --from-literal=bucketnames="openshift-logging" \
          --from-literal=endpoint="http://minio.minio.svc:9000" \
          --from-literal=access_key_id="loki-user" \
          --from-literal=access_key_secret="$LOKI_USER_SECRET"
secret/lokistack-minio created
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
$ oc project openshift-logging

# find default storageclass
DEFAULt_STORAGECLASS=`kubectl get storageclasses.storage.k8s.io -o=jsonpath='{.items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")].metadata.name}'`

$ cat <<EOF | oc apply -f -
apiVersion: loki.grafana.com/v1
kind: LokiStack
metadata:
  name: logging-loki
  namespace: openshift-logging
spec:
  tenants:
    mode: openshift-logging
  managementState: Managed
  limits:
    global:
      queries:
        queryTimeout: 3m
      retention:
        days: 3
  storage:
    schemas:
      - effectiveDate: '2024-10-11'
        version: v13
    secret:
      name: lokistack-minio
      type: s3
  hashRing:
    type: memberlist
  size: 1x.demo
  storageClassName: $DEFAULT_STORAGECLASS
  replicationFactor: 1
EOF
lokistack.loki.grafana.com/logging-loki created
```

Wait that all pods are running and the status of the CRD goes to ready.

```sh
$ oc get pod -l app.kubernetes.io/instance=logging-loki
NAME                                           READY   STATUS    RESTARTS   AGE
logging-loki-compactor-0                       1/1     Running   0          3d20h
logging-loki-distributor-55f9475cc6-jdhjb      1/1     Running   0          3d20h
logging-loki-gateway-778c49b997-dfzjc          2/2     Running   0          3d20h
logging-loki-gateway-778c49b997-k58v8          2/2     Running   0          3d20h
logging-loki-index-gateway-0                   1/1     Running   0          3d20h
logging-loki-ingester-0                        1/1     Running   0          3d20h
logging-loki-querier-55d7fbb758-222dw          1/1     Running   0          3d20h
logging-loki-query-frontend-754c4594c5-rmgww   1/1     Running   0          3d20h

$ oc get lokistacks.loki.grafana.com -o custom-columns=Status:status.conditions[0].message
Status
All components ready
```

If you select another size than 1x.demo multiple pods for the lokistack will be created.

*LokiStack Components*
- Gateway: The gateway receives requests and redirects them to the appropriate container based on the request’s URL.
- Distributor: The distributor service is responsible for handling incoming streams by clients.
Ingester: The ingester service is responsible for writing log data to long-term storage backends (DynamoDB, S3, Cassandra, etc.) on the write path and returning log data for in-memory queries on the read path.
- Query Frontend:  internally performs some query adjustments and holds queries in an internal queue.
- Querier: handles queries using the LogQL query language, fetching logs both from the ingesters and from long-term storage.
- Compactor: a specific service that reduces the index size by deduping the index and merging all the files to a single file per table.
- Index Gateway: downloads and synchronizes the index from the Object Storage in order to serve index queries to the Queriers and Rulers over gRPC.

## Configure the ClusterLogForwarder to the Lokistack

```sh
$ oc create sa collector
serviceaccount/collector created

$ oc adm policy add-cluster-role-to-user cluster-logging-write-application-logs -z collector 
clusterrole.rbac.authorization.k8s.io/cluster-logging-write-application-logs added: "collector"

$ oc adm policy add-cluster-role-to-user cluster-logging-write-audit-logs -z collector
clusterrole.rbac.authorization.k8s.io/cluster-logging-write-audit-logs added: "collector"

$ oc adm policy add-cluster-role-to-user cluster-logging-write-infrastructure-logs -z collector
clusterrole.rbac.authorization.k8s.io/cluster-logging-write-infrastructure-logs added: "collector"

$ oc adm policy add-cluster-role-to-user collect-application-logs -z collector
clusterrole.rbac.authorization.k8s.io/collect-application-logs added: "collector"

$ oc adm policy add-cluster-role-to-user collect-audit-logs -z collector
clusterrole.rbac.authorization.k8s.io/collect-audit-logs added: "collector"

$ oc adm policy add-cluster-role-to-user collect-infrastructure-logs -z collector
clusterrole.rbac.authorization.k8s.io/collect-infrastructure-logs added: "collector"

$ oc apply -f operators/logging/clusterlogforwarder.yaml
clusterlogforwarder.observability.openshift.io/collector created
```

You only need to apply the required roles for the your required log types (application, audit and infrastracture).

```sh
$ oc get clusterlogforwarders.observability.openshift.io collector --template='{{printf "%-55s %7s %-30s\n" "Type" "Status" "Reason/Message"}}{{range .status.conditions}}{{printf "%-55s %7s %s/%s\n" .type .status .reason .message}}{{end}}'
Type                                                     Status Reason/Message                
observability.openshift.io/ValidLokistackOTLPOutputs       True ValidationSuccess/
observability.openshift.io/Authorized                      True ClusterRolesExist/permitted to collect log types: [application audit infrastructure]
observability.openshift.io/Valid                           True ValidationSuccess/
Ready                                                      True ReconciliationComplete/

# InputConditions
$ oc get clusterlogforwarders.observability.openshift.io collector --template='{{printf "%-55s %7s %-30s\n" "Type" "Status" "Reason/Message"}}{{range .status.inputConditions}}{{printf "%-55s %7s %s/%s\n" .type .status .reason .message}}{{end}}'
Type                                                     Status Reason/Message                
observability.openshift.io/ValidInput-application          True ValidationSuccess/input "application" is valid
observability.openshift.io/ValidInput-infrastructure       True ValidationSuccess/input "infrastructure" is valid
observability.openshift.io/ValidInput-audit                True ValidationSuccess/input "audit" is valid

# OutputConditions
$ oc get clusterlogforwarders.observability.openshift.io collector --template='{{printf "%-72s %7s %-30s\n" "Type" "Status" "Reason/Message"}}{{range .status.outputConditions}}{{printf "%-72s %7s %s/%s\n" .type .status .reason .message}}{{end}}'
Type                                                                      Status Reason/Message                
observability.openshift.io/ValidOutput-default-lokistack-application        True ValidationSuccess/output "default-lokistack-application" is valid
observability.openshift.io/ValidOutput-default-lokistack-audit              True ValidationSuccess/output "default-lokistack-audit" is valid
observability.openshift.io/ValidOutput-default-lokistack-infrastructure     True ValidationSuccess/output "default-lokistack-infrastructure" is valid

# PipelineCondition
$ oc get clusterlogforwarders.observability.openshift.io collector --template='{{printf "%-57s %7s %-30s\n" "Type" "Status" "Reason/Message"}}{{range .status.pipelineConditions}}{{printf "%-57s %7s %s/%s\n" .type .status .reason .message}}{{end}}'

$ oc get daemonsets.apps collector
NAME        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
collector   6         6         6       6            6           kubernetes.io/os=linux   5d23h

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

The Cluster Observability Operator is required for the UI in the OpenShift console to display the log content.

```sh
$ oc apply -f operators/coo/uiplugin-logging.yaml
uiplugin.observability.openshift.io/logging created
```

Go to the OpenShift console and wait until the "refresh web console" pops up. Then the UIPlugin is available.

![](/images/refresh_webconsole.png)


## View Logs as Cluster Admin 

Administrator - Observe - Logs

- Application logs:
Container logs generated by user applications running in the cluster, except infrastructure container applications.

- Infrastructure logs:
Container logs generated by infrastructure namespaces: openshift*, kube*, or default, as well as journald messages from nodes.

- Audit logs:
Logs generated by auditd, the node audit system, which are stored in the /var/log/audit/audit.log file, and logs from the auditd, kube-apiserver, openshift-apiserver services, as well as the ovn project if enabled.

{ log_type="infrastructure"} |= `Started crio` | json | log_source="node" | hostname="master-0": all entries from master-0

## View Logs as User

- Administrator - Workloads/Pods -> select Pod - Aggregated Logs

Keep in mind that the Loki stack has a retention period!

- Developer - Observe - Logs

All messages in this namespace.

## (optional) Create Event Router

If you want to persist the events of the OpenShift platform you can use the Event Router.

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

More details can be found [Collecting and storing Kubernetes events](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html-single/logging/index#cluster-logging-eventrouter). Currently it is missing in 4.17 documentation, but it is working as designed.

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

### View the collected events in the console

select infra pod eventrouter-xxxx

### Uninstall the Event Router

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

Go to Grafan route and login
!!!

![Grafana Dashboard](images/grafana_dashboard.png)

## Access the Loki data with the Grafana dashboard

The preferred option for accessing the data stored in Loki managed by loki-operator when running on OpenShift with the default OpenShift tenancy model is to go through the LokiStack gateway and do proper authentication against the authentication service included in OpenShift.

The configuration uses oauth-proxy to authenticate the user to the Grafana instance and forwards the token through Grafana to LokiStack’s gateway service. This enables the configuration to fully take advantage of the tenancy model, so that users can only see the logs of their applications and only admins can view infrastructure and audit logs.

As the open-source version of Grafana does not support to limit datasources to certain groups of users, all datasources (“application”, “infrastructure”, “audit” and “network”) will be visible to all users. The infrastructure and audit datasources will not yield any data for non-admin users.

Keep in mind that the configuration points to the Loki Gateway server (GATEWAY_ADDRESS). This contains the Lokistack name (logging-loki). If you have changed, replace it with the correct name.

```sh
$ oc apply grafana/grafana_gateway_ocp_oauth.yaml
```

## Uninstall Logging Stack

```sh
$ oc projct openshift-logging

# Observability Operator
$ oc delete uiplugins.observability.openshift.io logging
uiplugin.observability.openshift.io "logging" deleted

# if no other UIPlugin is installed delete operator
$ oc delete subscriptions.operators.coreos.com -n openshift-operators cluster-observability-operator 
subscription.operators.coreos.com "cluster-observability-operator" deleted

$ oc delete csv -n openshift-operators cluster-observability-operator.0.4.1
clusterserviceversion.operators.coreos.com "cluster-observability-operator.0.4.1" deleted

$ oc delete crd uiplugins.observability.openshift.io
customresourcedefinition.apiextensions.k8s.io "uiplugins.observability.openshift.io" deleted

# OpenShift Logging Operator

$ oc delete clusterlogforwarders.observability.openshift.io collector
clusterlogforwarder.observability.openshift.io "collector" deleted

$ oc delete subscriptions.operators.coreos.com cluster-logging
subscription.operators.coreos.com "cluster-logging" deleted

$ oc delete csv cluster-logging.v6.1.0
clusterserviceversion.operators.coreos.com "cluster-logging.v6.1.0" deleted

$ oc delete crd logfilemetricexporters.logging.openshift.io \
                clusterlogforwarders.observability.openshift.io
customresourcedefinition.apiextensions.k8s.io "logfilemetricexporters.logging.openshift.io" deleted
customresourcedefinition.apiextensions.k8s.io "clusterlogforwarders.observability.openshift.io" deleted

# Loki Operator
$ oc delete lokistacks.loki.grafana.com logging-loki
lokistack.loki.grafana.com "logging-loki" deleted

$ oc delete subscriptions.operators.coreos.com -n openshift-operators-redhat loki-operator
subscription.operators.coreos.com "loki-operator" deleted

$ oc delete csv -n openshift-operators-redhat loki-operator.v6.1.0 
clusterserviceversion.operators.coreos.com "loki-operator.v6.1.0" deleted

$ oc delete pvc -l app.kubernetes.io/instance=logging-loki
persistentvolumeclaim "storage-logging-loki-compactor-0" deleted
persistentvolumeclaim "storage-logging-loki-index-gateway-0" deleted
persistentvolumeclaim "storage-logging-loki-ingester-0" deleted
persistentvolumeclaim "wal-logging-loki-ingester-0" deleted

$ oc delete crd alertingrules.loki.grafana.com \
                lokistacks.loki.grafana.com \
                recordingrules.loki.grafana.com \
                rulerconfigs.loki.grafana.com
customresourcedefinition.apiextensions.k8s.io "alertingrules.loki.grafana.com" deleted
customresourcedefinition.apiextensions.k8s.io "lokistacks.loki.grafana.com" deleted
customresourcedefinition.apiextensions.k8s.io "recordingrules.loki.grafana.com" deleted
customresourcedefinition.apiextensions.k8s.io "rulerconfigs.loki.grafana.com" deleted
```

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