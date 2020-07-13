---
layout: default
title: "Deploy Prometheus and Grafana on Kubernetes"
date: 2019-12-29
---

## Tested On
OS: Ubuntu 18.04  
Kubernetes Version: v1.15.3  
Docker Version: 18.09.8  
Prometheus Version: 2.12.0  

## Prometheus Deployment
* Create monitoring namespance

```
kubectl create namespace monitoring
```

* apply the following files

prometheus-cluster-role.yaml:

```
apiVersion: rbac.authorization.k8s.io/v1beta1
 kind: ClusterRole
 metadata:
   name: prometheus
 rules:
 apiGroups: [""]
 resources:
 nodes
 nodes/proxy
 services
 endpoints
 pods
 verbs: ["get", "list", "watch"]
 apiGroups:
 extensions
 resources:
 ingresses
 verbs: ["get", "list", "watch"]
 nonResourceURLs: ["/metrics"]
   verbs: ["get"]
 apiVersion: rbac.authorization.k8s.io/v1beta1
 kind: ClusterRoleBinding
 metadata:
   name: prometheus
 roleRef:
   apiGroup: rbac.authorization.k8s.io
   kind: ClusterRole
   name: prometheus
 subjects:
 kind: ServiceAccount
 name: default
 namespace: monitoring
```

prometheus-config-map.yaml:

```
apiVersion: v1
 kind: ConfigMap
 metadata:
   name: prometheus-server-conf
   labels:
     name: prometheus-server-conf
   namespace: monitoring
 data:
   prometheus.rules: |-
     groups:
     - name: devopscube demo alert
       rules:
       - alert: High Pod Memory
         expr: sum(container_memory_usage_bytes) > 1
         for: 1m
         labels:
           severity: slack
         annotations:
           summary: High Memory Usage
   prometheus.yml: |-
     global:
       scrape_interval: 5s
       evaluation_interval: 5s
     rule_files:
       - /etc/prometheus/prometheus.rules
     alerting:
       alertmanagers:
       - scheme: http
         static_configs:
         - targets:
           - "alertmanager.monitoring.svc:9093"
 scrape_configs:   - job_name: 'kubernetes-apiservers'     kubernetes_sd_configs:     - role: endpoints     scheme: https     tls_config:       ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt     bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token     relabel_configs:     - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]       action: keep       regex: default;kubernetes;https   - job_name: 'kubernetes-nodes'     scheme: https     tls_config:       ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt     bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token     kubernetes_sd_configs:     - role: node     relabel_configs:     - action: labelmap       regex: __meta_kubernetes_node_label_(.+)     - target_label: __address__       replacement: kubernetes.default.svc:443     - source_labels: [__meta_kubernetes_node_name]       regex: (.+)       target_label: __metrics_path__       replacement: /api/v1/nodes/${1}/proxy/metrics   - job_name: 'kubernetes-pods'     kubernetes_sd_configs:     - role: pod     relabel_configs:     - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]       action: keep       regex: true     - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]       action: replace       target_label: __metrics_path__       regex: (.+)     - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]       action: replace       regex: ([^:]+)(?::\d+)?;(\d+)       replacement: $1:$2       target_label: __address__     - action: labelmap       regex: __meta_kubernetes_pod_label_(.+)     - source_labels: [__meta_kubernetes_namespace]       action: replace       target_label: kubernetes_namespace     - source_labels: [__meta_kubernetes_pod_name]       action: replace       target_label: kubernetes_pod_name   - job_name: 'kubernetes-cadvisor'     scheme: https     tls_config:       ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt     bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token     kubernetes_sd_configs:     - role: node     relabel_configs:     - action: labelmap       regex: __meta_kubernetes_node_label_(.+)     - target_label: __address__       replacement: kubernetes.default.svc:443     - source_labels: [__meta_kubernetes_node_name]       regex: (.+)       target_label: __metrics_path__       replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor   - job_name: 'kubernetes-service-endpoints'     kubernetes_sd_configs:     - role: endpoints     relabel_configs:     - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]       action: keep       regex: true     - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]       action: replace       target_label: __scheme__       regex: (https?)     - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]       action: replace       target_label: __metrics_path__       regex: (.+)     - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]       action: replace       target_label: __address__       regex: ([^:]+)(?::\d+)?;(\d+)       replacement: $1:$2     - action: labelmap       regex: __meta_kubernetes_service_label_(.+)     - source_labels: [__meta_kubernetes_namespace]       action: replace       target_label: kubernetes_namespace     - source_labels: [__meta_kubernetes_service_name]       action: replace       target_label: kubernetes_name
```

prometheus-deployment.yaml:

```
apiVersion: extensions/v1beta1
 kind: Deployment
 metadata:
   name: prometheus-deployment
   labels:
     app: prometheus-server
   namespace: monitoring
 spec:
   replicas: 1
   template:
     metadata:
       labels:
         app: prometheus-server
     spec:
       containers:
         - name: prometheus
           image: prom/prometheus
           args:
             - "--config.file=/etc/prometheus/prometheus.yml"
             - "--storage.tsdb.path=/prometheus/"
           ports:
             - containerPort: 9090
           volumeMounts:
             - name: prometheus-config-volume
               mountPath: /etc/prometheus/
             - name: prometheus-storage-volume
               mountPath: /prometheus/
       volumes:
         - name: prometheus-config-volume
           configMap:
             defaultMode: 420
             name: prometheus-server-conf
     - name: prometheus-storage-volume       emptyDir: {}
```

prometheus-service.yaml:

```
apiVersion: v1
 kind: Service
 metadata:
   name: prometheus-service
   namespace: monitoring
   annotations:
       prometheus.io/scrape: 'true'
       prometheus.io/path:   /
       prometheus.io/port:   '8080'
 spec:
   selector:
     app: prometheus-server
   type: NodePort
   ports:
     - port: 8080
       targetPort: 9090
   selector:
     app: prometheus-server
```

prometheus-ingress-service.yml:

```
apiVersion: networking.k8s.io/v1beta1
 kind: Ingress
 metadata:
   name: prometheus-ingress
   namespace: monitoring
 spec:
   tls:
     - hosts:
       - prom.example.com
       secretName: wildcard.example.com.crt
   rules:
 host: prom.example.com http:   paths: path: /
 backend:
   serviceName: prometheus-service
   servicePort: 8080
```

* apply commands

```
kubectl apply -f prometheus-cluster-role.yaml
kubectl apply -f prometheus-config-map.yaml
kubectl apply -f prometheus-deployment.yaml
kubectl apply -f prometheus-service.yaml
kubectl apply -f prometheus-ingress-service.yml
```

* upload certificate for nginx https
```
kubectl create secret tls -n monitoring wildcard.example.com.crt --key wildcard.example.com.pem --cert wildcard.example.com.crt
```

## Grafana Deployment
* apply the following files:

grafana-configmap.yaml:

```
apiVersion: v1
 kind: ConfigMap
 metadata:
   name: cluster-monitoring-grafana-ini
   namespace: monitoring
   labels:
     app.kubernetes.io/name: cluster-monitoring
     app.kubernetes.io/component: grafana
 data:
   # Grafana's main configuration file. To learn more about the configuration options available to you,
   # consult https://grafana.com/docs/installation/configuration
   grafana.ini: |
     [analytics]
     check_for_updates = true
     [grafana_net]
     url = https://grafana.example.com
     [log]
     mode = console
     [paths]
     data = /var/lib/grafana/data
     logs = /var/log/grafana
     plugins = /var/lib/grafana/plugins
 apiVersion: v1
 kind: ConfigMap
 metadata:
   name: cluster-monitoring-grafana-datasources
   namespace: monitoring
   labels:
     app.kubernetes.io/name: cluster-monitoring
 data:
   # A file that specifies data sources for Grafana to use to populate dashboards.
   # To learn more about configuring this, consult https://grafana.com/docs/administration/provisioning/#datasources
   datasources.yaml: |
     apiVersion: 1
     datasources:
     - access: proxy
       isDefault: true
       name: prometheus
       type: prometheus
       url: http://prometheus-service.monitoring:8080
       version: 1
```

grafana-pv-data.yml:

```
apiVersion: v1
 kind: PersistentVolume
 metadata:
   name: grafana-data
   namespace: monitoring
   labels:
     name: grafana-data
 spec:
   capacity:
     storage: 200Gi
   accessModes:
 ReadWriteOnce persistentVolumeReclaimPolicy: Retain storageClassName: local-storage local: path: /var/lib/k8s/volumes/grafana/data nodeAffinity: required:   nodeSelectorTerms: matchExpressions: key: kubernetes.io/hostname
 operator: In
 values:
 k8s-02
```

grafana-secret.yaml:

```
apiVersion: v1
 kind: Secret
 metadata:
   name: cluster-monitoring-grafana
   namespace: monitoring
   labels:
     app.kubernetes.io/name: cluster-monitoring
     app.kubernetes.io/component: grafana
 type: Opaque
 data:
   # By default, admin-user is set to admin
   admin-user: YWRtaW4=
   admin-password: "base64encodedpassword"
```

grafana-serviceaccount.yaml:

```
apiVersion: v1
 kind: ServiceAccount
 metadata:
   name: grafana
   namespace: monitoring
```

grafana-service.yaml:

```
apiVersion: v1
 kind: Service
 metadata:
   name: grafana-service
   namespace: monitoring
   labels:
     k8s-app: grafana
     app.kubernetes.io/name: cluster-monitoring
     app.kubernetes.io/component: grafana
 spec:
   ports:
     # Routes port 80 to port 3000 of the Grafana StatefulSet Pods
     - name: http
       port: 80
       protocol: TCP
       targetPort: 3000
   selector:
     k8s-app: grafana
```

grafana-statefulset.yaml:

```
apiVersion: apps/v1beta2
 kind: StatefulSet
 metadata:
   name: cluster-monitoring-grafana
   namespace: monitoring
   labels: &Labels
     k8s-app: grafana
     app.kubernetes.io/name: cluster-monitoring
     app.kubernetes.io/component: grafana
 spec:
   serviceName: cluster-monitoring-grafana
   replicas: 1
   selector:
     matchLabels: *Labels
   template:
     metadata:
       labels: *Labels
     spec:
       serviceAccountName: grafana
       # Configure an init container that will chmod 777 Grafana's data directory
       # and volume before the main Grafana container starts up.
       # To learn more about init containers, consult https://kubernetes.io/docs/concepts/workloads/pods/init-containers/
       # from the official Kubernetes docs.
       initContainers:
           - name: "init-chmod-data"
             image: debian:9
             imagePullPolicy: "IfNotPresent"
             command: ["chmod", "777", "/var/lib/grafana"]
             volumeMounts:
             - name: grafana-data
               mountPath: "/var/lib/grafana"
       containers:
         - name: grafana
           # The main Grafana container, which uses the grafana/grafana:6.0.1 image
           # from https://hub.docker.com/r/grafana/grafana
           image: grafana/grafana:6.2.5
           imagePullPolicy: Always
           # Mount in all the previously defined ConfigMaps as volumeMounts
           # as well as the Grafana data volume
           volumeMounts:
             - name: config
               mountPath: "/etc/grafana/"
             - name: datasources
               mountPath: "/etc/grafana/provisioning/datasources/"
             - name: grafana-data
               mountPath: "/var/lib/grafana"
           ports:
             - name: service
               containerPort: 80
               protocol: TCP
             - name: grafana
               containerPort: 3000
               protocol: TCP
           # Set the GF_SECURITY_ADMIN_USER and GF_SECURITY_ADMIN_PASSWORD environment variables
           # using the Secret defined in grafana-secret.yaml
           env:
             - name: GF_SECURITY_ADMIN_USER
               valueFrom:
                 secretKeyRef:
                   name: cluster-monitoring-grafana
                   key: admin-user
             - name: GF_SECURITY_ADMIN_PASSWORD
               valueFrom:
                 secretKeyRef:
                   name: cluster-monitoring-grafana
                   key: admin-password
           # Define a liveness and readiness probe that will hit /api/health using port 3000.
           # To learn more about Liveness and Readiness Probes,
           # consult https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
           # from the official Kubernetes docs.
           livenessProbe:
             httpGet:
               path: /api/health
               port: 3000
             initialDelaySeconds: 60
             timeoutSeconds: 30
             failureThreshold: 10
             periodSeconds: 10
           readinessProbe:
             httpGet:
               path: /api/health
               port: 3000
             initialDelaySeconds: 60
             timeoutSeconds: 30
             failureThreshold: 10
             periodSeconds: 10
           # Define resource limits and requests of 50m of CPU and 100Mi of memory.
           resources:
             limits:
               cpu: 50m
               memory: 100Mi
             requests:
               cpu: 50m
               memory: 100Mi
       # Define configMap volumes for the above ConfigMap files, and volumeClaimTemplates
       # for Grafana's 2Gi Block Storage data volume, which will be mounted to /var/lib/grafana.
       volumes:
         - name: config
           configMap:
             name: cluster-monitoring-grafana-ini
         - name: datasources
           configMap:
             name: cluster-monitoring-grafana-datasources
   volumeClaimTemplates:
 metadata:
   name: grafana-data
 spec:
   accessModes: [ "ReadWriteOnce" ]
   storageClassName: "local-storage"
   resources:
     requests:
       storage: 200Gi
   selector:
     matchExpressions:
       - {key: name, operator: In, values: [grafana-data]}
```

grafana-ingress-service.yml:

```
apiVersion: networking.k8s.io/v1beta1
 kind: Ingress
 metadata:
   name: grafana-ingress
   namespace: monitoring
 spec:
   tls:
     - hosts:
       - grafana.example.com
       secretName: wildcard.example.com.crt
   rules:
 host: grafana.example.com http:   paths: path: /
 backend:
   serviceName: grafana-service
   servicePort: 80
```

* apply commands

```
kubectl apply -f grafana-configmap.yaml
kubectl apply -f grafana-pv-data.yml
kubectl apply -f grafana-secret.yaml
kubectl apply -f grafana-serviceaccount.yaml
kubectl apply -f grafana-service.yaml
kubectl apply -f grafana-statefulset.yaml
kubectl apply -f grafana-ingress-service.yml
```

* login to grafana with admin and your password and import some dashboards to monitor kuberentes. I used the following dashboards

<https://grafana.com/grafana/dashboards/10000>
<https://grafana.com/grafana/dashboards/315>
