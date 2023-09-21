---
weight: 999
title: "Kubernete Cluster"
description: ""
icon: "article"
date: "2023-09-21T15:39:22-04:00"
lastmod: "2023-09-21T15:39:22-04:00"
draft: true
toc: true
---
### Rancher

A cluster was added to the host pinarello, and Rancher was installed. Trying to add the felt cluster fails with a CrashLoopBackOff status.  The log of the pod shows:
```
sbrown@bianchi ~]$ kubectl logs -n cattle-system cattle-cluster-agent-6bb97bd9ff-qbdk2
INFO: Environment: CATTLE_ADDRESS=10.42.0.9 CATTLE_CA_CHECKSUM=0df5882f1714e8d8989215b7c25b59fd2cd623f62d568dd8adc34d1729c6fb06 CATTLE_CLUSTER=true CATTLE_CLUSTER_AGENT_PORT=tcp://10.43.129.180:80 CATTLE_CLUSTER_AGENT_PORT_443_TCP=tcp://10.43.129.180:443 CATTLE_CLUSTER_AGENT_PORT_443_TCP_ADDR=10.43.129.180 CATTLE_CLUSTER_AGENT_PORT_443_TCP_PORT=443 CATTLE_CLUSTER_AGENT_PORT_443_TCP_PROTO=tcp CATTLE_CLUSTER_AGENT_PORT_80_TCP=tcp://10.43.129.180:80 CATTLE_CLUSTER_AGENT_PORT_80_TCP_ADDR=10.43.129.180 CATTLE_CLUSTER_AGENT_PORT_80_TCP_PORT=80 CATTLE_CLUSTER_AGENT_PORT_80_TCP_PROTO=tcp CATTLE_CLUSTER_AGENT_SERVICE_HOST=10.43.129.180 CATTLE_CLUSTER_AGENT_SERVICE_PORT=80 CATTLE_CLUSTER_AGENT_SERVICE_PORT_HTTP=80 CATTLE_CLUSTER_AGENT_SERVICE_PORT_HTTPS_INTERNAL=443 CATTLE_CLUSTER_REGISTRY= CATTLE_INGRESS_IP_DOMAIN=sslip.io CATTLE_INSTALL_UUID=2ce92d8c-5a0a-485e-83cd-910a362cf557 CATTLE_INTERNAL_ADDRESS= CATTLE_IS_RKE=false CATTLE_K8S_MANAGED=true CATTLE_NODE_NAME=cattle-cluster-agent-6bb97bd9ff-qbdk2 CATTLE_SERVER=https://pinarello.6browns.org CATTLE_SERVER_VERSION=v2.6.6
INFO: Using resolv.conf: search cattle-system.svc.cluster.local svc.cluster.local cluster.local 6browns.org nameserver 10.43.0.10 options ndots:5
ERROR: https://pinarello.6browns.org/ping is not accessible (Could not resolve host: pinarello.6browns.org)
```
The dnsutils pod shows this as a responce to pinarello.6browns.org:
```
[sbrown@bianchi ~]$ kubectl exec -it dnsutils dig pinarello.6browns.org
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.

; <<>> DiG 9.9.5-9+deb8u19-Debian <<>> pinarello.6browns.org
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5865
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;pinarello.6browns.org.		IN	A

;; ANSWER SECTION:
pinarello.6browns.org.	5	IN	A	192.168.1.251

;; Query time: 8 msec
;; SERVER: 10.43.0.10#53(10.43.0.10)
;; WHEN: Wed Jul 13 01:13:40 UTC 2022
;; MSG SIZE  rcvd: 87
```
dnsutils is running on Tarmac, the register pod is running on Felt.

Attempts to delete and add the external cluster seems to have put the rancher cluster in an unstable state. I ran "rancher-cleanup" to delete the Rancher instance and reinstalled Rancher with helm.

```
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=192.168.1.251.sslip.io \
  --set replicas=1 \
  --set bootstrapPassword=vall27goff
```
Use "https://192.168.1.251.sslip.io" as the Rancher URL in this attempt.

This is the error:
```
INFO: Using resolv.conf: search cattle-system.svc.cluster.local svc.cluster.local cluster.local 6browns.org nameserver 10.43.0.10 options ndots:5
ERROR: https://192.168.1.251.sslip.io/ping is not accessible (Could not resolve host: 192.168.1.251.sslip.io)
```
The proper response "pong" is returned if accessed from an external laptop.

Run [diagnostic pods on Felt](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodename) to curl the ping URL and resolve the referenced name within the cluster, use **nodename** selector

### Cert-Manager

#### Test out the installation of Cert-Manager.

```
[sbrown@bianchi ~]$ cmctl check api
Not ready: Internal error occurred: failed calling webhook "webhook.cert-manager.io": failed to call webhook: Post "https://cert-manager-webhook.cert-manager.svc:443/mutate?timeout=10s": context deadline exceeded
```
#### Reinstall
After deleting all cert artifacts...

```
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.8.2 \
  --set installCRDs=true
```

Error:
```
Error: failed post-install: timed out waiting for the condition
```
The webhook pod was scheduled on Tarmac, There may be some resolution issues on Tarmac and Felt. (Fedora Server) or possible connectivity issues. 

Resolution on a dnsutils instance was successful.



```
[sbrown@bianchi Kubernetes]$ kubectl exec -i -t dnsutils -- nslookup cert-manager-webhook.cert-manager
Server:		10.43.0.10
Address:	10.43.0.10#53

Name:	cert-manager-webhook.cert-manager.svc.cluster.local
Address: 10.43.225.90
```

This was recommended on a help tread, and it seem to work for some reason.

```
$ kubectl delete mutatingwebhookconfiguration.admissionregistration.k8s.io cert-manager-webhook
$ kubectl delete validatingwebhookconfigurations.admissionregistration.k8s.io cert-manager-webhook
```
API Check
```
[sbrown@bianchi ~]$ cmctl check api
The cert-manager API is ready
[sbrown@bianchi ~]$ 
```
## DNS resolution CoreDNS

### Felt - Fedora Server

When the CoreDNS pod was scheduled on this node all requests to the DNS service from cluster pods timed out. This is possibly a network connectivity problem with the cluster networking (flannel).

### Look - FedoraCore

There is an issue with systemd-resolvers, and Kubernette that may cause a loop and fail to forward queries to the external DNS, typically defined in the /etc/resolv.conf file. A suggested resolution is to add a flag to the k3s service definition (kubernet/k3s program). Entering the following may resolve the problem. I believe these flags can be add to a configuration file for k3s instead of the systemd service definition.

```
ExecStart=/usr/local/bin/k3s \
    agent \
        'resolv-conf' \
        '/run/systemd/resolve/resolv.conf' \
        '--selinux' \

```

### Canyon - Armbian (Debian based ARM distro)

This default configuration seems to work as expected. Queries for external hosts though nslookup require a terminating dot. If it is not terminated the search domain is appended.

```
root@dnsutils:/# nslookup google.com
Server:		10.43.0.10
Address:	10.43.0.10#53

Non-authoritative answer:
Name:	google.com.6browns.org
Address: 173.72.245.3
```

This could be happening because the upstream DNS server (colnago.6browns.org - dnsmasq) is responding with the public IP for any unknown host on the 6browns.org domain.

From a k3s issue thread...

```
In addition to configuring k3s with environment variables and cli arguments, k3s can also use a config file.

By default, values present in a yaml file located at /etc/rancher/k3s/config.yaml will be used on install.

An example of a basic server config file is below:

# /etc/rancher/k3s/config.yaml
write-kubeconfig-mode: "0644"
tls-san:
  - "foo.local"
node-label:
  - "foo=bar"
  - "something=amazing"
In general, cli arguments map to their respective yaml key, with repeatable cli args being represented as yaml lists.

An identical configuration using solely cli arguments is shown below to demonstrate this:

k3s server \
  --write-kubeconfig-mode "0644"    \
  --tls-san "foo.local"             \
  --node-label "foo=bar"            \
  --node-label "something=amazing"
It is also possible to use both a configuration file and cli arguments. In these situations, values will be loaded from both sources, but cli arguments will take precedence. For repeatable arguments such as --node-label, the cli arguments will overwrite all values in the list.
```

Dig seems to work as expected. This may just be the way it works.