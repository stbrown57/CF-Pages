---
weight: 999
title: "Cloudflare and Kubernetes"
description: ""
icon: "article"
date: "2023-09-21T15:19:46-04:00"
lastmod: "2023-09-21T15:19:46-04:00"
draft: true
toc: true
---
Cloudflared tunnel on K3s, connect to ingress, instead of service. That would allow trusted certificates from both internal sources and the CF system.

One post says, Cloudflare tunnel can only link to a service. In that case the ingress can be made for the internal service, and some other means for the tunnel. At the moment the noTLScheck option is set.

https://blog.cloudflare.com/automated-origin-ca-for-kubernetes/


https://blog.cloudflare.com/cloudflare-ca-encryption-origin/

K3s cloudflared tunnel, check "access" option.
## Gitea

The instance is running using the included DB. The ROOT_URL parameter is not set correctly, and I need to review other option changes.

- [ ] Update values.yaml with ROOT_URL, now values-interalhadb.yaml
- [ ] Update helm instance
Use secrets for the Gitea admin username and password

Oauth configuration or LDAP
