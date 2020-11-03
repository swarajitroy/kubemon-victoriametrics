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

```
Swarajits-MacBook-Air:~ swarajitroy$ docker volume ls
DRIVER              VOLUME NAME
local               0cd8d968b41c3ff2c841c44affafd0ea4526a013f164042f210ac30169f73c74
local               1174af00f1c9f001080080eb6ad4c05c397b8c4fbb849c7b0b7f4a12c36212ff
local               qm1data
local               qmdata
local               victoriametrics_volume
Swarajits-MacBook-Air:~ swarajitroy$ docker volume ls | grep victoria
local               victoriametrics_volume
Swarajits-MacBook-Air:~ swarajitroy$ docker volume inspect victoriametrics_volume
[
    {
        "CreatedAt": "2020-11-03T16:59:25Z",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/victoriametrics_volume/_data",
        "Name": "victoriametrics_volume",
        "Options": {},
        "Scope": "local"
    }
]
```
