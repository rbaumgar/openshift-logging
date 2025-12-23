# How to Install Garage as S3 Storage Provider for OpenShift Logging

*By Robert Baumgartner, Red Hat Austria, December 2025 (OpenShift 4.20, OpenShift Logging 6.4)*

In this blog, I will guide you on

- How to Install Garage as S3 Storage Provider

`Garage` is a lightweight geo-distributed data store that implements the `Amazon S3` object storage protocol. 
It enables applications to store large blobs such as pictures, videos, images, documents, etc., in a redundant multi-node setting. 
S3 is versatile enough to also be used to publish a static website.

OpenShift Logging, Loki, Quay and Tempo require an S3 object store. If you don't already have an object store provider, use Garage.

[Garage: Deploying on Kubernetes Cookbook](https://garagehq.deuxfleurs.fr/documentation/cookbook/kubernetes/)

## Create a New Garage Project

First, create a dedicated namespace for Garage to keep your resources organized.

```shell
$ GARAGE_NAMEPSPACE=minio
$ oc new-project $GARAGE_NAMESPACE
```

## Install Garage

Download the official repository and deploy the Helm chart.

```shell
$ git clone https://git.deuxfleurs.fr/Deuxfleurs/garage -b main-v2
$ cd garage/scripts/helm
$ helm install --create-namespace --namespace $GARAGE_NAMESPACE garage ./garage
```

## Configure Security for OpenShift

To ensure Garage runs smoothly on OpenShift's security architecture, you need to adjust the `securityContext`.

* **Remove the default context:**
```shell
$ kubectl patch statefulsets garage --type='json' -p='[{"op": "remove", "path": "/spec/template/spec/securityContext"}]'
```


* **Optimize volume permissions:** This prevents startup delays when handling large numbers of objects.

Warning: `Setting volume ownership for 172b102d-21c8-46be-baf4-6720c4bc87cd/volumes/kubernetes.io~csi/pvc-057db6a4-ffe9-4352-9157-fb2641a9e1ff/mount is taking longer than expected, consider using OnRootMismatch`

See (https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#configure-volume-permission-and-ownership-change-policy-for-pods)

```shell
$ kubectl patch statefulsets garage --type='json' -p='[{"op": "add", "path": "/spec/template/spec/securityContext", "value": {"fsGroupChangePolicy": "OnRootMismatch"} }]'
statefulset.apps/garage patched
```

## Cluster Initialization

Once the pods are ready, you must link the nodes together to form a cluster.

### Capture Node Information

Run this script to grab the IP addresses and unique IDs of your Garage nodes:

```shell
for i in {0..2}; do 
    export IP_$i=`kubectl get pod garage-$i -o jsonpath='{.status.podIP}'`;
    export ID_$i=`kubectl exec garage-$i -- /garage node id 2>/dev/null`
done
```

### Connect the Nodes

Tell the nodes to communicate with each other:

```shell
for i in {1..2}; do 
    id=ID_${i}
    ip=IP_${i}
    kubectl exec garage-0 -- /garage node connect ${!id}@${!ip}:3901;
done

$ kubectl exec garage-0 -- /garage status
```

## Storage Layout & Capacity

Define how much storage each node should contribute to the cluster (e.g., 5Gi).

```shell
SIZE=5Gi
for i in {0..2}; do
    id=ID_${i}
    # Resize the PVC
    kubectl patch pvc data-garage-$i -p '{"spec":{"resources":{"requests":{"storage":"'${SIZE}'"}}}}'
    # Assign to the 'dc1' zone
    kubectl exec garage-$i -- /garage layout assign -z dc1 -c ${SIZE} ${!id}
done

$ kubectl get pvc -l app.kubernetes.io/name=garage
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      VOLUMEATTRIBUTESCLASS   AGE
data-garage-0   Bound    pvc-7c4980c9-2337-4720-a01a-728dd63fe7f3   5Gi        RWO            nfs-csi-storage   <unset>                 41h
data-garage-1   Bound    pvc-bcd09406-e6a3-4f8e-b660-8ce87e8bcf4b   5Gi        RWO            nfs-csi-storage   <unset>                 41h
data-garage-2   Bound    pvc-86436f02-ef59-43a9-a8ca-aabe0ff20aad   5Gi        RWO            nfs-csi-storage   <unset>                 41h
meta-garage-0   Bound    pvc-1e86c494-111f-404a-983b-b87fb0476507   100Mi      RWO            nfs-csi-storage   <unset>                 41h
meta-garage-1   Bound    pvc-f77f4f85-bf6e-4828-9f27-2a6e17000105   100Mi      RWO            nfs-csi-storage   <unset>                 41h
meta-garage-2   Bound    pvc-20703d54-480f-4e34-8685-4e31069ea4aa   100Mi      RWO            nfs-csi-storage   <unset>                 41h
```

Apply the layout as version 1 and check the status again.

```shell
# Finalize the layout
$ kubectl exec garage-0 -- /garage layout apply --version 1

# Check the status
$ kubectl exec garage-0 -- /garage status
Defaulted container "garage" out of: garage, garage-init (init)
2025-12-09T10:42:31.645340Z  INFO garage_net::netapp: Connected to 127.0.0.1:3901, negotiating handshake...    
2025-12-09T10:42:31.705346Z  INFO garage_net::netapp: Connection established to 8c823863890494a7    
==== HEALTHY NODES ====
ID                Hostname  Address            Tags  Zone  Capacity   DataAvail
8c823863890494a7  garage-2  10.128.4.115:3901  []    dc1   1000.0 MB  37.6 GB (4.7%)
655d52176e20ece6  garage-1  10.130.0.129:3901  []    dc1   1000.0 MB  37.6 GB (4.7%)
517afc37a110c7cd  garage-0  10.131.1.149:3901  []    dc1   1000.0 MB  37.6 GB (4.7%)
```

## Create Storage for Loki

Loki requires a Bucket, an Access Key, and a Secret to store logs.

### Create Key and Bucket

```shell
$ KeyName=logging
$ BucketName=openshift-logging

# Create Key
$ response=$(kubectl exec garage-0 -- /garage key create ${KeyName})

# Extract IDs (for Version 2.1.0)
$ KeyID=$(echo $response | awk '{print $8}')
$ KeySecret=$(echo $response | awk '{print $14}')

# Create Bucket and Assign Permissions
$ kubectl exec garage-0 -- /garage bucket create ${BucketName}
$ kubectl exec garage-0 -- /garage bucket allow ${BucketName} --read --write --key ${KeyName}
```

### Create the OpenShift Secret

Apply this secret so Loki can authenticate with Garage:

```shell
$ Loki_Namespace=openshift-logging
$ cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: lokistack-loki-s3
  namespace: ${Loki_Namespace}$
stringData:
  endpoint: http://garage.garage.svc:3900
  bucketnames: ${BucketName}
  access_key_id: ${KeyID}
  access_key_secret: ${KeySecret}
  region: garage
type: Opaque
EOF
```

Or by using `kubectl` command.

```shell
$ kubectl create secret generic lokistack-loki-s3 -n openshift-logging \
          --from-literal=bucketnames="${BucketName}" \
          --from-literal=endpoint="http://garage.garage.svc:3900" \
          --from-literal=access_key_id="${KeyID}" \
          --from-literal=access_key_secret="${KeySecret}" \
          --from-literal=region=garage
```

## Create Storage for Tempo

Tempo requires a Bucket, an Access Key, and a Secret to store logs.

```shell
$ KeyName=tempo
$ BucketName=tempo-demo
$ Tempo_Namespace=tempo-demo

# Create Key
$ response=$(kubectl exec garage-0 -- /garage key create ${KeyName})

# Extract IDs (for Version 2.1.0)
$ KeyID=$(echo $response | awk '{print $8}')
$ KeySecret=$(echo $response | awk '{print $14}')

# Create Bucket and Assign Permissions
$ kubectl exec garage-0 -- /garage bucket create ${BucketName}
$ kubectl exec garage-0 -- /garage bucket allow ${BucketName} --read --write --key ${KeyName}

$ cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: garage-test
  namespace: ${Tempo_Namespace}
stringData:
  endpoint: http://garage.garage.svc:3900
  bucket: ${BucketName}
  access_key_id: ${KeyID}
  access_key_secret: ${KeySecret}
type: Opaque
EOF
```

## How to Connect the AWS CLI with Garage

Make a port forward to get access to the API, in a separate window.

```shell
$ oc port-forward -n garage garage-0 3900:3900
```

Set the environment and use the AWS CLI.

```shell
export AWS_ACCESS_KEY_ID=${KeyID}
export AWS_SECRET_ACCESS_KEY=${KeySecret}
export AWS_DEFAULT_REGION='garage'
export AWS_ENDPOINT_URL=http://localhost:3900

// some example commands
aws s3 ls
aws s3 ls s3://openshift-logging --recursive
aws s3 rm s3://openshift-logging/infrastructure --recursive

aws s3api list-buckets
```

### If You Want to Set Up the Lifecycle for a Bucket

See KB solution https://access.redhat.com/solutions/7053212

```shell
cat > logging-bucket-lifecycle.json <<EOF
{
  "Rules": [
    {
      "Expiration": {
        "Days": 4  
      },
      "ID": "LifeCycle4Days",  
      "Filter": {},
      "Status": "Enabled"
    }
  ]
}
EOF

// st the lifecycle-configuration
aws s3api put-bucket-lifecycle-configuration --bucket openshift-logging --lifecycle-configuration file://logging-bucket-lifecycle.json

// check the lifecycle-configuration
aws s3api get-bucket-lifecycle-configuration --bucket openshift-logging

// remove the lifecycle json
rm logging-bucket-lifecycle.json
```

## Upgrade Garage Version

If you have not installed `main-v2` you can upgrade with the following steps.

```shell
$ kubectl scale statefulsets garage --replicas=0
statefulset.apps/garage scaled

$ kubectl set image statefulsets/garage garage=dxflrs/amd64_garage:v2.1.0

//change replication_mode = "3" -> replication_factor = 3
$ kubectl edit cm garage-config

$ kubectl scale statefulsets garage --replicas=3
```

## Uninstall Garage

If you no longer requires Garage, delete the Hlm chart and the PVCs.

```shell
// store the node IDs
for i in {0..2}; do 
    export ID_$i=`kubectl exec garage-$i -- /garage node id 2>/dev/null`
done

$ helm delete garage
$ oc delete pvc  data-garage-0 data-garage-1 data-garage-2 meta-garage-0 meta-garage-1 meta-garage-2

// delete garagenodes CRs
$ kubectl delete garagenodes.deuxfleurs.fr ${ID_0} ${ID_1} ${ID_2}

// delete namespace if needed
$ oc delete ns $GARAGE_NAMESPACE 
```

This document: 
**[Github: rbaumgar/openshift-logging](https://github.com/rbaumgar/openshift-logging/blob/main/HowToInstallGarageForOpenShiftLogging.md)**
