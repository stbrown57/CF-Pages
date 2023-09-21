---
weight: 999
title: "K3s Rebuild"
description: ""
icon: "article"
date: "2023-09-21T15:40:51-04:00"
lastmod: "2023-09-21T15:40:51-04:00"
draft: true
toc: true
---
## Remove current cluster
* On worker node:

```bash
sudo /var/local/bin/k3s-agent-uninstall.sh
sudo /var/local/bin/k3s-killall.sh
sudo rm -R /var/lib/rancher/k3s/agent
sudo rm -R /var/lib/rancher/k3s/data
sudo rm -R /var/lib/rancher/k3s/server
```
* On the master:

```bash
sudo /var/local/bin/k3s-uninstall.sh
sudo /var/local/bin/k3s-killall.sh
sudo rm -R /var/lib/rancher/k3s/agent
sudo rm -R /var/lib/rancher/k3s/data
sudo rm -R /var/lib/rancher/k3s/server
```

##  ...or image a new K3os instance
* Burn Armbian image to ssd card
  * Shutdown cluster and remove all modules
  * Image sopine0
  * install module0
  * boot
### ...try
  * Armbian_20.11_Pine64so_bionic_current_5.8.16
    * No ethernet
  * Armbian_20.11.3_Pine64so_focal_current_5.9.14
### Modify [patch]( https://forum.pine64.org/showthread.php?tid=10432) image to enable Ethernet
1. Find offset 

```
 fdisk -l  <imagefile>

```
sector * start = offset

```
sudo mount -o loop,rw,sync,offset=<offset> <imagefile> /mnt/img

```
2. Edit dtb file add delay 500
3. Boot and install K3os

```
curl -sfL https://github.com/rancher/k3os/releases/download/v0.19.4-dev.5/k3os-rootfs-arm64.tar.gz | tar zxvf - --strip-components=1 -C /
```
Copy config with key and unique host name to node
 Check the directory /var/lib/rancher/k3s/storage were # is the node number
```
cp myconfig#.yaml /k3os/system/config.yaml
sync
```
## Installation
* On Master:  
  Disable "servicelbl", since we will be installing "metallb" as a load balancer.

```bash
sudo curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--no-deploy servicelbl" sh -s -
```
* collect the token:
* 
```
K3S_TOKEN -> /var/lib/rancher/k3s/server/node-token

```
* On worker nodes:
* 
```bash
sudo curl -sfL https://get.k3s.io | K3S_URL=https://sopine0.6browns.org:6443 K3S_TOKEN=K10cbfb18d6cc1d1466a93bd4412e086ce7fb61a6d7eecfaf3c179d84b252595292::server:caad6f691ddb92aff4c94edb7154a791 sh -
```
## Configure Cluster Access
Create a config
> Copy /etc/rancher/k3s/k3s.yaml on your machine located outside the cluster as ~/.kube/config. Then replace “localhost” with the IP or name of your K3s server. kubectl can now manage your K3s cluster.

## Addition systems
### [Metallb](https://metallb.universe.tf/installation/)

```bash
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.9.3/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.9.3/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```
Configure load balancer
```bash
kubectl apply -f metallb-config.yaml
```

### [Cert-Manager](https://cert-manager.io/docs/installation/kubernetes/)
```bash
# Kubernetes 1.15+
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.1/cert-manager.crds.yaml
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.14.1
```
Output
```
NAME: cert-manager
LAST DEPLOYED: Mon Nov 23 19:37:22 2020
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
cert-manager has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://cert-manager.io/docs/configuration/

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://cert-manager.io/docs/usage/ingress/
```

### [Dashboard](https://rancher.com/docs/k3s/latest/en/installation/kube-dashboard/)
```bash
GITHUB_URL=https://github.com/kubernetes/dashboard/releases
VERSION_KUBE_DASHBOARD=$(curl -w '%{url_effective}' -I -L -s -S ${GITHUB_URL}/latest -o /dev/null | sed -e 's|.*/||')
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/${VERSION_KUBE_DASHBOARD}/aio/deploy/recommended.yaml

kubectl apply -f dashboard.admin-user.yaml 
kubectl apply -f dashboard.admin-user-role.yaml 
````

### Get Token for Dashboard
[Kubernetes Dashboard documentation](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)

Collect token
```
 kubectl get secrets -n kubernetes-dashboard
```

Find admin-user-token instance and describe
```
kubectl describe secret admin-user-token-<instance> -n kubernetes-dashboard
```

Run proxy and connect

```
kuberctl proxy
```

[connect...](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/)