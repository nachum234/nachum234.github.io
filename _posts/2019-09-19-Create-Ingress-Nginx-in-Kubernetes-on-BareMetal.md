---
layout: post
title: "Kubernetes Dashboard Behind Ingress-Nginx"
date: 2019-12-23
---

## Tested On
OS: Ubuntu 18.04  
Kubernetes Version: v1.15.3  
Docker Version: 18.09.8  

## Procedure
I used the “Via the host network” Solution describe in kubernetes docs: <https://kubernetes.github.io/ingress-nginx/deploy/baremetal/#via-the-host-network>

* Download the mandatory file from ingress-nginx

```
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml -o mandatory.yaml
```

* Edit the mandatory file and change the Deployment to Daemonset, remove the replicas from Daemonset spec and add hostNetwork: true to the Daemonset spec

```
vi mandatory.yaml
```

```
...
apiVersion: apps/v1
 kind: DaemonSet
 metadata:
   name: nginx-ingress-controller
   namespace: ingress-nginx
   labels:
     app.kubernetes.io/name: ingress-nginx
     app.kubernetes.io/part-of: ingress-nginx
 spec:
   selector:
     matchLabels:
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
   template:
     metadata:
       labels:
         app.kubernetes.io/name: ingress-nginx
         app.kubernetes.io/part-of: ingress-nginx
       annotations:
         prometheus.io/port: "10254"
         prometheus.io/scrape: "true"
     spec:
       hostNetwork: true
       serviceAccountName: nginx-ingress-serviceaccount
       containers:
         - name: nginx-ingress-controller
           image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.25.1
           args:
             - /nginx-ingress-controller
             - --configmap=$(POD_NAMESPACE)/nginx-configuration
             - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
             - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
             - --publish-service=$(POD_NAMESPACE)/ingress-nginx
             - --annotations-prefix=nginx.ingress.kubernetes.io
           securityContext:
             allowPrivilegeEscalation: true
             capabilities:
               drop:
                 - ALL
               add:
                 - NET_BIND_SERVICE
             # www-data -> 33
             runAsUser: 33
           env:
             - name: POD_NAME
               valueFrom:
                 fieldRef:
                   fieldPath: metadata.name
             - name: POD_NAMESPACE
               valueFrom:
                 fieldRef:
                   fieldPath: metadata.namespace
           ports:
             - name: http
               containerPort: 80
             - name: https
               containerPort: 443
           livenessProbe:
             failureThreshold: 3
             httpGet:
               path: /healthz
               port: 10254
               scheme: HTTP
             initialDelaySeconds: 10
             periodSeconds: 10
             successThreshold: 1
             timeoutSeconds: 10
           readinessProbe:
             failureThreshold: 3
             httpGet:
               path: /healthz
               port: 10254
               scheme: HTTP
             periodSeconds: 10
             successThreshold: 1
             timeoutSeconds: 10
```

* Run kubectl apply

```
kubectl apply -f mandatory.yaml
```