# How to Install MinIO for OpenShift Logging

*By Robert Baumgartner, Red Hat Austria, Janurary 2025 (OpenShift 4.17, OpenShift Logging 6.1)*

In this blog, I will guide you on

- How to Install MinIO for OpenShift Logging

OpenShift Logging and Loki requires an object store. If you don't have already an object store use MinIO.

## Create a New MinIO Project

Create a new project (for example minio) :

```shell
$ MINIO_NAMESPACE=minio
$ oc new-project $MINIO_NAMESPACE
```

## Create the MinIO Objects

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
  minioPassword: $MINIO_ADMIN_PWD
  minioUser: minio
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

$ oc get pod
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

## Create a bucket and user for the Lokistack

```shell
BUCKET=<bucket_name> e.g. openshift-logging 
# create loki user secret
BUCKET_SECRET=`openssl rand -base64 12`

# create an alias for the connection
$ oc rsh deployments/minio-server \
         mc alias set myminio http://localhost:9000 minio $MINIO_ADMIN_PWD
Added `myminio` successfully.

# create an access-policy file
$ cat >bucket-access-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
             "Action": ["s3:*"],
             "Effect": "Allow",
             "Resource": ["arn:aws:s3:::${BUCKET}",
                          "arn:aws:s3:::${BUCKET}/*"]
        }
    ]
}
EOF

# copy policy file to pod
$ cat bucket-access-policy.json | \
  oc exec deployments/minio-server -i -- sh -c "cat /dev/stdin > /tmp/bucket-access-policy.json"

# create new policy
$ oc rsh deployments/minio-server \
         mc admin policy create myminio $BUCKET-access-policy /tmp/bucket-access-policy.json
Created policy `openshift-logging-access-policy` successfully.

# cleanup: remove policy file
$ oc rsh deployments/minio-server rm /tmp/bucket-access-policy.json
$ rm bucket-access-policy.json

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
         mc mb myminio/$BUCKET
Bucket created successfully `myminio/openshift-logging`.

# (optional) list all buckets
$ oc rsh deployments/minio-server \
         mc ls myminio
[2025-01-09 11:28:51 UTC]     0B openshift-logging/

# create user
$ oc rsh deployments/minio-server \
         mc admin user add myminio $BUCKET $BUCKET_SECRET
Added user `loki-user` successfully.

# (optional) list all users
$ oc rsh deployments/minio-server \
         mc admin user ls myminio
enabled    loki-user

# attach openshift-logging-acccess-policy to user openshift-logging
$ oc rsh deployments/minio-server \
         mc admin policy attach myminio $BUCKET-access-policy --user $BUCKET
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

You can log in to the MinIO console with minio/$MINIO_ADMIN_PWD

```shell
# get MinIO console URL
$ oc get route minio-console -o jsonpath='{.spec.host}'
minio-console-minio.apps.rbaumgar.demo.net
```

When you use the MinIO server as a general-purpose object storage (s3), you should remember that you assign similar policies to other users (openshift-logging-access-policy).
If you are using the default policies (readonly, readwrite, and writeonly) those users can access all buckets on the server.

The configuration of the MinIO server is not HA. There is also a MinIO operator available.

If you have networkpolicies in use, allow the project openshift-logging access to the project minio on port 9000.

## Create a Secret for Loki 

Loki requires a secret to define how to access the MinIO object store.

```sh
$ kubectl create secret generic lokistack-loki-s3 -n openshift-logging\
          --from-literal=bucketnames="openshift-logging" \
          --from-literal=endpoint="http://minio.$MINIO_NAMESPACE.svc:9000" \
          --from-literal=access_key_id="$BUCKET" \
          --from-literal=access_key_secret="$BUCKET_SECRET"
secret/lokistack-loki-s3 created
```

The endpoint consists= <svc>.<project>.svc:9000.

Loki supports different preconfigured sizes.

## Uninstall MinIO

If you no longer requires Minio

```sh
$ oc delete ns $MINIO_NAMESPACE 
```

This document: 
**[Github: rbaumgar/openshift-logging](https://github.com/rbaumgar/openshift-logging/blob/main/HowToInstallMinioForOpenShiftLogging.md)**
