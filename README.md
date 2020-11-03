# kubemon-victoriametrics
All about Kubernetes monitoring using prometheus federation - victoriametrics/victoria-metrics

Some initial architecture and proof of concept with victoria-metrics - a time series database which can act as a prometheus federation platform as well as act as a datasource for Grafana.

I approached the work in following setup

| Syntax | Description |
| ----------- | ----------- |
| Header | Title |
| Paragraph | Text |


## A quick run with Docker Desktop 
---

| Syntax | Description |
| ----------- | ----------- |
| Header | Title |
| Paragraph | Text |


We have a free single-node VictoriaMetrics - fast time series database, long-term remote storage for Prometheus- in docker. 

```
Swarajits-MacBook-Air:~ swarajitroy$ docker pull victoriametrics/victoria-metrics
Using default tag: latest
latest: Pulling from victoriametrics/victoria-metrics
df20fa9351a1: Pull complete
6e80a8b898a4: Pull complete
687f13b71017: Pull complete
Digest: sha256:02e04c263ab4ebe5311d9675ac15e588878a0211aceb052d911b4fe5f4a4cb6b
Status: Downloaded newer image for victoriametrics/victoria-metrics:latest
docker.io/victoriametrics/victoria-metrics:latest
```
