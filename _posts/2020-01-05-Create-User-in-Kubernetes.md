---
layout: post
title: "Create User in Kubernetes"
date: 2020-01-05
---

## Tested On
OS: Ubuntu 18.04  
Kubernetes Version: v1.17.0  

This guide will show you how to create user in kubernetes and use it inside a bash script to run some automate tasks.

Here I will show how to create a backup job in Jenkins for chef server that runs inside kubernetes.

## Create Kubernetes User

* Create a jenkins-robot service account and bind it to cluster-admin role

```
kubectl -n chef create serviceaccount jenkins-robot
kubectl -n chef create rolebinding jenkins-robot-binding --clusterrole=cluster-admin --serviceaccount=chef:jenkins-robot
```

* get service account token name and decrypt the token with base64

```
TOKEN_NAME=$(kubectl -n chef get serviceaccount jenkins-robot -o go-template --template='{{range .secrets}}{{.name}}{{"\n"}}{{end}}')
kubectl -n chef get secrets ${TOKEN_NAME} -o go-template --template '{{index .data "token"}}' | base64 -d  
```

## Create a Jenkins job

* Upload jenkins-robot token to jenkins credentials as a secret text
* Upload kubernetes apiserver certificate to jenkins credentials as a secret file. default file location: /etc/kubernetes/pki/apiserver.crt in the control plain server
* Example for a bash script that I use:

```
#!/bin/bash
Configure kubectl
PATH=${PATH}:~/bin/
if [ ! -x ~/bin/kubectl ]
then
  curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.17.0/bin/linux/amd64/kubectl
  chmod +x ./kubectl
  mkdir ~/bin/
  mv ./kubectl ~/bin/kubectl
fi
kubectl config set-cluster prod --server=https://k8s-cp.example.com:6443 --certificate-authority=${CA}
kubectl config set-credentials jenkins-robot --token=${TOKEN}
kubectl config set-context prod --cluster=prod --namespace=default --user=jenkins-robot
kubectl config use-context prod
POD_NAME=chef-0
kubectl -n chef exec -i ${POD_NAME} -- chef-server-ctl backup --yes
TAR_FILE=$(kubectl -n chef exec -i ${POD_NAME} -- ls -lrt /var/opt/chef-backup/ | tail -1 | awk '{print $NF}')
rm -f chef-backup*.tgz
kubectl -n chef cp ${POD_NAME}:/var/opt/chef-backup/${TAR_FILE} ${TAR_FILE}
```

* Upload tar file to s3 with publish artifacts to s3 bucket
