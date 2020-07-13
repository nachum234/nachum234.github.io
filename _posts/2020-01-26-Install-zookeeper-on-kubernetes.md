---
layout: default
title: "Install zookeeper on kubernetes"
date: 2020-01-26
---

## Tested On
OS: Ubuntu 18.04  
Kubernetes Version: v1.17.0  
Zookeeper Version: 3.5.6

In this guide you will learn how to deploy the official image of zookeeper on kubernetes.  
In this guide I will use local volumes because I am using kubernetes on bare-metal servers.

## Install zookeeper
* create local directories for zookeeper data volumes on all servers that will run zookeeper

```
sudo mkdir -p /var/lib/k8s/volumes/zookeeper/data
```

* apply the following namespace

```
apiVersion: v1  
kind: Namespace  
metadata:  
  name: kafka  
```

```
kubectl apply -f namespace.yml
```

* apply the following physical volumes

```
apiVersion: v1  
kind: PersistentVolume  
metadata:  
  name: zookeeper-data-01  
  labels:  
    name: zookeeper-data  
spec:  
  capacity:  
    storage: 50Gi  
  accessModes:  
  - ReadWriteOnce  
  persistentVolumeReclaimPolicy: Retain  
  storageClassName: local-storage  
  local:  
    path: /var/lib/k8s/volumes/zookeeper/data  
  nodeAffinity:  
    required:   
      nodeSelectorTerms:  
      - matchExpressions:  
        - key: kubernetes.io/hostname  
          operator: In  
          values:  
          - k8s-01  
---  
apiVersion: v1  
kind: PersistentVolume  
metadata:  
  name: zookeeper-data-02  
  labels:  
    name: zookeeper-data  
spec:  
  capacity:  
    storage: 50Gi  
  accessModes:  
  - ReadWriteOnce  
  persistentVolumeReclaimPolicy: Retain  
  storageClassName: local-storage  
  local:  
    path: /var/lib/k8s/volumes/zookeeper/data  
  nodeAffinity:  
    required:   
      nodeSelectorTerms:  
      - matchExpressions:  
        - key: kubernetes.io/hostname  
          operator: In  
          values:  
          - k8s-02  
---  
apiVersion: v1  
kind: PersistentVolume  
metadata:  
  name: zookeeper-data-03  
  labels:  
    name: zookeeper-data  
spec:  
  capacity:  
    storage: 50Gi  
  accessModes:  
  - ReadWriteOnce  
  persistentVolumeReclaimPolicy: Retain   
  storageClassName: local-storage  
  local:  
    path: /var/lib/k8s/volumes/zookeeper/data  
  nodeAffinity:  
    required:   
      nodeSelectorTerms:  
      - matchExpressions:  
        - key: kubernetes.io/hostname  
          operator: In  
          values:  
          - k8s-03  
```

```
kubectl apply -f zookeeper-pv.yml
```

* apply the following zookeeper cluster

```
apiVersion: v1
kind: Service
metadata:
  name: zk-hs
  namespace: kafka
  labels:
    app: zk
spec:
  ports:
  - port: 2888
    name: server
  - port: 3888
    name: leader-election
  clusterIP: None
  selector:
    app: zk
---
apiVersion: v1
kind: Service
metadata:
  name: zk-cs
  namespace: kafka
  labels:
    app: zk
spec:
  ports:
  - port: 2181
    name: client
  selector:
    app: zk
---
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
  namespace: kafka
spec:
  selector:
    matchLabels:
      app: zk
  maxUnavailable: 1
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zk
  namespace: kafka
spec:
  selector:
    matchLabels:
      app: zk
  serviceName: zk-hs
  replicas: 3
  updateStrategy:
    type: RollingUpdate
  podManagementPolicy: OrderedReady
  template:
    metadata:
      labels:
        app: zk
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                    - zk
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: kubernetes-zookeeper
        imagePullPolicy: Always
        image: "zookeeper:3.5.6"
        env:
        - name: ZOO_SERVERS
          value: "server.1=zk-0.zk-hs.kafka.svc.cluster.local:2888:3888;2181 server.2=zk-1.zk-hs.kafka.svc.cluster.local:2888:3888;2181 server.3=zk-2.zk-hs.kafka.svc.cluster.local:2888:3888;2181"
        resources:
          requests:
            memory: "1Gi"
            cpu: "0.5"
        volumeMounts:
        - name: zookeeper-data
          mountPath: /data
        ports:
        - containerPort: 2181
          name: client
        - containerPort: 2888
          name: server
        - containerPort: 3888
          name: leader-election
      initContainers:
      - name: init-myservice
        image: busybox:1.28
        command: ['sh', '-c', 'echo $(( $(echo ${POD_NAME} | cut -d "-" -f 2) + 1 )) > /data/myid']
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        volumeMounts:
        - name: zookeeper-data
          mountPath: /data
  volumeClaimTemplates:
metadata:
  name: zookeeper-data
spec:
  accessModes: [ "ReadWriteOnce" ]
  storageClassName: "local-storage"
  resources:
    requests:
      storage: 50Gi
  selector:
    matchExpressions:
      - {key: name, operator: In, values: [zookeeper-data]}
```

```      
kubectl apply -f zookeeper.yaml
```
