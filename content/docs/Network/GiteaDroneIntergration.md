---
weight: 999
title: "Gitea Drone Intergration"
description: ""
icon: "article"
date: "2023-09-21T14:24:55-04:00"
lastmod: "2023-09-21T14:24:55-04:00"
draft: true
toc: true
---
* Token: <token>
* [ ] Create OAuth Application
  * Client ID: <ID>
  * Client Secret: <secret>
  * Application Name: drone
  * Redirect URI: http://derosa.k3s.6browns.org/login
* [ ] Create a shared secret
```bash
openssl rand -hex 16
<return value>
```
* [ ] Download drone on Derosa
```bash
 docker pull drone/drone
```
* [ ] Start server
```bash
DRONE_GITEA_CLIENT_ID=<ID>
DRONE_GITEA_CLIENT_SECRET=<secret>
DRONE_GITEA_SERVER=https://trek.6browns.org
DRONE_RPC_SECRET=<other secret>
DRONE_SERVER_HOST=derosa.6browns.org
DRONE_SERVER_PROTO=http

docker run \
  --volume=/mnt/HD1/var/lib/drone:/data \
  --env=DRONE_GITEA_SERVER={{DRONE_GITEA_SERVER}} \
  --env=DRONE_GITEA_CLIENT_ID={{DRONE_GITEA_CLIENT_ID}} \
  --env=DRONE_GITEA_CLIENT_SECRET={{DRONE_GITEA_CLIENT_SECRET}} \
  --env=DRONE_RPC_SECRET={{DRONE_RPC_SECRET}} \
  --env=DRONE_SERVER_HOST={{DRONE_SERVER_HOST}} \
  --env=DRONE_SERVER_PROTO={{DRONE_SERVER_PROTO}} \
  --publish=80:80 \
  --publish=443:443 \
  --restart=always \
  --detach=true \
  --name=drone \
  drone/drone
```

* [ ] Create Kubernette Runners
  * [ ] rbac drone service account
  * [ ] Example deployment
    * failed
```
time="2020-03-09T15:32:57Z" level=error msg="cannot ping the remote server" error="Post https://drone.6browns.org/rpc/v2/ping: x509: certificate is not valid for any names, but wanted to match drone.6browns.org"
```
  * Try this interactively and look at the drone port 3000, this is used by Grafana on Derosa