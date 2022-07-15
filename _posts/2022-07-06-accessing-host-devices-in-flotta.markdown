---
layout: post
title:  "Accessing host devices in Flotta"
date:   2022-07-06 12:00:00 +0100
categories: flotta
author: Jordi Gil <jgil@redhat.com>
tags:
  - podman
  - rootless
  - devices
  - webcam

summary-1: Accessing host devices in Flotta
---

In today's container security, it is important to reduce the risk of a security breach in a container that can escape from the container runtime and put the device at risk. One way to reduce such risks is to run the container with a non-root user, so that if there is such spill the attacker won't be able to gain access to root level privileges. These type of containers are what is called rootless containers.

Rootless containers also limit the access to the host devices by default, as it depends on the privileges of the runtime user access. In Flotta we create the user `flotta` as part of the flotta rpm installation and run the workloads with that user. By default the flotta user does not have access to any plugged device so if there is a need two things to happen to enable a workload to access a mounted device:
* Modify the `flotta` user to include the supplementary group assigned to the device.
* Instruct `crun` to keep the running user's groups in the container.
* Disable SELinux in podman due to labeling issues between the container and the device.

## Configuing the host ##

Granting access to the `flotta` user to a mounted host device can be achieved by simply adding the `flotta` user to the group in which the device belongs to. For this example, I'll use my laptop's webcam installed in `/dev/video2`:

```bash
[flotta@fedora ~]$ ls -la /dev/video2
crw-rw----+ 1 root video 81, 2 Jul  5 11:30 /dev/video2
```

If we want `flotta` to access this device, we'll need to add it to the group `video`

```bash
[flotta@fedora ~]$ sudo usermod -a -G video flotta
```

Printing the groups it shows that the `flotta` user now has also access to the additional group `video`

```bash
[flotta@fedora ~]$ id
uid=1001(flotta) gid=1001(flotta) groups=1001(flotta),39(video) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

To test that we have access, we capture a frame from the camera using `ffmpeg`:

```bash
[flotta@fedora ~]$ ffmpeg -f video4linux2 -s 640x480 -i /dev/video2 -ss 0:0:2 -frames 1 /tmp/helloworld.jpg
ffmpeg version 5.0.1 Copyright (c) 2000-2022 the FFmpeg developers
  built with gcc 12 (GCC)
...
...
Output #0, image2, to '/tmp/helloworld.jpg':
  Metadata:
    encoder         : Lavf59.16.100
  Stream #0:0: Video: mjpeg, yuvj444p(pc, progressive), 640x360, q=2-31, 200 kb/s, 30 fps, 30 tbn
    Metadata:
      encoder         : Lavc59.18.100 mjpeg
    Side data:
      cpb: bitrate max/min/avg: 0/0/200000 buffer size: 0 vbv_delay: N/A
frame=    1 fps=0.5 q=1.6 Lsize=N/A time=00:00:00.03 bitrate=N/A speed=0.0166x    
video:4kB audio:0kB subtitle:0kB other streams:0kB global headers:0kB muxing overhead: unknown
```

Configure podman to disable SELinux on all rootless containers:

```bash
[flotta@fedora ~]$ mkdir -p ~/.config/containers
[flotta@fedora ~]$ cat <<EOF >>.config/containers/containers.conf 
[containers]
label = false
EOF
```

## Defining the workload ##
At this step we have ensured that `flotta` as a user in the host has access to the webcam. Now we need to guarantee that the container is also able to share the same privileges. To that we will leverage on the annotation `run.oci.keep_original_groups=1`, supported by the `crun` container runtime, that allows containers to inherit the running user's groups. We inject the annotation in the EdgeWorkload with prefix `podman/` so that the Flotta operator knows that it needs to be propagated it is included as part of the pod's manifest on the agent side. 

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

In this example we are mounting the host device `/dev/video2` to the container's equivalent `/dev/video2`. The image we use here is a vanilla fedora with ffmpeg-free rpm installed and the script run will execute the ffmpeg command we run earlier and store the jpeg in `/tmp/helloworld.jpg`.

Here is the contents of the script:

```bash
#!/bin/sh

for i in {1..10}; do
  echo Taking snapshot
  /usr/bin/ffmpeg -f video4linux2 -s 640x480 -i /dev/video2 -ss 0:0:2 -frames 1 /tmp/helloworld-$i.jpg
  echo Sleeping 10 seconds
  /usr/bin/sleep 10
done
```

## Running the workload ##

At this point, we are ready to deploy the workload on the device. For this example, I have created a VM running fedora and configured it as a I have labeled my edge device with `webcam` to make sure that the Flotta controller schedules the workload on this specific device.

```bash
[jgil@fedora ~]$ kubectl create -f workload_webcam.yaml
```

Since I'm running this on a VM, I'm going to ssh to the device and become the `flotta` user to access the container and see that the images were created:

```bash
[jgil@fedora ~] ssh -l root 192.168.122.23
[root@device ~] su -l flotta -s /bin/bash
[flotta@device ~] podman exec -it webcam-fedora ls -lart /tmp
total 56
dr-xr-xr-x. 1 root root   24 Jul 14 21:44 ..
-rw-r--r--. 1 root root 3740 Jul 14 21:44 helloworld-1.jpg
-rw-r--r--. 1 root root 3456 Jul 14 21:45 helloworld-2.jpg
-rw-r--r--. 1 root root 3875 Jul 14 21:45 helloworld-3.jpg
-rw-r--r--. 1 root root 3832 Jul 14 21:45 helloworld-4.jpg
-rw-r--r--. 1 root root 3891 Jul 14 21:46 helloworld-5.jpg
-rw-r--r--. 1 root root 3805 Jul 14 21:46 helloworld-6.jpg
-rw-r--r--. 1 root root 4298 Jul 14 21:46 helloworld-7.jpg
-rw-r--r--. 1 root root 4824 Jul 14 21:46 helloworld-8.jpg
-rw-r--r--. 1 root root 4843 Jul 14 21:47 helloworld-9.jpg
-rw-r--r--. 1 root root 4718 Jul 14 21:47 helloworld-10.jpg
drwxrwxrwt. 1 root root  322 Jul 14 21:47 .
```

As an exercise, you can enhance this example by adding a data sync path to upload the images to a remote S3 storage using the data transfer capabilities of the Flotta agent. This way you won't need to sneak into the VM like I did to see that the images were created. 

## Conclusion ##

Flotta is gradually improving the support of workloads that can run on devices without compromising the security of the device. With Flotta it is possible to run workloads that generate and consume data and are capable of synchronizing with a remote compatible S3 storage, as well as workloads that require access to the host mounted devices, such as webcams or sensors.

There is still room for improvement in the usability and security aspect of accessing host devices, which we hope to improve in the near future. Feel free to drop suggestions or enhancements to the [project](https://github.com/project-flotta/) in the github repository!.
