---
layout: documentation
---

# Operator Config

## Deploying workloads

Workload can be deployed to devices **after** they are registered with the
cluster. `EdgeWorkload` can be deployed only on edge devices in the same
namespace.

### Deploying by device ID

---
**Note**

When both methods are used in one `EdgeWorkload`, this one takes precedence over the selector-based one.

---

`EdgeWorkload` can be deployed on one chosen device by specifying its name in `spec.device` property.

For example, for `EdgeDevice` `242e48d0-286b-4170-9b97-95502066e6ae`, following property should be set in `EdgeWorkload` yaml:

```yaml
spec:
  ...
  device: 242e48d0-286b-4170-9b97-95502066e6ae
  ...
```

### Deploying with selector

`EdgeWorkload` can be installed on multiple devices using label selectors.

To deploy your workload using this method:

1) Label chosen `EdgeDevice` objects;

 For example:

 `oc label edgedevice 242e48d0-286b-4170-9b97-95502066e6ae dc=emea`

2) Select `dc=emea` label in the `EdgeWorkload` specification:

```yaml
spec:
 deviceSelector:
   matchLabels:
     dc: emea
```
   or
```yaml
spec:
 deviceSelector:
   matchExpressions:
     - key: dc
       operator: In
       values: [emea]
```

The second approach can be used for matching multiple values of one label. For example:
```yaml
spec:
 deviceSelector:
   matchExpressions:
     - key: dc
       operator: In
       values: [emea, apac]
```

3) Create the `EdgeWorkload` in the cluster:

```shell
kubectl apply -f your-workload.yaml
```

### Inspecting workload status

To check statuses of all workloads deployed to an edge device:

```shell
kubectl get edgedevice <edge device name> -ojsonpath="{range .status.workloads[*]}{.name}{':\t'}{.phase}{'\n'}{end}"
```

To list all devices having chosen workload deployed:

```shell
oc get edgedevice -l workload/<workload-name>="true"
```

`EdgeDevice` is labeled with `workload/<workload-name>="true"` when `EdgeWorkload` is added to it.

## Container Image Registries Credentials

Container images used by `EdgeWorkloads` may be hosted in private, protected
repositories requiring clients to present correct credentials. Users can provide
said credentials to Flotta operator as a `Secret` referred to by an
`EdgeWorkload`.

Secret should contain correct docker auth file under the `.dockerconfigjson` key:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pull-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: ewoJImF1dGhzIjogewoJCSJxdWF5LmlvIjogewoJCQkiYXV0aCI6ICJabTl2SUdKaGNpQm1iMjhnWW1GeUNnPT0iCgkJfQoJfQp9Cg==
```

where `.dockerconfigjson` is a base64 encoded Docker auth file JSON:

```json
{
  "auths": {
    "quay.io": {
      "auth": "Zm9vIGJhciBmb28gYmFyCg=="
    }
  }
}
```

The above JSON can be taken from file created by running `podman login`, `docker
login` or similar.

Secret created above must be placed in the same namespace as `EdgeWorkload`
using it.

Reference to the above `Secret` in an `EdgeWorkload` spec:
```yaml
spec:
  imageRegistries:
    secretRef:
      name: pull-secret
```


## Podman auto-update

Auto update containers according to their auto-update policy.

Auto-update looks up containers with a specified `io.containers.autoupdate`
label. This label is set by the `flotta-operator`. Flotta-operator looks up
labels on `EdgeWorkloads` CR prefixed by `podman/` string. If such label is
found it's propageted to the podman pod and containers.

If  the  label  is  present  and set to registry, Podman reaches out to the
corresponding registry to check if the image has been updated. The label image
is an alternative to registry maintained for backwards compatibility.  An image
is considered updated if the digest in the local storage is different than the
one of the remote image.  If an image must be  updated,  Podman  pulls  it  down
and restarts the systemd unit executing the container.

If `ImageRegistries.AuthFileSecret` is defined, Podman reaches out to the
corresponding authfile when pulling images.

Edgedevice start the `podman-auto-update.timer` which is responsible for
executing the `podman-auto-update.service`. This unit is triggered daily.

To create a Edgeworkload with auto-update feature enabled define following
label:
```yaml
apiVersion: management.project-flotta.io/v1alpha1
kind: EdgeWorkload
metadata:
  name: nginx
  labels:
    podman/io.containers.autoupdate: registry
spec:
  deviceSelector:
    matchLabels:
      device: local
  type: pod
  pod:
    spec:
      containers:
        - name: nginx
          image: quay.io/bitnami/nginx:1.20
```

## Logging

To change the verbosity of the logger, the user can update the value of the `LOG_LEVEL` field.
Admitted values are: 	`debug`, `info`, `warn`, `error`, `dpanic`, `panic`, and `fatal`.
Refer to [zapcore docs](https://github.com/uber-go/zap/blob/v1.15.0/zapcore/level.go#L32) for details on each log level.

For example:

```bash
kubectl patch cm -n flotta flotta-manager-config \
  --type merge \
  --patch '{"data":{"LOG_LEVEL": "debug"}}'
```

In case of:
-  _Inside the cluster_ run, the pod will be automatically restarted.
-  _Outside the cluster_ run, the user must set the `LOG_LEVEL` field and manually restart the operator.


## Enabling access to host devices in workloads

It is possible to access a mounted host device from a workload running in the device simply by adding a specific `crun` annotation `run.oci.keep_original_groups=1` and granting access rights to the `flotta` user in the host device. By default, Flotta devices come pre-installed with the crun continer runtime and workloads run in a rootless mode for enhanced security. This container runtime allows sharing the running user's group IDs with the container's user, so that it enables containers to access host resources with the same IDs as the running host user. But before a container can access these resources, the user's host needs to have access in the first place. This is by no means an escalation of privileges, since the container is granted only as much access as the host user has. This is an example of the annotation used in an EdgeWorkload manifest. Flotta propagates annotations and labels in EdgeWorkloads to the pod manifests running in the devices when they are prefixed with `podman/`. When processing the labels and annotations, Flotta removes the prefix and passes only the remaining contents of the key and the given value to the pod manifest:

```yaml
apiVersion: management.project-flotta.io/v1alpha1
kind: EdgeWorkload
metadata:
  name: webcam
  annotations:
    podman/run.oci.keep_original_groups: "1"
...
```

Granting access from the host device is simply done by adding the suplementary group to the `flotta` user. For instance, given a device that has a webcam mounted in `/dev/video2` and has the group `video` with read and write access like the following:

```bash
[flotta@device ~]$ ls -la /dev/video2
crw-rw----+ 1 root video 81, 2 Jul  6 09:16 /dev/video2
```

A container can have access to it if the `flotta` user has the supplementary user group `video`:

```bash
[flotta@device ~]$ id
uid=1001(flotta) gid=1001(flotta) groups=1001(flotta),39(video) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

Running this workload will allow the container to access read and write to the `/dev/video2` device in the host:

```yaml
apiVersion: management.project-flotta.io/v1alpha1
kind: EdgeWorkload
metadata:
  name: webcam
  annotations:
    podman/run.oci.keep_original_groups: "1"
spec:
  deviceSelector:
    matchLabels:
      app: webcam
  type: pod
  pod:
    spec:
      containers:
      - command:
        - /bin/sh 
        args:
        - script.sh
        image: quay.io/jordigilh/ffmpeg:latest
        name: fedora
        volumeMounts:
        - mountPath: /dev/video2
          name: video
      volumes:
      - name: video 
        hostPath:
        path: /dev/video2
```