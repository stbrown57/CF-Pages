---
weight: 999
title: "CoreDNS Troubleshooting"
description: ""
icon: "article"
date: "2023-09-21T14:28:17-04:00"
lastmod: "2023-09-21T14:28:17-04:00"
draft: true
toc: true
---

## Test Cluster
Run a quick Alpine instance for debugging:
```bash
kubectl run -it --rm --restart=Never alpine --image=alpine sh
```
1. Ping gateway on another node  
  ```
3: eth0@if21: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue state UP 
    link/ether aa:82:29:16:75:a2 brd ff:ff:ff:ff:ff:ff
    inet 10.42.1.3/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::a882:29ff:fe16:75a2/64 scope link 
       valid_lft forever preferred_lft forever
  ping 10.42.2.1
PING 10.42.2.1 (10.42.2.1): 56 data bytes
64 bytes from 10.42.2.1: seq=0 ttl=63 time=55.057 ms
  ```
2. Lookup external host name  
  ```bash
/ # nslookup slashdot.org
Server:		10.43.0.10
Address:	10.43.0.10:53

Non-authoritative answer:
Name:	slashdot.org
Address: 216.105.38.15
  ```

3. Ping host name (failed)
  ```bash
/ # ping slashdot.org
PING slashdot.org (173.72.137.37): 56 data bytes
  ```
4. Ping IP address (succeeded)
  ```bash
/ # ping 216.105.38.15
PING 216.105.38.15 (216.105.38.15): 56 data bytes
64 bytes from 216.105.38.15: seq=0 ttl=53 time=109.920 ms
  ```
5. Activate loging on CoreDNS queries
```bash
kubectl get configmap -n kube-system coredns -o json |  kubectl get configmap -n kube-system coredns -o json | sed -e 's_loadbalance_log\\n    loadbalance_g' | kubectl apply -f -
```
  .. or
  edit CoreDNS Config and add  "log" directive
```bash
$ kubectl edit configmap coredns -n kube-system
```
  add data.Corefile.:53.log

## Symtoms
- Dig from a pod connects to the coredns server on port 53 udp and resolves as expected.  www.google.com is recognized as a FQDN
- Ping from a pod connects to the coredns server on a port other than 53 (but appear to be in the fifties)  and attempts to resolve as a relative name (the search domains are added sequentially). Using the terminating dns "dot" will resolve the name as a FQDN.  ie. "www.google.com." works, note the terminating period (dot).

## Google Search Review
A Google search on the symptoms observed provide a number of articles that may apply to the problem.
- [UDP Connection Tracking and DNS](https://www.contentful.com/blog/2020/01/07/coredns-nodecache-blog/)
-  Diagnostic container in the CoreDNS pod (sidecar?)
- Kubernetes "service" with an external "endpoint"
- CoreDNS plugins