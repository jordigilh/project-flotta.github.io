---
title: Data Synchronization
layout: documentation
---


## Design

Flotta agent and Operator provide functionality of bidirectional synchronization of contents between 
on-device directories and control-plane object storage. User can choose between
in-cluster OCS storage or external storage. OCS takes precedence over external
storage. The architecture of that solution is depicted by the diagrams below.

{% plantuml %}
@startuml

frame Kubernetes {
    component "Flotta Operator" as operator
    component "Flotta Edge API" as edgeAPI
    database "Object Bucket Claim" as buckets
    interface S3
}

frame "Edge Device" {
    node "Flotta Agent" as deviceAgent
}

buckets <--> S3: API
deviceAgent <--> S3: Upload/Download files
deviceAgent -up---> edgeAPI : Get configuration

operator --> buckets: Provision

@enduml

{% endplantuml %}

## OCS storage

### Object Bucket Claim

The objects uploaded from the edge device are stored in a device-dedicated
Object Bucket Claim. Object Bucket Claim is provisioned when a device is
registered with the cluster. The OBC is created in the same namespace and with
the same name as `EdgeDevice` it's created for.

### S3
The Object Bucket Claim is exposed with S3 API and can be accessed with any
client that supports that protocol (i.e. AWS S3 CLI and libraries).

Information needed to access the Bucket using S3 API is stored in following
resources in the same namespace as the OBC (and `EdgeDevice`):

* **ConfigMap** with the same name as the OBC contains the Bucket name and in-cluster service endpoint address;
* **Secret** with the same name as the OBC contains `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

### External S3 storage

You can use any S3 storage server for data upload.
In order to configure storage for the device you need to do the following:
- configure EdgeDevice
- create storage configuration resources

## Configuring EdgeDevice
The storage configuration is taken from a user supplied ConfigMap and Secret.
Both resources must be located in the same namespace as the `EdgeDevice`.
These resources are specified in the 'spec.storage.s3' section. Here's an
example of EdgeDevice with storage configuration:

```yml
metadata:
  namespace: edgedevice-namespace
spec:
  requestTime: "2021-10-19T18:13:04Z",
  storage:
    s3:
      configMapName: "s3configmap-name",
      secretName: "s3secret-name",
```

### Configuration resources

Here are examples of the resources:

#### ConfigMap

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: s3configmap-name
  namespace: edgedevice-namespace
data:
  BUCKET_HOST: play.min.io
  BUCKET_NAME: device-bucket-6
  BUCKET_PORT: "443"
  BUCKET_REGION: us-east-1
```

#### Secret

```yml
apiVersion: v1
kind: Secret
metadata:
  name: s3secret-name
  namespace: edgedevice-namespace
type: Opaque
data:
  AWS_ACCESS_KEY_ID: eWRheWFnaTEzNjk=
  AWS_SECRET_ACCESS_KEY: eWRheWFnaTEzNjk=
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUZDVEND... # optional
```

### S3 server for development and testing

You can use play.min.io as an S3 storage server.
Use `minioadmin` as username and password or check [this
page](https://docs.min.io/minio/baremetal/console/minio-console.html#minio-console)
for the updated username (access key) and password (secret key).
The region of the bucket will probably be `us-east-1`.
Go to `Settings` for viewing and editing the region.


## Bi-directional configuration

The Flotta agent periodically downloads configuration from the Flotta operator
and part of that configuration is data paths mapping for data synchronization 
from the edge device to the S3 storage - specified in each `EdgeWorkload`:

```yaml
spec:
  data:
    egress:
      - source: local/upload/log
        target: remote/log
      - source: local/upload/telemetry
        target: remote/telemetry
    ingress:
      - source: remote/data
        target: local/download/data
      
```

Each `egress` item specifies which on-device directory (`source`) should be
synchronized to which directory (`target`). `source` directory is always a
subdirectory of a "well-known" `/export` directory in every container running on
the device.

At the same time, the `ingress` pair specifies the data paths for the downstream data synchronization
between the remote storage and the device. The `source` field defines the directory in the S3 storage
that will be used as synchronization source point, whereas the `target` directory determines the end point
inside the device where the content will be synchronized to. Note that currently there are no controls in place
to control how much storage is being consumed by the ingress synchronization, which can lead to the device storage
being filled due to the ingress synchronization process.

`Ingress` and `egress` data synchronizations are independent from each other and are not required to be defined together in the workload manifest. You can have workloads that only import (`ingress`) or export (`egress`), or both like in this example. 

The `/export` directory is shared among containers of one workload (pod), but
different workloads (pods) have them separate; each workload has `/export`
directory backed by different host path volume. It is added automatically and
should not be part of the `EdgeWorkload`.

The device configuration provided by the Flotta operator also contains S3
connection details (endpoint URL, bucket name, keys) for connecting to a bucket.

### Example

EdgeWorkload:
```yaml
apiVersion: management.project-flotta.io/v1alpha1
kind: EdgeWorkload
metadata:
  name: os-stats
spec:
  deviceSelector:
    matchLabels:
      dc: home
  data:
    egress:
      - source: stats
        target: statistics
  type: pod
  pod:
    spec:
      containers:
        - name: stats-collector
          image: quay.io/jdzon/os-stats:v1
```

Pod specification used to run the workload (generated):

```yaml
kind: Pod
metadata:
  creationTimestamp: null
  name: os-stats
spec:
  containers:
  - image: quay.io/jdzon/os-stats:v1
    name: stats-collector
    resources: {}
    volumeMounts:
    - mountPath: /export
      name: export-os-stats
  volumes:
  - hostPath:
      path: /var/local/yggdrasil/device/volumes/os-stats
      type: DirectoryOrCreate
    name: export-os-stats```
```

In this case on-device `/export/stats` directory will be synced to a
`statistics` subdirectory in the bucket.

## Upload files

The Flotta agent synchronizes paths specified in the configuration every 15
seconds. Only new or changed files are transferred.

Files removed on the device are not removed from the storage. Unregistering the device will remove the used storage.
