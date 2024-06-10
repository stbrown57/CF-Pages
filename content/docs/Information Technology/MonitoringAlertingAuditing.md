+++
weight = 999
title = "Monitoring, Alerting and Auditing"
description = ""
icon = "article"
date = "2023-09-21T14:57:41-04:00"
lastmod = "2023-09-21T14:57:41-04:00"
draft = true
toc = true
+++
## Overview ##

This should be self evident for obvious reasons.  Monitoring and review of logs on a production system will provide an understanding of the base level operation of the systems.  It may be a good practice to stress the systems while logging and collecting metrics to help define the operating parameters of the functional system. Knowing the limits of the system through testing will help define an alerting system for production monitoring and help define strategist for expanding the capacities of these systems. Alerting will notify  operators of potential impending problems, a change in workflow, and triggers to capacity changes.

## Metric Collection ##

* Prometheus and Grafana  
  A Prometheus server was installed on "DeRosa" with "node_exporters" on each of the other systems running in the environment.  The results are very cool in the simple load monitoring initially configured.  Prometheus does not store logs, so another method will be required either in addition to the Prometheus data store or as a replacement.  
  * Rancher Rio includes both Prometheus and Grafana in the stack.
  * Loki is Grafana log analysis application
* The "TICK" stack [(Telegraf, InfluxDB, Chronograf, Kapacitor)](https://www.influxdata.com/blog/how-to-use-the-open-source-tick-stack-to-spin-up-a-modern-monitoring-system-for-your-application-and-infrastructure/) may be a workable alternative.  InfluxDB accepts strings in time series, which has been use to store logs. Chronograf may be an alternative to Grafana.

## Log Collection and Analysis ##

There are a lot of reasons for collecting and analyzing logs and ultimately there is probably no single method in addressing these needs.  I looks like [Fluentd](https://www.fluentd.org/) was developed to cope with this problem.  Fluentd is a kind of log router and translator. It takes logs from many sources and forwards them to backend engins for storage and analysis.  Elasticsearch, AlienVault, and object stores.

* TICK stack
* Syslog and something to analize, alert and archive

## [Other Monitoring Systems](https://stackify.com/top-server-monitoring-tools/?utm_medium=pushnotifications&utm_source=onesignal&utm_campaign=onesignal) ##

This listening seems to be a mixed bag of traditional SNMP monitoring applications along with some cloud based metric monitoring.  

### NetData ###

Collect SoC temp
>
> The old BASH module can be used for this.
>
> To enable the old BASH module do these:
>
> 1. Disable sensors for python.d.plugin by editing /etc/netdata/python.d.conf and setting sensors: no.
> 2. Enable sensors for charts.d.plugin by editing /etc/netdata/charts.d.conf and setting sensors=force.
>
