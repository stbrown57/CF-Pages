---
weight: 999
title: "ONPSense Reverse Proxy"
description: ""
icon: "article"
date: "2023-09-21T13:51:38-04:00"
lastmod: "2023-09-21T13:51:38-04:00"
draft: true
toc: true
---

## Overview

All ingress into the BFNet network will have to enter the network from the public interface to the OPNsense gateway (browngw.6browns.org).  The NGINX reverse proxy will need to "route"  connections based on the host information on incoming packets, since all packets will be routed via IP to the public interface.  The documentation for [OPNsense identifies four TLS configurations](https://docs.opnsense.org/manual/reverse_proxy.html#) 

1.  Breaking up the connection on the firewall (down- and upstream are using TLS)
1.  Decrypt an encrypted upstream (downstream plain, upstream TLS protected)
1.  TLS Offloading (downstream is TLS protected, upstream is plain)
1.  TLS Passthough

The first and fourth options seem to be the most practicable, while option two (downstream plain) will be problematic with modern browsers that expect to connect securely.  Option three (downstream TLS, upstream plain), this may be an option for external access, but again browsers do not seem to like unencrypted connection that are designed in this model with internal connections.  I will look at this option, because it is the easiest and will provide a good start.

## TLS Offloading (downstream is TLS protected, upstream is plain)

This option will ultimately not be as useful as the two fully encrypted options, but it may have limited use and it will be the easiest to configure and test.

- [x] Create a HTTP target  
    Use NGINX-app as the target, this instance resides in the "dev" namespace.
    - [x] Point local DNS nginx-app.k3s.6browns.org to Traefik LB IP
    - [x] Test connection from GW
    
      ```bash
      root@browngw:~ # curl -Iv http://nginx-app.k3s.6browns.org
      ```

    - [x] Create proxy forward host on GW to "nginx-app.k3s.6browns.org"
      - [x] NGINX ->Configuration -> Upstream -> Upstream Server
            * Description: NGINX-app Upstream Server
            * Server: nginx-app.k3s.6browns.org
            * Port: 80
            * Server Priority: 1
      - [x] NGINX ->Configuration -> Upstream -> Upstream
            * Description: NGINX-app_Upstream
            * Server Entries: NGINX-app-Upstream-Server
- [x] Configure a TLS endpoint on GW
  - [x] HTTP Location
    * Description: NGINX app Redirect
    * URL Pattern: /
    * Upstream Servers: NGINX-app_Upstream
  - [x] HTTP Server
    * Server Name: nginx-app.k3s.6browns.org
    * Location: NGINX app Redirect
    * TLS Certificate: nginx-app.k3s.6browns.org (ACME Client) - previously provisioned

## Breaking up the connection on the firewall (down- and upstream are using TLS)

This option may be the most usefull. It will encrypt traffic both into the GW and from the GW to the target.  The downstream encryption will use a Let's Encrypt certificate as previous described. Traffic to the target will use an internal certificate signed by a CA on the GW. 

- [x] Create Internal certificate
  - [x] GW Trust -> Certificates
      * Method: Create an internal Cerificate
      * Descriptive Name: whoami-internal
      * Certificate Authority: bfnet-intermediate-ca
      * Type: Client and Server
      * Common Name: whoami.6browns.org
      * Alternate Names: DNS whoami.k3s.6browns.org
      * Create TLS secret in "whoami" namespace
    - [x] Install internal certificate
    
        ```bash
      kubectl create secret tls whoami.k3s.6browns.org-tls --key="whoami-internal.key" --cert="whoami-internal.crt" -n whoami
        ```
      - [x] Create Ingressroute with websecure endpoint and internal cert
      - [x] Test connection from GW
      
         ````bash
        root@browngw:~ # curl -Iv https://whoami.k3s.6browns.org
          ````
- [x] Create Upstream via HTTPS
- [x] Create redirect virtual server with Let's Encrypt endpoint
      
AWS test instance:

```bash
stephenbrown@Stephens-MBP .ssh %  ssh -i "accesstest.pem" ec2-user@ec2-44-203-146-109.compute-1.amazonaws.com
```

The connections failed. First the Cloudflare configuration for DNS-01 verification had to be reconfigured. Next I was receiving a 404 error from the curl test on the AWS instance. For whatever reason I had to edit the NGINX.conf file to change the proxy_pass directive from http to https.

The day after a successful connection, I started to get 505 errors. Looking at the "HTTP error" logs, it appeared the interconnection failed due to an untrusted certificate.  Curl on the GW works since I added the bfnet-ca to the trusted file "CAfile: /usr/local/etc/ssl/cert.pem". I'm guessing the NGINX system is not using this ROOT CA. Manually setting the configuration proxy_verify=off resolved the problem, but I will need to have the internal certs trusted by the NGINX implementation. 

```
upstream upstreambdc14352f04040f6864d11a0766a6153 {
server whoami.k3s.6browns.org:443 weight=1;

}
...
    proxy_pass **https**://upstreambdc14352f04040f6864d11a0766a6153;
```

## TLS Passthough

There may be instances where the secure channel can not be broken as was configured above. In this instance cert-manger can be used to generate a ACME cert and the firewall NGINX proxy can use SNI routing to pass traffic to the cluster. I will try this for Teleport.  

- [ ] Cert-mangaer generate certificate secrets
- [ ] Install Teleport chart, use the MetalLB
- [ ] Configure SNI NGINX routing
- [ ] Test with curl on AWS instance