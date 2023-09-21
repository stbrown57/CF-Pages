---
weight: 999
title: "6browns.org Web Routing"
description: ""
icon: "article"
date: "2023-09-21T14:19:39-04:00"
lastmod: "2023-09-21T14:19:39-04:00"
draft: true
toc: true
---

## Tasks
* [x] Register wildcard record with DynDNS
  * \*.6browns.org -> public 
  * How about DNS authentication for LetsEncrypt
* [x] Port forward http and https to Trek internal IP
* [x] Configure Caddy2 to reverse proxy appropriate hostnames
  * This assumes Caddy works as an HTTPS endpoint and proxies the backend, much like what I have traditionally done with NGINX and WordPress
```config
gitea.6browns.org {
# Templates give static sites some dynamic features
#templates
  # Compress responses according to Accept-Encoding headers
  encode gzip zstd
  # Make HTML file extension optional
  try_files {path}.html {path}
  # Send requests to /gitea to backend
  reverse_proxy localhost:3000
}
trek.6browns.org {
	file_server
}
drone.6browns.org {
	encode gzip zstd
	reverse_proxy derosa.6browns.org:80
}
```
## Test connections
* [x] Trek.6browns.org
  ```bash
# curl -I https://trek.6browns.org
HTTP/2 200 
accept-ranges: bytes
content-type: text/html; charset=utf-8
etag: "q1t8v31u7"
last-modified: Sun, 01 Dec 2019 01:45:03 GMT
server: Caddy
content-length: 2383
date: Sun, 08 Mar 2020 23:45:46 GMT
```
 * [x] gitea.6browns.org
  ```bash
# curl -I https://gitea.6browns.org
HTTP/2 200 
date: Sun, 08 Mar 2020 23:49:44 GMT
server: Caddy
set-cookie: lang=en-US; Path=/; Max-Age=2147483647
set-cookie: i_like_gitea=452cbdb05616bd3f; Path=/; HttpOnly
set-cookie: _csrf=Vw-au-MsKAcOoPfMgyJMT8daMJY6MTU4MzcxMTM4NDI2MTQ3MDYzMA%3D%3D; Path=/; Expires=Mon, 09 Mar 2020 23:49:44 GMT; HttpOnly
x-frame-options: SAMEORIGIN
```
* [x] drone.6browns.org
```bash
$ curl -I https://drone.6browns.org
HTTP/2 200 
cache-control: no-cache, no-store, must-revalidate, private, max-age=0
content-type: text/html; charset=UTF-8
date: Mon, 09 Mar 2020 02:00:31 GMT
expires: Thu, 01 Jan 1970 00:00:00 UTC
pragma: no-cache
server: Caddy
x-accel-expires: 0
x-frame-options: DENY
x-xss-protection: 1; mode=block
content-length: 786

```
