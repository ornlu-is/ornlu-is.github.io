---
title: "Six Degrees of Kevin Bacon 3: Collecting the Data"
date: 2023-07-16T15:19:59+01:00
categories: ["Six Degrees of Kevin Bacon"]
description: "Web scraping movie and actor information to build a graph of all actors, connecting them via movies they worked together on."
draft: true
---

{{< admonition type=tip title="Other posts in this series" open=true >}}
* [Six Degrees of Kevin Bacon 1: Introduction](https://ornlu-is.github.io/six_degrees_of_kevin_bacon_1/)
* [Six Degrees of Kevin Bacon 2: Web Scraping Movie Data](https://ornlu-is.github.io/six_degrees_of_kevin_bacon_2/)
{{< /admonition >}}

## Planning

{{< figure src="/images/six_degrees_of_kevin_bacon_3/get_movie_links.png" title="Sequence diagram for the data collection" >}}

## Crawler

takes care of collecting the links where the data exists

## Deduplicator

Implemented as a Golang dictionary, which works like a hash map

## Queue

Standard FIFO data structure

## Scraper

handles fetching the actual data from the cast links

## Storage

fancy term for writing stuff into plain files

## Monitoring

Prometheus + Grafana (+ maybe Loki?)

## Next Step

next step is to analyze stuff using Python and graph-tools
