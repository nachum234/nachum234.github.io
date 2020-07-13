---
layout: default
title: "Kubernetes Dashboard Behind Ingress-Nginx"
date: 2019-12-23
---

## Tested On
OS: Ubuntu 18.04  
Kubernetes Version: v1.15.3  
Docker Version: 18.09.8  
Kubernetes Dashboard Version: v1.10.1  

## Installation
* Download kubernetes-dashboard yaml file

```
curl https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml -o kubernetes-dashboard.yaml
```

* Edit the file to change dashboard configuration to use http and insecure port. Here is the file that I used:

```
Copyright 2017 The Kubernetes Authors.
 #
 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at
 #
 http://www.apache.org/licenses/LICENSE-2.0
 #
 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
 ------------------- Dashboard Secret -------------------
 apiVersion: v1
 kind: Secret
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard-certs
   namespace: kube-system
 type: Opaque

 ------------------- Dashboard Service Account -------------------
 apiVersion: v1
 kind: ServiceAccount
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard
   namespace: kube-system

 ------------------- Dashboard Role & Role Binding -------------------
 kind: Role
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   name: kubernetes-dashboard-minimal
   namespace: kube-system
 rules:
   # Allow Dashboard to create 'kubernetes-dashboard-key-holder' secret.
 apiGroups: [""]
 resources: ["secrets"]
 verbs: ["create"]
 # Allow Dashboard to create 'kubernetes-dashboard-settings' config map.
 apiGroups: [""]
 resources: ["configmaps"]
 verbs: ["create"]
 # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
 apiGroups: [""]
 resources: ["secrets"]
 resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs"]
 verbs: ["get", "update", "delete"]
 # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
 apiGroups: [""]
 resources: ["configmaps"]
 resourceNames: ["kubernetes-dashboard-settings"]
 verbs: ["get", "update"]
 # Allow Dashboard to get metrics from heapster.
 apiGroups: [""]
 resources: ["services"]
 resourceNames: ["heapster"]
 verbs: ["proxy"]
 apiGroups: [""]
 resources: ["services/proxy"]
 resourceNames: ["heapster", "http:heapster:", "https:heapster:"]
 verbs: ["get"]

 apiVersion: rbac.authorization.k8s.io/v1
 kind: RoleBinding
 metadata:
   name: kubernetes-dashboard-minimal
   namespace: kube-system
 roleRef:
   apiGroup: rbac.authorization.k8s.io
   kind: Role
   name: kubernetes-dashboard-minimal
 subjects:
 kind: ServiceAccount
 name: kubernetes-dashboard
 namespace: kube-system

 ------------------- Dashboard Deployment -------------------
 kind: Deployment
 apiVersion: apps/v1
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard
   namespace: kube-system
 spec:
   replicas: 1
   revisionHistoryLimit: 10
   selector:
     matchLabels:
       k8s-app: kubernetes-dashboard
   template:
     metadata:
       labels:
         k8s-app: kubernetes-dashboard
     spec:
       containers:
       - name: kubernetes-dashboard
         image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
         ports:
         - containerPort: 8444
           protocol: TCP
         args:
           - --enable-insecure-login
           - --port=8443
           - --insecure-port=8444
           - --insecure-bind-address=0.0.0.0
           # Uncomment the following line to manually specify Kubernetes API server Host
           # If not specified, Dashboard will attempt to auto discover the API server and connect
           # to it. Uncomment only if the default does not work.
           # - --apiserver-host=http://my-address:port
         volumeMounts:
         - name: kubernetes-dashboard-certs
           mountPath: /certs
           # Create on-disk volume to store exec logs
         - mountPath: /tmp
           name: tmp-volume
         livenessProbe:
           httpGet:
             scheme: HTTP
             path: /
             port: 8444
           initialDelaySeconds: 30
           timeoutSeconds: 30
       volumes:
       - name: kubernetes-dashboard-certs
         secret:
           secretName: kubernetes-dashboard-certs
       - name: tmp-volume
         emptyDir: {}
       serviceAccountName: kubernetes-dashboard
       # Comment the following tolerations if Dashboard must not be deployed on master
       tolerations:
       - key: node-role.kubernetes.io/master
         effect: NoSchedule

 ------------------- Dashboard Service -------------------
 kind: Service
 apiVersion: v1
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard
   namespace: kube-system
 spec:
   ports:
     - port: 80
       targetPort: 8444
   selector:
     k8s-app: kubernetes-dashboard
```

* apply the file

```
kubectl apply -f kubernetes-dashboard.yaml
```

* Upload your certificate for ingress-nginx

```
kubectl create secret tls -n kube-system wildcard.example.com.crt --key wildcard.example.com.pem --cert wildcard.example.com.crt
```

* Create Ingress file for nginx configuration

```
vi ingress-service.yml
```

```
apiVersion: networking.k8s.io/v1beta1
 kind: Ingress
 metadata:
   name: k8s-dashboard-ingress
   namespace: kube-system
 spec:
   tls:
     - hosts:
       - k8s-dashboard.example.com
       secretName: wildcard.example.com.crt
   rules:
 host: k8s-dashboard.example.com http:   paths: path: /
 backend:
   serviceName: kubernetes-dashboard
   servicePort: 80
```

* Create dashboard admin user

```
vi dashboard-adminuser.yaml
```

```
apiVersion: v1
 kind: ServiceAccount
 metadata:
   name: admin-user
   namespace: kube-system
 apiVersion: rbac.authorization.k8s.io/v1
 kind: ClusterRoleBinding
 metadata:
   name: admin-user
 roleRef:
   apiGroup: rbac.authorization.k8s.io
   kind: ClusterRole
   name: cluster-admin
 subjects:
 kind: ServiceAccount
 name: admin-user
 namespace: kube-system
```

```
kubectl apply -f dashboard-adminuser.yaml
```

* Get the token of dashboard admin user
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

* Browse to your dashboard https://k8s-dashboard.example.com and login with the token of dashboard admin user
