---
weight: 999
title: "Kubernetes Sidecar"
description: ""
icon: "article"
date: "2023-09-21T15:31:07-04:00"
lastmod: "2023-09-21T15:31:07-04:00"
draft: true
toc: true
---
Create a "sidecar" container to run "tcpdump" in the pod.
- Identify an  appropriate image with tcpdump  
  - d97jro/tcpdump
- Create a yaml file to modify the 

```yaml
Name:         coredns-6c6bb68b64-ncffx
Namespace:    kube-system
Labels:       k8s-app=kube-dns
              pod-template-hash=6c6bb68b64
Containers:
  tcpdump:
    Image: d97jro/tcpdump
```

I'm not sure how you are suppose to run this image, so I'll create a standard alpine container that I will install tcpdump once it is in the pod.  The container may have to be updated later with a volume for the collection.

Edit deployment config...

```bash
 kubectl edit deployment coredns -n kube-system

     containers:
>      - command:
>         - /bin/sleep
>         - infinity
>       image: alpine
>        imagePullPolicy: Always
>        name: tcpdump
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      - args:
        - -conf
        - /etc/coredns/Corefile
```
Connect to the container and install tcpdump
```bash
kubectl exec -it coredns-6457d5c768-7s95p -c tcpdump /bin/sh -n kube-system
# apk update
# apk add tcpdump
```