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
| 12 | Use case definition and demo with CNCF K8S cluster | |
| 13 | Install VictoriaMetrics stand-alone as a Pod into Openshift 4.5 cluster |
| 14 | Install Grafana stand-alone as a Pod into Openshift 4.5 cluster |
| 15 | Install Prometheus stand-alone as a Pod into Openshift 4.5 cluster |
| 16 | Install custom scraping endpoint (Java Springboot or Standalone Java) as a Pod into Openshift 4.5 cluster |
| 17 | Use case definition and demo with Openshift 4.5 cluster | 
| 18 | Run VictoriaMetrics as a Kubernetes Operator |
| 19 | Solution Architecture Quality - High Availability | |
| 20 | Solution Architecture Quality - Security | |
| 21 | Solution Architecture Quality - Performance Benchmark | |
| 22 | Solution Architecture Quality - Observability | |




## 01. Install VictoriaMetrics stand-alone on Docker Desktop
---

| Task ID | Task Name | Remarks
| ----------- | ----------- | ------|
| A | Pull latest Victoriametrics image from Docker Hub | |
| B | Create a Docker volume for Victoriametrics data storage |
| C | Run Victoriametrics as a docker container with docker volume attached |
| D | Test Victoriametric own metric endpoint URL |


### 01.A Pull latest Victoriametrics image from Docker Hub
---

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

### 01.B Create a Docker volume for Victoriametrics data storage
---

```
Swarajits-MacBook-Air:~ swarajitroy$ docker volume create victoriametrics_volume
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

### 01.C Run Victoriametrics as a docker container with docker volume attached
---

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

### 01.D Test Victoriametric own metric endpoint URL
---

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

## 02. Install Prometheus stand-alone on Docker Desktop
---

| Task ID | Task Name | Remarks
| ----------- | ----------- | ------|
| A | Pull latest Victoriametrics image from Docker Hub | |

## 02.A Install Prometheus stand-alone on Docker Desktop
---

```
Swarajits-MacBook-Air:lib swarajitroy$ docker pull prom/prometheus
Using default tag: latest
latest: Pulling from prom/prometheus
Digest: sha256:60190123eb28250f9e013df55b7d58e04e476011911219f5cedac3c73a8b74e6
Status: Downloaded newer image for prom/prometheus:latest
docker.io/prom/prometheus:latest

Swarajits-MacBook-Air:lib swarajitroy$ docker images
REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
victoriametrics/victoria-metrics   latest              58ed7efae12a        2 days ago          24.1MB
prom/prometheus                    latest              7adf5a25557b        2 weeks ago         168MB

```

## 07. Install VictoriaMetrics stand-alone as a Pod into CNCF k8s cluster
---

| Task ID | Task Name | Remarks
| ----------- | ----------- | ------|
| A | Create a PV (Persistent Volume) intended for Victoriametrics Storage | |
| B | Create a PVC (Persistent Vomume Claim) for the Victoriametrics Storage |
| C | Create a k8s Pod manifest (YAML) for VictoriaMetrics attached with volume |
| D | Test Victoriametric own metric endpoint URL |

### 07.A Create a PV (Persistent Volume) intended for Victoriametrics Storage

```
{172.31.17.227}[Ansible Node 1][NFS Sever][K8S Master] ~/swararoy_k8s/victoriametrics >cat vicmet_pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-vicmet
spec:
  capacity:
    storage: 100Mi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /mnt/nfs_share
    server: 172.31.17.227

{172.31.17.227}[Ansible Node 1][NFS Sever][K8S Master] ~/swararoy_k8s/victoriametrics >kubectl apply -f vicmet_pv.yaml
persistentvolume/nfs-pv-vicmet created

{172.31.17.227}[Ansible Node 1][NFS Sever][K8S Master] ~/swararoy_k8s/victoriametrics >kubectl get pv
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM             STORAGECLASS   REASON   AGE
nfs-pv          100Mi      RWX            Recycle          Bound       default/nfs-pvc   nfs                     3d18h
nfs-pv-vicmet   100Mi      RWX            Recycle          Available                     nfs                     95s

{172.31.17.227}[Ansible Node 1][NFS Sever][K8S Master] ~/swararoy_k8s/victoriametrics >kubectl describe pv nfs-pv-vicmet
Name:            nfs-pv-vicmet
Labels:          <none>
Annotations:     <none>
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    nfs
Status:          Available
Claim:
Reclaim Policy:  Recycle
Access Modes:    RWX
VolumeMode:      Filesystem
Capacity:        100Mi
Node Affinity:   <none>
Message:
Source:
    Type:      NFS (an NFS mount that lasts the lifetime of a pod)
    Server:    172.31.17.227
    Path:      /mnt/nfs_share
    ReadOnly:  false
Events:        <none>

```

### 07.B Create a PVC (Persistent Vomume Claim) for the Victoriametrics Storage

```
{172.31.17.227}[Ansible Node 1][NFS Sever][K8S Master] ~/swararoy_k8s/victoriametrics >cat vicmet_pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-vicmet
spec:
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Mi
      

{172.31.17.227}[Ansible Node 1][NFS Sever][K8S Master] ~/swararoy_k8s/victoriametrics >kubectl apply -f vicmet_pvc.yaml
persistentvolumeclaim/nfs-pvc-vicmet created

{172.31.17.227}[Ansible Node 1][NFS Sever][K8S Master] ~/swararoy_k8s/victoriametrics >kubectl get pvc
NAME             STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-pvc          Bound    nfs-pv          100Mi      RWX            nfs            3d18h
nfs-pvc-vicmet   Bound    nfs-pv-vicmet   100Mi      RWX            nfs            25s

{172.31.17.227}[Ansible Node 1][NFS Sever][K8S Master] ~/swararoy_k8s/victoriametrics >kubectl describe pvc nfs-pvc-vicmet
Name:          nfs-pvc-vicmet
Namespace:     default
StorageClass:  nfs
Status:        Bound
Volume:        nfs-pv-vicmet
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      100Mi
Access Modes:  RWX
VolumeMode:    Filesystem
Mounted By:    <none>
Events:        <none>


```
### 07.C Create a k8s Pod manifest (YAML) for VictoriaMetrics attached with volume

```
{172.31.17.227}[Ansible Node 1][NFS Sever][K8S Master] ~/swararoy_k8s/victoriametrics >cat vicmet_pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: vicmet-pv-pod
spec:
  volumes:
    - name: vicmet-pv-storage
      persistentVolumeClaim:
        claimName: nfs-pvc-vicmet
  containers:
    - name:  vicmet-server
      image: victoriametrics/victoria-metrics
      ports:
        - containerPort: 8428
          name: "vicmet-server"
      volumeMounts:
        - mountPath: "/victoria-metrics-data"
          name: vicmet-pv-storage
          
{172.31.17.227}[Ansible Node 1][NFS Sever][K8S Master] ~/swararoy_k8s/victoriametrics >kubectl apply -f vicmet_pod.yaml
pod/vicmet-pv-pod created

{172.31.17.227}[Ansible Node 1][NFS Sever][K8S Master] ~/swararoy_k8s/victoriametrics >kubectl get pods -o wide
NAME            READY   STATUS    RESTARTS   AGE   IP                NODE                           NOMINATED NODE   READINESS GATES
vicmet-pv-pod   1/1     Running   0          3m    192.168.193.130   cd4a104e2c1c.mylabserver.com   <none>           <none>

{172.31.17.227}[Ansible Node 1][NFS Sever][K8S Master] ~ >kubectl describe pod vicmet-pv-pod
Name:         vicmet-pv-pod
Namespace:    default
Priority:     0
Node:         cd4a104e2c1c.mylabserver.com/172.31.21.240
Start Time:   Wed, 04 Nov 2020 04:18:05 +0000
Labels:       <none>
Annotations:  cni.projectcalico.org/podIP: 192.168.193.130/32
              cni.projectcalico.org/podIPs: 192.168.193.130/32
Status:       Running
IP:           192.168.193.130
IPs:
  IP:  192.168.193.130
Containers:
  vicmet-server:
    Container ID:   docker://6ca135c8e7568730e960208c40d8750c91fbd484ddd668111dfde44474f29389
    Image:          victoriametrics/victoria-metrics
    Image ID:       docker-pullable://victoriametrics/victoria-metrics@sha256:02e04c263ab4ebe5311d9675ac15e588878a0211aceb052d911b4fe5f4a4cb6b
    Port:           8428/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 04 Nov 2020 04:18:10 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-pjcmk (ro)
      /victoria-metrics-data from vicmet-pv-storage (rw)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  vicmet-pv-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  nfs-pvc-vicmet
    ReadOnly:   false
  default-token-pjcmk:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-pjcmk
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:          <none>

{172.31.17.227}[Ansible Node 1][NFS Sever][K8S Master] ~/swararoy_k8s/victoriametrics >kubectl exec --stdin --tty vicmet-pv-pod -- /bin/sh
# ls /victoria-metrics-data
   data            flock.lock          indexdb         snapshots       


```

### 07.D Test Victoriametric own metric endpoint URL

```

{172.31.17.227}[Ansible Node 1][NFS Sever][K8S Master] ~/swararoy_k8s/victoriametrics >curl -v http://192.168.193.130:8428/metrics
*   Trying 192.168.193.130...
* TCP_NODELAY set
* Connected to 192.168.193.130 (192.168.193.130) port 8428 (#0)
> GET /metrics HTTP/1.1
> Host: 192.168.193.130:8428
> User-Agent: curl/7.58.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: text/plain
< Date: Wed, 04 Nov 2020 04:27:38 GMT
< Transfer-Encoding: chunked
<
vm_active_force_merges 0
vm_active_merges{type="indexdb"} 0
vm_active_merges{type="storage/big"} 0


```
