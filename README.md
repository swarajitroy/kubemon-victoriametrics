# kubemon-victoriametrics
All about Kubernetes monitoring using prometheus federation - victoriametrics/victoria-metrics

Some initial architecture and proof of concept with victoria-metrics - a time series database which can act as a prometheus federation platform as well as act as a datasource for Grafana.

I approached the work in following setup

| Task ID | Task Name | Remarks
| ----------- | ----------- | ------|
| 01 | Install VictoriaMetrics stand-alone on Docker Desktop |
| 02 | Install Grafana stand-alone on Docker Desktop |
| 03 | Install Prometheus stand-alone on Docker Desktop | 
| 04 | Install a custom scraping endpoint (Java Springboot or Standalone Java)|
| 05 | Using Docker Compose - install this as a set on Docker Desktop  |
| 06 | Use case definition and demo with Docker Desktop |
| 07 | Install VictoriaMetrics stand-alone as a Pod into CNCF k8s cluster |
| 08 | Install Grafana stand-alone as a Pod into CNCF k8s cluster |
| 09 | Install Prometheus stand-alone as a Pod into CNCF k8s cluster |
| 10 | Install a custom scraping endpoint (Java Springboot or Standalone Java) |
| 11 | Using a HELM chart - install this as a set on CNCF K8S cluster |
| 12 |  Use case definition and demo with CNCF K8S cluster | |



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

```
Swarajits-MacBook-Air:lib swarajitroy$ docker run -d  --rm -v victoriametrics_volume:/victoria-metrics-data -p 8428:8428 victoriametrics/victoria-metrics
30eae1b457e7892ea14aa6f2cdce846e8b59ab3cec1bcf2aa22ad6d7496a6544

Swarajits-MacBook-Air:lib swarajitroy$ docker ps | grep -i victoriametrics
30eae1b457e7        victoriametrics/victoria-metrics             "/victoria-metrics-p…"   About a minute ago   Up About a minute   0.0.0.0:8428->8428/tcp   frosty_shockley

docker logs 30eae1b457e7
2020-11-03T17:09:37.236Z	info	VictoriaMetrics/lib/logger/flag.go:12	build version: victoria-metrics-20201102-004452-tags-v1.45.0-0-gd396c265a
2020-11-03T17:09:37.254Z	info	VictoriaMetrics/app/victoria-metrics/main.go:37	starting VictoriaMetrics at ":8428"...
2020-11-03T17:09:37.320Z	info	VictoriaMetrics/app/victoria-metrics/main.go:46	started VictoriaMetrics in 0.066 seconds
2020-11-03T17:09:37.322Z	info	VictoriaMetrics/lib/httpserver/httpserver.go:82	starting http server at http://:8428/
2020-11-03T17:09:37.323Z	info	VictoriaMetrics/lib/httpserver/httpserver.go:83	pprof handlers are exposed at http://:8428/debug/pprof/

```

```
Swarajits-MacBook-Air:lib swarajitroy$ curl -v http://localhost:8428/metrics
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8428 (#0)
> GET /metrics HTTP/1.1
> Host: localhost:8428
> User-Agent: curl/7.63.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: text/plain
< Date: Tue, 03 Nov 2020 17:16:05 GMT
< Transfer-Encoding: chunked
<
vm_active_force_merges 0
vm_active_merges{type="indexdb"} 0
vm_active_merges{type="storage/big"} 0
vm_active_merges{type="storage/small"} 0
vm_assisted_merges_total{type="indexdb"} 0
vm_assisted_merges_total{type="storage/small"} 0
vm_blocks{type="indexdb"} 0
vm_blocks{type="storage/big"} 0
vm_blocks{type="storage/small"} 0
vm_cache_collisions_total{type="storage/metricName"} 0
vm_cache_collisions_total{type="storage/tsid"} 0
vm_cache_entries{type="indexdb/dataBlocks"} 0
vm_cache_entries{type="indexdb/indexBlocks"} 0
vm_cache_entries{type="indexdb/tagFilters"} 0
```


