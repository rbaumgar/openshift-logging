# How to Install Garage as S3 Storage Provider for OpenShift Logging

*By Robert Baumgartner, Red Hat Austria, Janurary 2026 (OpenShift 4.20, OpenShift Logging 6.1)*

In this blog, I will guide you on

- How to Install Garage as S3 Storage Provider for OpenShift Logging

OpenShift Logging and Loki requires an S3 object store. If you don't have already an object store provider use Garage.

Garage is a lightweight geo-distributed data store that implements the `Amazon S3` object storage protocol. 
It enables applications to store large blobs such as pictures, video, images, documents, etc., in a redundant multi-node setting. 
S3 is versatile enough to also be used to publish a static website.

[Garage: Deploying on Kubernetes Cookbook](https://garagehq.deuxfleurs.fr/documentation/cookbook/kubernetes/)

## Create a New Garage Project

Create a new project (for example garage) :

```shell
$ GARAGE_NAMEPSPACE=minio
$ oc new-project $GARAGE_NAMESPACE
```

## Install Garage

Download the Git repo and install with Helm.

```shell
$ git clone https://git.deuxfleurs.fr/Deuxfleurs/garage -b main-v2
$ cd garage/scripts/helm
$ helm install --create-namespace --namespace $GARAGE_NAMESPACE garage ./garage
```

## (for OpenShift) Remove securityContext

Remove the provided `securityContext` provided by the Helm chart.

```shell
$ kubectl patch statefulsets garage --type='json' -p='[{"op": "remove", "path": "/spec/template/spec/securityContext"}]'
```

## Store IDs and IPs

When the pods are ready store the IP adresses and the IDs.

```shell
for i in {0..2}; do 
    export IP_$i=`kubectl get pod garage-$i -o jsonpath='{.status.podIP}'`;
    export ID_$i=`kubectl exec garage-$i -- /garage node id 2>/dev/null`
done

$ env | grep ID_
ID_2=9af310d6cae6d78807d7884471604a9b828f14b2b7195b5c159839fae9022ba3
ID_1=cbc7fbfdd7a234da887fd99f3dfa9788ba816fa9b69216effe68811237b411c0
ID_0=a3802ce46bd50e6aaa7f3c573c16ab914f92451097690b32db93ce88c8e1fdef
$ env | grep IP_
IP_2=10.128.4.123
IP_1=10.130.0.151
IP_0=10.131.1.96
```

## Connect the Nodes 1-2 to 0

Connect the three nodes together and check the status.

```shell
for i in {1..2}; do 
    id=ID_${i}
    ip=IP_${i}
    kubectl exec garage-0 -- /garage node connect ${!id}@${!ip}:3901;
done

$ kubectl exec garage-0 -- /garage status
```

## Resize Storage PVCs and Assign It

Resize the PVCs to the required size and assign it to a zone called `dc1`.

```shell
SIZE=5Gi
for i in {0..2}; do
    id=ID_${i}
    kubectl patch pvc data-garage-$i -p '{"spec":{"resources":{"requests":{"storage":"'${SIZE}'"}}}}'
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

Apply the layout as verion 1 and check again the status.

```shell
$ kubectl exec garage-0 -- /garage layout apply --version 1

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

## Create a Key for the Lokistack

Create a `Key` and extract the `ID` and the `Secret` depending on the Garage version.

```shell
$ KeyName=logging
$ response=`kubectl exec garage-0 -- /garage key create ${KeyName}`
Defaulted container "garage" out of: garage, garage-init (init)
2025-12-09T11:35:42.975201Z  INFO garage_net::netapp: Connected to 127.0.0.1:3901, negotiating handshake...    
2025-12-09T11:35:43.017081Z  INFO garage_net::netapp: Connection established to 517afc37a110c7cd    

// for Version 2.1.0
$ KeyID=`echo $response | awk '{print $8}'`
$ KeySecret=`echo $response | awk '{print $14}'`

// for Version 1.3.0
$ KeyID=`echo $response | awk '{print $6}'`
$ KeySecret=`echo $response | awk '{print $9}'`

$ kubectl exec garage-0 -- /garage key list
Defaulted container "garage" out of: garage, garage-init (init)
2025-12-09T11:36:18.580684Z  INFO garage_net::netapp: Connected to 127.0.0.1:3901, negotiating handshake...    
2025-12-09T11:36:18.625244Z  INFO garage_net::netapp: Connection established to 517afc37a110c7cd    
List of keys:
  GKae6c7b941e98bbf4b6733d5b  logging

## Create a bucket a for the Lokistack

Set BucketName, create it and allow access (read and write).

$ BucketName=openshift-logging
$ kubectl exec garage-0 -- /garage bucket create ${BucketName}
Defaulted container "garage" out of: garage, garage-init (init)
2025-12-09T11:32:47.669354Z  INFO garage_net::netapp: Connected to 127.0.0.1:3901, negotiating handshake...    
2025-12-09T11:32:47.714092Z  INFO garage_net::netapp: Connection established to 517afc37a110c7cd    
Bucket openshift-logging was created.

$ kubectl exec garage-0 -- /garage bucket allow ${BucketName} --read --write --key ${KeyName}
Defaulted container "garage" out of: garage, garage-init (init)
2025-12-09T11:56:55.101382Z  INFO garage_net::netapp: Connected to 127.0.0.1:3901, negotiating handshake...    
2025-12-09T11:56:55.144112Z  INFO garage_net::netapp: Connection established to 517afc37a110c7cd    
New permissions for GKae6c7b941e98bbf4b6733d5b on openshift-logging: read true, write true, owner false.

$ kubectl exec garage-0 -- /garage bucket list
Defaulted container "garage" out of: garage, garage-init (init)
2025-12-09T11:33:21.445333Z  INFO garage_net::netapp: Connected to 127.0.0.1:3901, negotiating handshake...    
2025-12-09T11:33:21.487056Z  INFO garage_net::netapp: Connection established to 517afc37a110c7cd    
List of buckets:
  openshift-logging    da34200476c4b0f117aae612bfaedd35f458d4b043299835fc95580d77151cd9

## Create a Secret for Loki 

Loki requires a secret to define how to access the S3 object store.

```shell
$ cat <<EOF |oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: lokistack-loki-s3
  namespace: openshift-logging
stringData:
  endpoint: http://garage.garage.svc:3900
  bucketnames: ${BucketName}
  access_key_id: ${KeyID}
  access_key_secret: ${KeySecret}
  region: garage
type: Opaque
EOF
secret/garage-test created
```

or by kubectl command.

```shell
$ kubectl create secret generic lokistack-loki-s3 -n openshift-logging \
          --from-literal=bucketnames="${BucketName}" \
          --from-literal=endpoint="http://garage.garage.svc:3900" \
          --from-literal=access_key_id="${KeyID}" \
          --from-literal=access_key_secret="${KeySecret}" \
          --from-literal=region=garage

secret/lokistack-loki-s3 created
```

## How to Connect the AWS CLI with Garage

Make a port forward to get access to the API, in a seperate window.

```shell
$ oc port-forward -n garage garage-0 3900:3900
```

Set the envirnment and use the AWS CLI.

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

### If You Want to Setup the Lifecycle for a Bucket

```shell
cat > logging-bucket-lifecycle.json <<EOF
{
  "Rules": [
    {
      "Expiration": {
        "Days": 1  
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

```sh
$ helm delete garage
$ oc delete pvc  data-garage-0 data-garage-1 data-garage-2 meta-garage-0 meta-garage-1 meta-garage-2
$ oc delete ns $GARAGE_NAMESPACE 
```

This document: 
**[Github: rbaumgar/openshift-logging](https://github.com/rbaumgar/openshift-logging/blob/main/HowToInstallGarageForOpenShiftLogging.md)**
