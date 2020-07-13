---
layout: post
title: "Install Kubernetes Cluster on BareMetal"
date: 2019-09-19
---

## Tested On
OS: Ubuntu 18.04  
Kubernetes Version: v1.15.3  
Docker Version: 18.09.8  

## Prerequisites
* Install Docker (I used: <https://docs.docker.com/install/linux/docker-ce/ubuntu/> )

```
apt-get update
apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io
```

* Install kubelet kubeadm and kubectl

```
curl -s  https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
apt-get install -y kubelet kubeadm kubectl
```

* Configure docker for kubernetes

```
cat > /etc/docker/daemon.json <<EOF
 {
   "exec-opts": ["native.cgroupdriver=systemd"],
   "log-driver": "json-file",
   "log-opts": {
     "max-size": "100m"
   },
   "storage-driver": "overlay2"
 }
 EOF
systemctl daemon-reload
systemctl restart docker
```

* Disable swap for kubernetes

```
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

# Installation
* Initialize kubernetes cluster (I have two network interfaces one for public and one for private so I use the apiserver-advertise-address with the private address)

```
kubeadm init --apiserver-advertise-address 172.18.73.71 --apiserver-cert-extra-sans k8s-api.example.com
```

* Configure kubectl

```
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get pods --all-namespaces
```

* Install weaveworks network plugin

```
kubectl apply -n kube-system -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

* If you want to run containers on the muster than remove the master taint

```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

* Join worker servers to kubernetes cluster

```
kubeadm join 172.18.73.71:6443 --apiserver-advertise-address 172.18.73.72 --token xxxxxxxxxxx --discovery-token-ca-cert-hash sha256:asdjlkasjfljasfljsldjflsdj
```

* If you have multiple network interface like me than you need to add the following routes on the worker servers

```
ip route add 10.96.0.1/32 dev ens1(private_network_inteface)
kubectl get pods --all-namespaces - to check that all pods are running
```
