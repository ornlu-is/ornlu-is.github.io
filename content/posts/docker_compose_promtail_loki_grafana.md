---
title: "Hooking Promtail, Loki, and Grafana to your Docker Compose stack"
date: 2023-08-11T11:46:41+01:00
description: "How to add Loki, promtail, and Grafana to a local docker compose stack"
categories: ["Docker", "Grafana"]
draft: false
---

When testing software locally, one of the main tools at a software engineer's disposal is Docker, more particularly, the Docker Compose tool. This tool allows engineers to define and run a multi-container setup using YAML files. The vast majority of software produces a form of output known as logs, which provide information on what is happening in a running application, such as errors, possible warnings, general information, and more. 

## The issue with inspecting Docker Compose log messages

When you run a Docker Compose setup locally, you'll usually see all container logs being printed in rapid succession in your terminal, and, depending on the amount of logs your multi-container stack produces, this might be very difficult to navigate, reason about, and correlate log messages from different containers. Moreover, if you are interested in a single container's logs, you'd have to get the container ID from the output of `docker ps` and then get the logs with `docker logs <CONTAINER ID>`. And even then, you'd only see them in plaintext format printed on the terminal, which is a poor way of navigating logs if there are many of them. Surely there must be a better way.

## Your favourite super hero to the rescue: Loki and his sidekick Promtail

Loki is a log aggregation system designed by Grafana Labs and, most of all, it's open source software, which is important because I don't like paying for stuff that I can get for free. To add Loki to a Docker Compose stack, we have to first understand how it works. In itself, Loki is just an aggregation and storage system, it does not collect the logs by itself, nor does it provide a nice user interface to explore them. This means that we need to other components: one for collecting the logs, and another to query and visualise them. 

To collect the logs, we'll use a piece of software called Promtail, which is an agent that takes local logs and sends them to a Loki instance. Additionally, it also handles target discovery and attaching labels to log streams. Finally, to query and visualise the logs, we'll simply use a Grafana instance where we'll use our Loki instance as a data source.

## The application we are monitoring

Since this is just an example to showcase how to set up this particular monitoring stack, I created the most basic application I could think of. All it does is start and print a Monty Python quote every second:

```go
package main

import (
	"log"
	"time"
)

func main() {
	for {
		log.Println("Oh! Now we see the violence inherent in the system! Help, help, I'm being repressed!")
		time.Sleep(1 * time.Second)
	}
}
```

{{< admonition type=tip title="Containerising the application" open=true >}}
To run this application with Docker Compose, it has to be containerised. For this, I used a build-step container and you can read more about this on a previous post that I wrote:
* https://ornlu-is.github.io/slim_docker_images/
{{< /admonition >}}

## Setting up the log monitoring stack with Docker Compose

There are several steps to this process. The first one is to create a `docker-compose.yaml` file with the following contents:

```yaml
version: "3.8"

services:
  
  loki:
    container_name: loki
    hostname: loki
    image: grafana/loki
    ports:
      - 3100:3100
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - ./loki:/etc/loki/
  
  promtail:
    container_name: promtail
    hostname: promtail
    image: grafana/promtail
    command: -config.file=/etc/promtail/docker-config.yaml
    volumes:
      - ./promtail/docker-config.yaml:/etc/promtail/docker-config.yaml
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      - loki

  grafana:
    container_name: grafana
    hostname: grafana
    image: grafana/grafana
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
      - GF_AUTH_DISABLE_LOGIN_FORM=true
    ports:
      - 3000:3000
    volumes:
      - ./grafana/provisioning/:/etc/grafana/provisioning/
    depends_on:
      - promtail

  app:
    container_name: app
    hostname: app
    build:
      context: .
      dockerfile: Dockerfile
    labels:
      logging: "promtail"
      logging_jobname: "container_logs"
    depends_on:
      - grafana
```

If you didn't limit yourself to copying the YAML file above and actually read it, you probably have some questions. There are a bunch of bind mounts defined in our `docker-compose.yaml` file. Each of them serves a different purpose and we'll go through them one by one.

{{< admonition type=tip title="Bind mounts" open=true >}}
Bind mounts are basically files or directories on the host machine that are mounted into a container, meaning that the container can access these files/directories.
{{< /admonition >}}

For Loki to know how it is configured, it needs, you've guessed it, a configuration file! So we create a directory aptly named `loki`, and place a `local-config.yaml` file inside with the following configuration:

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h
```

Note that this is just the default Loki configuration for a Loki instance running locally, so there is nothing particularly noteworthy here. However, the same is not true for Promtail! As before, we create a `promtail` directory with a `docker-config.yaml` inside it:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: scraper
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
        filters:
          - name: label
            values: ["logging=promtail"]
    relabel_configs:
      - source_labels: ["__meta_docker_container_name"]
        regex: "/(.*)"
        target_label: "container"
      - source_labels: ["__meta_docker_container_log_stream"]
        target_label: "logstream"
      - source_labels: ["__meta_docker_container_label_logging_jobname"]
        target_label: "job"
```

Let us unpack this. In our `docker-compose.yaml` file we have added two labels to our application: the `logging` and the `logging_jobname` labels, with values `promtail` and `container_logs`, respectively. This is what is going to be used by Promtail to know what to scrape, *i.e.*, what logs to collect, and how to relabel them. Additionally, we also have to specify the `host` under `docker_sd_configs` carefully, because it must be `unix:///var/run/docker.sock`, which is the socket to which Docker writes the application logs.

There is only one thing left to configure, and that is Grafana. We will create yet another directory with the most unexpected name ever, `grafana`, and we'll create two additional directories inside it: `datasources` and `provisioning`. Inside `provisioning`, we'll create a `dashboard.yaml` file with the following contents:

```yaml
apiVersion: 1

providers:
  - name: 'Loki'
    orgId: 1
    folder: ''
    type: file
    editable: true
    options:
      path: /etc/grafana/provisioning/dashboards
```

And, inside the `datasources` directory, we'll create a `datasources.yaml` file to configure Loki as a data source for Grafana:
```yaml
apiVersion: 1

datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://localhost:3100
    isDefault: true
```

## Pre-built dashboards in a Docker Compose Grafana instance

It is obviously very boring and laborious to always have to create the dashboard from scratch just so we can explore the log messages properly. Fortunately, this actually only needs to be done once! Spin up the Docker Compose environment and navigate to your Grafana instance at `http://localhost:3000`. Create a dashboard as you normally would and then, when you save it, copy its JSON definition to the clipboard. To have this dashboard created by default, just paste this JSON into a `my_dashboard.json` file and place it inside the `grafana/provisioning` directory. And you're done, you'll never have to create that dashboard ever again.

## Dealing with the 'Unauthorized' error from the Grafana dashboard

There is a chance that you get an "Unauthorized" error when attempting to access the Grafana dashboard on your Docker Compose Grafana instance. Fret not because this is exceedingly easy to solve. Simply open `http://localhost:3000` in a private/incognito browser and you'll no longer have this issue. This error pops up whenever you have already opened another Grafana instance in your browser before and thus your browser saved some Grafana session cookies that it then attempts to use for the instance you are running with Docker Compose. Obviously/hopefully this fails because these cookies are not a valid authorization method for the Grafana instance you just spun up.

## Solving the 'too many outstanding requests' error

At the time of writing this, Loki is not shipped with sensible defaults. This translates to, if you build a few panels in a single dashboard, you start getting empty panels and a "Too many outstanding requests" error in Grafana (I got this error after adding 6 panels, which is very little). This is because Loki is shipped with a few limits on how many requests from Grafana it can handle. After tinkering with these limits for a while, I found that adding the following lines to the Loki config file solved that issue:

```yaml
limits_config:
  split_queries_by_interval: 24h
  max_query_parallelism: 100

query_scheduler:
  max_outstanding_requests_per_tenant: 4096

frontend:
  max_outstanding_per_tenant: 4096
```

And now you're all set to efficiently explore logs from Docker Compose environments.

{{< admonition type=tip title="Link to the code" open=true >}}
https://github.com/ornlu-is/docker_compose_loki_example
{{< /admonition >}}
