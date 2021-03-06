---
title: "kubernetes-local-volume"
date: 2018-03-07T21:48:15+08:00
tags:
- kubernetes
archives:
- kubernetes
categories:
- kubernetes
coverImage: "https://res.cloudinary.com/ddvxfzzbe/image/upload/v1513355392/ChMkJ1f8ljWIBAmcAA-gWT6p-0oAAWzegGSHVwAD6Bx012_telyks.jpg"
thumbnailImage: https://res.cloudinary.com/ddvxfzzbe/image/upload/v1542165911/favicon_z3wusk.png
---
Use kubernetes local volume for pods.

<!--more-->

# Kubernetes local volume

kubernetes提供很多种类型的volume.
如果上云的话你可能会考虑gcePersistentDisk,awsElasticBlockStore,azureDisk
私有云的话你会考虑cephfs,glusterfs,nfs.
但是如果这些条件都不具备,你可以考虑看看 local volume.

包括之前用了很久的hostpath+nodeselector,但是二者组合的方式对于部署statefulset的应用来说，很不方便，kubecon上有关于local-volume这块的介绍吸引了我们.
首先，我们来看一下文档
  1. local volume的介绍(https://kubernetes.io/docs/concepts/storage/volumes/#local)
  2. github：(https://github.com/kubernetes-incubator/external-storage/tree/master/local-volume）


仔细阅读了这两篇文章之后，我们来做个实验.(kubernets 1.9.3)

1. step1

> api-server, controller-manager, scheduler, and all kubelets 开启 feature-gates的功能：
```
--feature-gates=PersistentLocalVolumes=true,VolumeScheduling=true,MountPropagation=true
```

2. step2
 
Creating a StorageClass:
```
$cat local-storage.yaml
  # Only create this for K8s 1.9+
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: local-storage
  provisioner: kubernetes.io/no-provisioner
  volumeBindingMode: WaitForFirstConsumer
```

3. step3

Manually create local persistent volume

> 文档中提供了一个 local volume static provisioner,大概的功能就是自动将你定义的path里的子文件夹创建成persistent volume.这里没有使用这种方式,而是选择了手动创建.
```
$cat local-persistent-volume.yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: example-local-pv
      annotations:
        "volume.alpha.kubernetes.io/node-affinity": '{
          "requiredDuringSchedulingIgnoredDuringExecution": {
            "nodeSelectorTerms": [
              { "matchExpressions": [
                { "key": "kubernetes.io/hostname",
                  "operator": "In",
                  "values": ["192.168.199.140"]
                }
              ]}
              ]}
            }'
    spec:
      capacity:
        storage: 5Gi
      accessModes:
      - ReadWriteOnce
      persistentVolumeReclaimPolicy: Retain
      storageClassName: local-storage
      local:
        path: /local_volume
```

4. step4

创建一个 statefulset 服务（当你需要replicas为多个时，需要提前创建多个pv在local-storageclass里）：

```
$cat local-statefulset.yaml
      apiVersion: apps/v1beta1
      kind: StatefulSet
      metadata:
        name: local-test
      spec:
        serviceName: "local-service"
        replicas: 1
        template:
          metadata:
            labels:
              app: local-test
          spec:
            containers:
            - name: test-container
              image: gcr.io/google_containers/busybox:1.24
              command:
              - "/bin/sh"
              args:
              - "-c"
              - "count=0; count_file=\"/usr/test-pod/count\"; test_file=\"/usr/test-pod/test_file\"; if [ -e $count_file ]; then count=$(cat $count_file); fi; echo $((count+1)) > $count_file; while [ 1 ]; do date >> $test_file; echo \"This is $MY_POD_NAME, count=$(cat $count_file)\" >> $test_file; sleep 10; done"
              volumeMounts:
              - name: local-vol
                mountPath: /usr/test-pod
              env:
              - name: MY_POD_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.name
            securityContext:
              fsGroup: 1234
        volumeClaimTemplates:
        - metadata:
            name: local-vol
          spec:
            accessModes: [ "ReadWriteOnce" ]
            storageClassName: "local-storage"
            resources:
      requests:
        storage: 1Gi
```

ex. 你可以选择创建不同local-storage来为不同的服务提供local volume.达到方便人工管理的效果.









