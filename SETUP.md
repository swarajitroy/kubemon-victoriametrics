VictoriaMetric App
---

## Table of Content (ToC)
---

* [00. Setup Tasks](#00-setup-tasks)
* [01. Run VictoriaMetric as Statefulset](#01-run-victoriametric-as-a-statefulset)
* [02. Run Prometheus Instance A Statefulset](#02-run-prometheus-instance-a-as-a-statefulset)
* [03. Run Prometheus Instance B Statefulset](#03-run-prometheus-instance-b-as-a-statefulset)
* [04. Run Grafana Statefulset](#04-run-grafana-as-a-statefulset)
* [05. Appmetric application - exposition](#05-appmetric-app---a-prometheus-exposition)
* [06. Run Appmetric Instance 01](#06-run-appmetric-endpoint-instance-01-as-a-deployment)
* [07. Run Appmetric Instance 02](#07-run-appmetric-endpoint-instance-02-as-a-deployment)
* [08. Configure Prometheus Instance 01 to scrape AppMetric endpoint Instance 01](#08-configure-prometheus-instance-01-to-scrape-appmetric-endpoint-instance-01)
* [09. Configure Prometheus Instance 02 to scrape AppMetric endpoint Instance 02](#09-configure-prometheus-instance-02-to-scrape-appmetric-endpoint-instance-02)
* [10. Configure Prometheus Instance 01 to remote write](#10-configure-prometheus-instance-01-to-remote-write-to-victoriametric)
* [11. Configure Prometheus Instance 02 to remote write](#11-configure-prometheus-instance-01-to-remote-write-to-victoriametric)
* [12. End to End Demo](#12-end-to-end-demo-using-grafana--promql)

* [13. VictroriaMetric Additional](#13-victoriametric-additional)
  * [13.1 Security - Authentication](#131-victoriametric-security---authentication)
  * [13.2 Security - Encryption](#132-victoriametric-security---encryption)
  * [13.3 Security - Auth Proxy](#133-victoriametric-security---vmauth)
  * [13.4 Helm Chart](#134-helm-chart)
  * [13.5 Kubernetes Operator](#135-kubernetes-operator)
  * [13.6 ArgoCD GitOps](#136-argocd)
  * [13.7 Monitoring](#137-monitoring)
  * [13.8 Backup](#138-backup)
  * [13.9 Alerting](#139-alerting)



## 00. Setup Tasks
---

| ID | Task | Remarks
| ----------- | ----------- | ------ |
| A | The dnstools image  |  infoblox/dnstools|
| B | Minikube with Docker Registry   |  |

### 00.A The dnstools image
---

This setup uses a publicly available image named infoblox/dnstools and it has utilities like ping, host, dig, dnsperf, curl, dnsdump. 


```
Swarajits-MacBook-Air:~ swarajitroy$ kubectl run --restart=Never --rm -it --image infoblox/dnstools swararoy-dnstools
If you don't see a command prompt, try pressing enter.
dnstools# exit
pod "swararoy-dnstools" deleted
```

### 00.B Minikube with Docker Registry 
---

GCR/ECR/ACR/Docker: minikube has an addon, registry-creds which maps credentials into minikube to support pulling from Google Container Registry (GCR), Amazon’s EC2 Container Registry (ECR), Azure Container Registry (ACR), and Private Docker registries. You will need to run minikube addons configure registry-creds and minikube addons enable registry-creds to get
up and running.

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ minikube addons configure registry-creds

Do you want to enable AWS Elastic Container Registry? [y/n]: n

Do you want to enable Google Container Registry? [y/n]: n

Do you want to enable Docker Registry? [y/n]: y
-- Enter docker registry server url: https://hub.docker.com/
-- Enter docker registry username: swarajitory
-- Enter docker registry password:

Do you want to enable Azure Container Registry? [y/n]: n
✅  registry-creds was successfully configured

Swarajits-MacBook-Air:victoriametrics swarajitroy$ minikube addons enable registry-creds
🌟  The 'registry-creds' addon is enabled

```

[Back to TOC](#table-of-content-toc)


## 01. Run VictoriaMetric as a StatefulSet
---

| ID | Task | Remarks
| ----------- | ----------- | ------ |
| A | Mount a local directory to Minikube  | |
| B | Create Persistence Volume  | local |
| C | Create Persistent Volume Claim |local | 
| D | Create Statefulset for VictoriaMetric | 
| E | Test VictoriaMetric Endpoints within Pods and outside |


### 01.A Mount a local directory to Minikube
---

```
minikube mount  /Users/swarajitroy/K8S_VOLUMES:/swarajitroy-data
Swarajits-MacBook-Air:vicmet-vol swarajitroy$ docker ps
CONTAINER ID        IMAGE                                 COMMAND                  CREATED             STATUS              PORTS                                                                                                      NAMES
b0d2e482a0c4        gcr.io/k8s-minikube/kicbase:v0.0.13   "/usr/local/bin/entr…"   35 hours ago        Up 2 hours          127.0.0.1:32775->22/tcp, 127.0.0.1:32774->2376/tcp, 127.0.0.1:32773->5000/tcp, 127.0.0.1:32772->8443/tcp   minikube

Swarajits-MacBook-Air:vicmet-vol swarajitroy$ docker exec -it b0d2e482a0c4 /bin/sh
# ls /
Release.key  afbjorklund-public.key.asc  boot  dev  home     kind  lib32  libx32  mnt  proc  run   srv		     sys  usr
Users	     bin			 data  etc  kic.txt  lib   lib64  media   opt  root  sbin  swarajitroy-data  tmp  var
# ls /swarajitroy-data

```

### 01.B Create Persistence Volume
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat swararoy-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: swararoy-pv-volume
spec:
  capacity:
    storage: 250Mi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /swararoy-data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - minikube

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f swararoy-pv.yaml
persistentvolume/swararoy-pv-volume created
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get pv
NAME                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS    REASON   AGE
swararoy-pv-volume   250Mi      RWO            Delete           Available           local-storage            7s
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe pv swararoy-pv-volume
Name:              swararoy-pv-volume
Labels:            <none>
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"PersistentVolume","metadata":{"annotations":{},"name":"swararoy-pv-volume"},"spec":{"accessModes":["ReadWriteOn...
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      local-storage
Status:            Available
Claim:
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          250Mi
Node Affinity:
  Required Terms:
    Term 0:        kubernetes.io/hostname in [minikube]
Message:
Source:
    Type:  LocalVolume (a persistent volume backed by local storage on a node)
    Path:  /swararoy-data
Events:    <none>

```

### 01.C Create Persistence Volume claim
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat swararoy-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: swararoy-pv-volume-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 250Mi
  storageClassName: local-storage

Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat swararoy-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: swararoy-pv-volume-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 250Mi
  storageClassName: local-storage


Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f swararoy-pvc.yaml
persistentvolumeclaim/swararoy-pv-volume-claim created

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get pvc
NAME                       STATUS   VOLUME               CAPACITY   ACCESS MODES   STORAGECLASS    AGE
swararoy-pv-volume-claim   Bound    swararoy-pv-volume   250Mi      RWO            local-storage   7s
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe pv swararoy-pv-volume-claim
Error from server (NotFound): persistentvolumes "swararoy-pv-volume-claim" not found
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe pvc swararoy-pv-volume-claim
Name:          swararoy-pv-volume-claim
Namespace:     default
StorageClass:  local-storage
Status:        Bound
Volume:        swararoy-pv-volume
Labels:        <none>
Annotations:   kubectl.kubernetes.io/last-applied-configuration:
                 {"apiVersion":"v1","kind":"PersistentVolumeClaim","metadata":{"annotations":{},"name":"swararoy-pv-volume-claim","namespace":"default"},"s...
               pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      250Mi
Access Modes:  RWO
VolumeMode:    Filesystem
Mounted By:    <none>
Events:        <none>
  
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get pv
NAME                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                              STORAGECLASS    REASON   AGE
swararoy-pv-volume   250Mi      RWO            Delete           Bound    default/swararoy-pv-volume-claim   local-storage            4m38s
```

### 01.D Create Statefulset for VictoriaMetric
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat victoriametric-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: victoria-metrics-headless-service
  labels:
    app: victoria-metrics
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: victoria-metrics

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: victoria-metrics
spec:
  serviceName: "victoria-metrics-service"
  replicas: 1
  selector:
    matchLabels:
      app: victoria-metrics
  template:
    metadata:
      labels:
        app: victoria-metrics
    spec:
      containers:
      - name: victoria-metrics
        image: victoriametrics/victoria-metrics
        ports:
        - containerPort: 8428
          name: server-port
        volumeMounts:
        - name: victoriametric-local-pv
          mountPath: /victoria-metrics-data
      volumes:
      - name: victoriametric-local-pv
        persistentVolumeClaim:
          claimName: swararoy-pv-volume-claim

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f victoriametric-statefulset.yaml
service/victoria-metrics-headless-service unchanged
statefulset.apps/victoria-metrics created

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get svc
NAME                                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes                          ClusterIP   10.96.0.1    <none>        443/TCP   35h
victoria-metrics-headless-service   ClusterIP   None         <none>        80/TCP    4h54m
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe svc victoria-metrics-headless-service
Name:              victoria-metrics-headless-service
Namespace:         default
Labels:            app=victoria-metrics
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"victoria-metrics"},"name":"victoria-metrics-headless-ser...
Selector:          app=victoria-metrics
Type:              ClusterIP
IP:                None
Port:              web  80/TCP
TargetPort:        80/TCP
Endpoints:         <none>
Session Affinity:  None
Events:            <none>

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get statefulsets
NAME               READY   AGE
victoria-metrics   1/1     16s
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe statefulset victoria-metrics
Name:               victoria-metrics
Namespace:          default
CreationTimestamp:  Sun, 08 Nov 2020 23:12:00 +0530
Selector:           app=victoria-metrics
Labels:             <none>
Annotations:        kubectl.kubernetes.io/last-applied-configuration:
                      {"apiVersion":"apps/v1","kind":"StatefulSet","metadata":{"annotations":{},"name":"victoria-metrics","namespace":"default"},"spec":{"replic...
Replicas:           1 desired | 1 total
Update Strategy:    RollingUpdate
  Partition:        824640842312
Pods Status:        1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=victoria-metrics
  Containers:
   victoria-metrics:
    Image:        victoriametrics/victoria-metrics
    Port:         8428/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:
      /victoria-metrics-data from victoriametric-local-pv (rw)
  Volumes:
   victoriametric-local-pv:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  swararoy-pv-volume-claim
    ReadOnly:   false
Volume Claims:  <none>
Events:
  Type    Reason            Age   From                    Message
  ----    ------            ----  ----                    -------
  Normal  SuccessfulCreate  39s   statefulset-controller  create Pod victoria-metrics-0 in StatefulSet victoria-metrics successful
  
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe pod  victoria-metrics-0
Name:         victoria-metrics-0
Namespace:    default
Priority:     0
Node:         minikube/192.168.49.2
Start Time:   Sun, 08 Nov 2020 23:12:00 +0530
Labels:       app=victoria-metrics
              controller-revision-hash=victoria-metrics-5b456dcbb9
              statefulset.kubernetes.io/pod-name=victoria-metrics-0
Annotations:  <none>
Status:       Running
IP:           172.17.0.5
IPs:
  IP:           172.17.0.5
Controlled By:  StatefulSet/victoria-metrics
Containers:
  victoria-metrics:
    Container ID:   docker://5d0c9b353ab7b9a5f499d1f83e652becc60edb6fe38e50899c2a311e4a55012f
    Image:          victoriametrics/victoria-metrics
    Image ID:       docker-pullable://victoriametrics/victoria-metrics@sha256:8bb7e56b7792df71668186329b57df753d630698f5881dc6493ac014be37d9a1
    Port:           8428/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sun, 08 Nov 2020 23:12:08 +0530
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-8vhsg (ro)
      /victoria-metrics-data from victoriametric-local-pv (rw)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  victoriametric-local-pv:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  swararoy-pv-volume-claim
    ReadOnly:   false
  default-token-8vhsg:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-8vhsg
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>                     Successfully assigned default/victoria-metrics-0 to minikube
  Normal  Pulling    90s        kubelet, minikube  Pulling image "victoriametrics/victoria-metrics"
  Normal  Pulled     84s        kubelet, minikube  Successfully pulled image "victoriametrics/victoria-metrics" in 5.7686433s
  Normal  Created    84s        kubelet, minikube  Created container victoria-metrics
  Normal  Started    84s        kubelet, minikube  Started container victoria-metrics  
  
  

```

### 01.E Test VictoriaMetric Endpoints within Pods and outside
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl exec -it shell-demo /bin/sh

# curl http://victoria-metrics-headless-service.default.svc.cluster.local:8428/metrics
vm_active_force_merges 0
vm_active_merges{type="indexdb"} 0
vm_active_merges{type="storage/big"} 0
vm_active_merges{type="storage/small"} 0
vm_assisted_merges_total{type="indexdb"} 0
vm_assisted_merges_total{type="storage/small"} 0

```

[Back to TOC](#table-of-content-toc)

## 02. Run Prometheus Instance A as a StatefulSet
---

| ID | Task | Remarks
| ----------- | ----------- | ------ |
| A | Mount a local directory to Minikube  | |
| B | Create Persistence Volume  | local |
| C | Create Persistent Volume Claim |local | 
| D | Create ConfigMap holding prometheus confuration |local | 
| E | Create Statefulset for Prometheus Instance A | 
| F | Test Prometheus Instance A Endpoints within Pods and outside |

### 02.A Mount a local directory to Minikube
---

```
mkdir -p /Users/swarajitroy/K8S_PROM_1A_VOL
minikube mount  /Users/swarajitroy/K8S_PROM_1A_VOL:/swarajitroy-prom01-data

Swarajits-MacBook-Air:victoriametrics swarajitroy$ docker ps
CONTAINER ID        IMAGE                                 COMMAND                  CREATED             STATUS              PORTS                                                                                                      NAMES
b0d2e482a0c4        gcr.io/k8s-minikube/kicbase:v0.0.13   "/usr/local/bin/entr…"   2 days ago          Up 17 hours         127.0.0.1:32775->22/tcp, 127.0.0.1:32774->2376/tcp, 127.0.0.1:32773->5000/tcp, 127.0.0.1:32772->8443/tcp   minikube

Swarajits-MacBook-Air:victoriametrics swarajitroy$ docker ps
CONTAINER ID        IMAGE                                 COMMAND                  CREATED             STATUS              PORTS                                                                                                      NAMES
b0d2e482a0c4        gcr.io/k8s-minikube/kicbase:v0.0.13   "/usr/local/bin/entr…"   2 days ago          Up 17 hours         127.0.0.1:32775->22/tcp, 127.0.0.1:32774->2376/tcp, 127.0.0.1:32773->5000/tcp, 127.0.0.1:32772->8443/tcp   minikube
Swarajits-MacBook-Air:victoriametrics swarajitroy$ docker exec -it b0d2e482a0c4 /bin/sh
# ls /
Release.key  afbjorklund-public.key.asc  boot  dev  home     kind  lib32  libx32  mnt  proc  run   srv		     swarajitroy-prom01-data  tmp  var
Users	     bin			 data  etc  kic.txt  lib   lib64  media   opt  root  sbin  swarajitroy-data  sys		      usr
# ls /swarajitroy-prom01-data


```

### 02.B Create Persistence Volume
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f prometheus_01-pv.yaml
persistentvolume/prom01-pv-volume created
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get pv
NAME                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                              STORAGECLASS    REASON   AGE
prom01-pv-volume     150Mi      RWO            Delete           Available                                      local-storage            5s
swararoy-pv-volume   250Mi      RWO            Delete           Bound       default/swararoy-pv-volume-claim   local-storage            17h
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe pv prom01-pv-volume
Name:              prom01-pv-volume
Labels:            <none>
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"PersistentVolume","metadata":{"annotations":{},"name":"prom01-pv-volume"},"spec":{"accessModes":["ReadWriteOnce...
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      local-storage
Status:            Available
Claim:
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          150Mi
Node Affinity:
  Required Terms:
    Term 0:        kubernetes.io/hostname in [minikube]
Message:
Source:
    Type:  LocalVolume (a persistent volume backed by local storage on a node)
    Path:  /swarajitroy-prom01-data
Events:    <none>

```

### 02.C Create Persistence Volume Claim
---

```

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f prometheus_01-pvc.yaml
persistentvolumeclaim/prom01-pv-volume-claim created
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get pvc
NAME                       STATUS   VOLUME               CAPACITY   ACCESS MODES   STORAGECLASS    AGE
prom01-pv-volume-claim     Bound    prom01-pv-volume     150Mi      RWO            local-storage   8s
swararoy-pv-volume-claim   Bound    swararoy-pv-volume   250Mi      RWO            local-storage   17h
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe pvc prom01-pv-volume-claim
Name:          prom01-pv-volume-claim
Namespace:     default
StorageClass:  local-storage
Status:        Bound
Volume:        prom01-pv-volume
Labels:        <none>
Annotations:   kubectl.kubernetes.io/last-applied-configuration:
                 {"apiVersion":"v1","kind":"PersistentVolumeClaim","metadata":{"annotations":{},"name":"prom01-pv-volume-claim","namespace":"default"},"spe...
               pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      150Mi
Access Modes:  RWO
VolumeMode:    Filesystem
Mounted By:    <none>
Events:        <none>

```

### 02.D Create ConfigMap holding prometheus configuration
---

```

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f prometheus_01-configMap.yaml
configmap/prometheus-01-server-configmap created
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get configmaps
NAME                             DATA   AGE
prometheus-01-server-configmap   1      8s
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe configmap prometheus-01-server-configmap
Name:         prometheus-01-server-configmap
Namespace:    default
Labels:       name=prometheus-01-server-configmap
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"v1","data":{"prometheus.yml":"global:\n   scrape_interval:     15s \n   evaluation_interval: 15s \n   external_labels:\n   ...

Data
====
prometheus.yml:
----
global:
   scrape_interval:     15s
   evaluation_interval: 15s
   external_labels:
       openshift_instance: SDI_CAAS_QA
       datacenter: DC-01

alerting:
   alertmanagers:
     - static_configs:
        - targets:
          # - alertmanager:9093


scrape_configs:

- job_name: 'prometheus'
  static_configs:
    - targets: ['localhost:9090']
Events:  <none>
```
### 02.E Create Statefulset for Prometheus Instance A
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat prometheus_01-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: prom-01-service
  labels:
    app: prometheus-01
spec:
  ports:
  - port: 9090
    name: http
  clusterIP: None
  selector:
    app: prometheus-01
---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prom-01-statefulset
spec:
  serviceName: "prom-01-statefulset"
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-01
  template:
    metadata:
      labels:
        app: prometheus-01
    spec:

      initContainers:
      - name: "init-chown-data"
        image: "busybox:latest"
        imagePullPolicy: "IfNotPresent"
        command: ["chown", "-R", "65534:65534", "/swarajitroy-prom01-data"]
        volumeMounts:
        - name: prometheus-01-local-pv
          mountPath: /swarajitroy-prom01-data
          subPath: ""

      containers:
      - name:  prometheus-01
        image: prom/prometheus
        args: ["--config.file=/etc/config/prometheus.yml", "--storage.tsdb.path=/swarajitroy-prom01-data" , "--web.console.libraries=/etc/prometheus/console_libraries" , "--web.console.templates=/etc/prometheus/consoles", "--web.enable-lifecycle"]
        ports:
        - containerPort: 9090
          name: server-port
        volumeMounts:
        - name: prometheus-01-local-pv
          mountPath: /swarajitroy-prom01-data
        - name: prometheus-01-config-volume
          mountPath: /etc/config
      volumes:
      - name: prometheus-01-local-pv
        persistentVolumeClaim:
          claimName: prom01-pv-volume-claim
      - name: prometheus-01-config-volume
        configMap:
          name: prometheus-01-server-configmap

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get svc
NAME                                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
kubernetes                          ClusterIP   10.96.0.1    <none>        443/TCP    2d9h
prom-01-service                     ClusterIP   None         <none>        9090/TCP   69m
victoria-metrics-headless-service   ClusterIP   None         <none>        80/TCP     26h

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get statefulsets
NAME                  READY   AGE
prom-01-statefulset   1/1     122m
victoria-metrics      1/1     21h

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe statefulset prom-01-statefulset
Name:               prom-01-statefulset
Namespace:          default
CreationTimestamp:  Mon, 09 Nov 2020 18:46:51 +0530
Selector:           app=prometheus-01
Labels:             <none>
Annotations:        kubectl.kubernetes.io/last-applied-configuration:
                      {"apiVersion":"apps/v1","kind":"StatefulSet","metadata":{"annotations":{},"name":"prom-01-statefulset","namespace":"default"},"spec":{"rep...
Replicas:           1 desired | 1 total
Update Strategy:    RollingUpdate
  Partition:        824634446872
Pods Status:        1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=prometheus-01
  Init Containers:
   init-chown-data:
    Image:      busybox:latest
    Port:       <none>
    Host Port:  <none>
    Command:
      chown
      -R
      65534:65534
      /swarajitroy-prom01-data
    Environment:  <none>
    Mounts:
      /swarajitroy-prom01-data from prometheus-01-local-pv (rw)
  Containers:
   prometheus-01:
    Image:      prom/prometheus
    Port:       9090/TCP
    Host Port:  0/TCP
    Args:
      --config.file=/etc/config/prometheus.yml
      --storage.tsdb.path=/swarajitroy-prom01-data
      --web.console.libraries=/etc/prometheus/console_libraries
      --web.console.templates=/etc/prometheus/consoles
      --web.enable-lifecycle
    Environment:  <none>
    Mounts:
      /etc/config from prometheus-01-config-volume (rw)
      /swarajitroy-prom01-data from prometheus-01-local-pv (rw)
  Volumes:
   prometheus-01-local-pv:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  prom01-pv-volume-claim
    ReadOnly:   false
   prometheus-01-config-volume:
    Type:       ConfigMap (a volume populated by a ConfigMap)
    Name:       prometheus-01-server-configmap
    Optional:   false
Volume Claims:  <none>
Events:         <none>



```

### 02.F Test Prometheus Instance A Endpoints within Pods and outside

```
# ping prom-01-service.default.svc.cluster.local
PING prom-01-service.default.svc.cluster.local (172.17.0.7) 56(84) bytes of data.
64 bytes from 172-17-0-7.prom-01-service.default.svc.cluster.local (172.17.0.7): icmp_seq=1 ttl=64 time=0.221 ms
64 bytes from 172-17-0-7.prom-01-service.default.svc.cluster.local (172.17.0.7): icmp_seq=2 ttl=64 time=0.520 ms

# curl http://prom-01-service.default.svc.cluster.local:9090/metrics | more
 
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 8.14e-05
go_gc_duration_seconds{quantile="0.25"} 0.0002467
go_gc_duration_seconds{quantile="0.5"} 0.0004072
go_gc_duration_seconds{quantile="0.75"} 0.0006224
go_gc_duration_seconds{quantile="1"} 0.0028588
go_gc_duration_seconds_sum 0.0338667


```
[Back to TOC](#table-of-content-toc)

## 03. Run Prometheus Instance B as a StatefulSet
---

| ID | Task | Remarks
| ----------- | ----------- | ------ |
| A | Mount a local directory to Minikube  | |
| B | Create Persistence Volume  | local |
| C | Create Persistent Volume Claim |local | 
| D | Create ConfigMap holding prometheus configuration |local | 
| E | Create Statefulset for Prometheus Instance B | 
| F | Test Prometheus Instance B Endpoints within Pods and outside |

### 03.A Mount a local directory to Minikube
---

```
Swarajits-MacBook-Air:~ swarajitroy$ mkdir -p /Users/swarajitroy/K8S_PROM_1B_VOL
minikube mount  /Users/swarajitroy/K8S_PROM_1B_VOL:/swarajitroy-prom02-data

Swarajits-MacBook-Air:victoriametrics swarajitroy$ docker ps
CONTAINER ID        IMAGE                                 COMMAND                  CREATED             STATUS              PORTS                                                                                                      NAMES
b0d2e482a0c4        gcr.io/k8s-minikube/kicbase:v0.0.13   "/usr/local/bin/entr…"   2 days ago          Up 17 hours         127.0.0.1:32775->22/tcp, 127.0.0.1:32774->2376/tcp, 127.0.0.1:32773->5000/tcp, 127.0.0.1:32772->8443/tcp   minikube

Swarajits-MacBook-Air:victoriametrics swarajitroy$ docker ps
CONTAINER ID        IMAGE                                 COMMAND                  CREATED             STATUS              PORTS                                                                                                      NAMES
b0d2e482a0c4        gcr.io/k8s-minikube/kicbase:v0.0.13   "/usr/local/bin/entr…"   2 days ago          Up 17 hours         127.0.0.1:32775->22/tcp, 127.0.0.1:32774->2376/tcp, 127.0.0.1:32773->5000/tcp, 127.0.0.1:32772->8443/tcp   minikube
Swarajits-MacBook-Air:victoriametrics swarajitroy$ docker exec -it b0d2e482a0c4 /bin/sh
# ls /
# ls /
Release.key  afbjorklund-public.key.asc  boot  dev  home     kind  lib32  libx32  mnt  proc  run   srv		     swarajitroy-grafana-data  swarajitroy-prom02-data	tmp  var
Users	     bin			 data  etc  kic.txt  lib   lib64  media   opt  root  sbin  swarajitroy-data  swarajitroy-prom01-data   sys			usr
#
# ls /swarajitroy-prom02-data


```

### 03.B Create Persistence Volume
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat prometheus_02-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prom02-pv-volume
spec:
  capacity:
    storage: 150Mi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /swarajitroy-prom02-data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - minikube

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f prometheus_02-pv.yaml
persistentvolume/prom02-pv-volume created

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get pv
NAME                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                              STORAGECLASS    REASON   AGE
grafana-pv-volume    150Mi      RWO            Delete           Bound       default/grafana-pv-volume-claim    local-storage            2d
prom01-pv-volume     150Mi      RWO            Delete           Bound       default/prom01-pv-volume-claim     local-storage            2d6h
prom02-pv-volume     150Mi      RWO            Delete           Available                                      local-storage            8s
swararoy-pv-volume   250Mi      RWO            Delete           Bound       default/swararoy-pv-volume-claim   local-storage            3d

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe pv prom02-pv-volume
Name:              prom02-pv-volume
Labels:            <none>
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"PersistentVolume","metadata":{"annotations":{},"name":"prom02-pv-volume"},"spec":{"accessModes":["ReadWriteOnce...
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      local-storage
Status:            Available
Claim:
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          150Mi
Node Affinity:
  Required Terms:
    Term 0:        kubernetes.io/hostname in [minikube]
Message:
Source:
    Type:  LocalVolume (a persistent volume backed by local storage on a node)
    Path:  /swarajitroy-prom02-data
Events:    <none>

```

### 03.C Create Persistence Volume Claim
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat prometheus_02-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prom02-pv-volume-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 150Mi
  storageClassName: local-storage

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f prometheus_02-pvc.yaml
persistentvolumeclaim/prom02-pv-volume-claim created

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get pvc
NAME                       STATUS   VOLUME               CAPACITY   ACCESS MODES   STORAGECLASS    AGE
grafana-pv-volume-claim    Bound    grafana-pv-volume    150Mi      RWO            local-storage   14h
prom01-pv-volume-claim     Bound    prom01-pv-volume     150Mi      RWO            local-storage   2d6h
prom02-pv-volume-claim     Bound    prom02-pv-volume     150Mi      RWO            local-storage   5s
swararoy-pv-volume-claim   Bound    swararoy-pv-volume   250Mi      RWO            local-storage   3d

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe pvc prom02-pv-volume-claim
Name:          prom02-pv-volume-claim
Namespace:     default
StorageClass:  local-storage
Status:        Bound
Volume:        prom02-pv-volume
Labels:        <none>
Annotations:   kubectl.kubernetes.io/last-applied-configuration:
                 {"apiVersion":"v1","kind":"PersistentVolumeClaim","metadata":{"annotations":{},"name":"prom02-pv-volume-claim","namespace":"default"},"spe...
               pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      150Mi
Access Modes:  RWO
VolumeMode:    Filesystem
Mounted By:    <none>
Events:        <none>

```

### 03.D Create ConfigMap holding prometheus configuration
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat prometheus_02-configMap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-02-server-configmap
  labels:
    name: prometheus-02-server-configmap
data:
  prometheus.yml: |-
    global:
       scrape_interval:     15s
       evaluation_interval: 15s
       external_labels:
           openshift_instance: SDI_CAAS_PROD
           datacenter: DC-02

    alerting:
       alertmanagers:
         - static_configs:
            - targets:
              # - alertmanager:9093


    scrape_configs:

    - job_name: 'prometheus'
      static_configs:
        - targets: ['localhost:9090']
        
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f prometheus_02-configMap.yaml
configmap/prometheus-02-server-configmap created
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get configMaps
NAME                                  DATA   AGE
prometheus-01-server-configmap        1      3d4h
prometheus-02-server-configmap        1      7s
swararoy-grafana-dashboardproviders   1      37h
swararoy-grafana-datasources          1      24h
swararoy-grafana-ini                  1      37h

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe configMap prometheus-02-server-configmap
Name:         prometheus-02-server-configmap
Namespace:    default
Labels:       name=prometheus-02-server-configmap
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"v1","data":{"prometheus.yml":"global:\n   scrape_interval:     15s \n   evaluation_interval: 15s \n   external_labels:\n   ...

Data
====
prometheus.yml:
----
global:
   scrape_interval:     15s
   evaluation_interval: 15s
   external_labels:
       openshift_instance: SDI_CAAS_PROD
       datacenter: DC-02

alerting:
   alertmanagers:
     - static_configs:
        - targets:
          # - alertmanager:9093


scrape_configs:

- job_name: 'prometheus'
  static_configs:
    - targets: ['localhost:9090']
Events:  <none>

```

### 03.E Create Statefulset for Prometheus Instance B
---

```
apiVersion: v1
kind: Service
metadata:
  name: prom-02-service
  labels:
    app: prometheus-02
spec:
  ports:
  - port: 9090
    name: http
  clusterIP: None
  selector:
    app: prometheus-02
---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prom-02-statefulset
spec:
  serviceName: "prom-02-statefulset"
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-02
  template:
    metadata:
      labels:
        app: prometheus-02
    spec:

      initContainers:
      - name: "init-chown-data"
        image: "busybox:latest"
        imagePullPolicy: "IfNotPresent"
        command: ["chown", "-R", "65534:65534", "/swarajitroy-prom02-data"]
        volumeMounts:
        - name: prometheus-02-local-pv
          mountPath: /swarajitroy-prom02-data
          subPath: ""

      containers:
      - name:  prometheus-02
        image: prom/prometheus
        args: ["--config.file=/etc/config/prometheus.yml", "--storage.tsdb.path=/swarajitroy-prom02-data" , "--web.console.libraries=/etc/prometheus/console_libraries" , "--web.console.templates=/etc/prometheus/consoles", "--web.enable-lifecycle"]
        ports:
        - containerPort: 9090
          name: server-port
        volumeMounts:
        - name: prometheus-02-local-pv
          mountPath: /swarajitroy-prom02-data
        - name: prometheus-02-config-volume
          mountPath: /etc/config
      volumes:
      - name: prometheus-02-local-pv
        persistentVolumeClaim:
          claimName: prom02-pv-volume-claim
      - name: prometheus-02-config-volume
        configMap:
          name: prometheus-02-server-configmap


Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f prometheus_02-statefulset.yaml
service/prom-02-service created
statefulset.apps/prom-02-statefulset created

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get svc
NAME                                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
grafana-service                     ClusterIP   None         <none>        8080/TCP   29h
kubernetes                          ClusterIP   10.96.0.1    <none>        443/TCP    5d12h
prom-01-service                     ClusterIP   None         <none>        9090/TCP   3d4h
prom-02-service                     ClusterIP   None         <none>        9090/TCP   7m39s
victoria-metrics-headless-service   ClusterIP   None         <none>        80/TCP     4d5h
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe svc prom-02-service
Name:              prom-02-service
Namespace:         default
Labels:            app=prometheus-02
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"prometheus-02"},"name":"prom-02-service","namespace":"de...
Selector:          app=prometheus-02
Type:              ClusterIP
IP:                None
Port:              http  9090/TCP
TargetPort:        9090/TCP
Endpoints:         172.17.0.9:9090
Session Affinity:  None
Events:            <none>

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get statefulsets -o wide
NAME                     READY   AGE     CONTAINERS         IMAGES
grafana-01-statefulset   1/1     25h     grafana-01         grafana/grafana
prom-01-statefulset      1/1     3d4h    prometheus-01      prom/prometheus
prom-02-statefulset      1/1     8m56s   prometheus-02      prom/prometheus
victoria-metrics         1/1     4d      victoria-metrics   victoriametrics/victoria-metrics

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe statefulset prom-02-statefulset
Name:               prom-02-statefulset
Namespace:          default
CreationTimestamp:  Thu, 12 Nov 2020 23:36:13 +0530
Selector:           app=prometheus-02
Labels:             <none>
Annotations:        kubectl.kubernetes.io/last-applied-configuration:
                      {"apiVersion":"apps/v1","kind":"StatefulSet","metadata":{"annotations":{},"name":"prom-02-statefulset","namespace":"default"},"spec":{"rep...
Replicas:           1 desired | 1 total
Update Strategy:    RollingUpdate
  Partition:        824643384696
Pods Status:        1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=prometheus-02
  Init Containers:
   init-chown-data:
    Image:      busybox:latest
    Port:       <none>
    Host Port:  <none>
    Command:
      chown
      -R
      65534:65534
      /swarajitroy-prom02-data
    Environment:  <none>
    Mounts:
      /swarajitroy-prom02-data from prometheus-02-local-pv (rw)
  Containers:
   prometheus-02:
    Image:      prom/prometheus
    Port:       9090/TCP
    Host Port:  0/TCP
    Args:
      --config.file=/etc/config/prometheus.yml
      --storage.tsdb.path=/swarajitroy-prom02-data
      --web.console.libraries=/etc/prometheus/console_libraries
      --web.console.templates=/etc/prometheus/consoles
      --web.enable-lifecycle
    Environment:  <none>
    Mounts:
      /etc/config from prometheus-02-config-volume (rw)
      /swarajitroy-prom02-data from prometheus-02-local-pv (rw)
  Volumes:
   prometheus-02-local-pv:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  prom02-pv-volume-claim
    ReadOnly:   false
   prometheus-02-config-volume:
    Type:       ConfigMap (a volume populated by a ConfigMap)
    Name:       prometheus-02-server-configmap
    Optional:   false
Volume Claims:  <none>
Events:
  Type    Reason            Age    From                    Message
  ----    ------            ----   ----                    -------
  Normal  SuccessfulCreate  9m18s  statefulset-controller  create Pod prom-02-statefulset-0 in StatefulSet prom-02-statefulset successful

```

### 03.F Test Prometheus Instance B Endpoints within Pods and outside

```
dnstools# ping prom-02-service.default.svc.cluster.local
PING prom-02-service.default.svc.cluster.local (172.17.0.9): 56 data bytes
64 bytes from 172.17.0.9: seq=0 ttl=64 time=1.571 ms
64 bytes from 172.17.0.9: seq=1 ttl=64 time=0.169 ms

dnstools# curl http://prom-02-service.default.svc.cluster.local:9090/metrics | more
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3944   # HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 7.11e-05
go_gc_duration_seconds{quantile="0.25"} 0.0002091
go_gc_duration_seconds{quantile="0.5"} 0.0004392
go_gc_duration_seconds{quantile="0.75"} 0.0018599
go_gc_duration_seconds{quantile="1"} 15.4702021
go_gc_duration_seconds_sum 17.8589584
go_gc_duration_seconds_count 172


```
[Back to TOC](#table-of-content-toc)

## 04. Run Grafana as a StatefulSet
---

| ID | Task | Remarks
| ----------- | ----------- | ------ |
| A | Mount a local directory to Minikube  | |
| B | Create Persistence Volume  | local |
| C | Create Persistent Volume Claim |local | 
| D | Create ConfigMaps holding Grafana configuration | | 
| E | Create Secrets holding Grafana configuration | | 
| F | Create Grafana Statefulset service | 
| G | Test Grafana within Pods and outside |

### 04.A Mount a local directory to Minikube 
---

```
mkdir -p /Users/swarajitroy/K8S_GRAFANA_VOL
minikube mount  /Users/swarajitroy/K8S_GRAFANA_VOL:/swarajitroy-grafana-data

Swarajits-MacBook-Air:victoriametrics swarajitroy$ docker ps
CONTAINER ID        IMAGE                                 COMMAND                  CREATED             STATUS              PORTS                                                                                                      NAMES
b0d2e482a0c4        gcr.io/k8s-minikube/kicbase:v0.0.13   "/usr/local/bin/entr…"   2 days ago          Up 26 hours         127.0.0.1:32775->22/tcp, 127.0.0.1:32774->2376/tcp, 127.0.0.1:32773->5000/tcp, 127.0.0.1:32772->8443/tcp   minikube

Swarajits-MacBook-Air:victoriametrics swarajitroy$ docker exec -it b0d2e482a0c4 /bin/sh
# ls /
Release.key  afbjorklund-public.key.asc  boot  dev  home     kind  lib32  libx32  mnt  proc  run   srv		     swarajitroy-grafana-data  sys  usr
Users	     bin			 data  etc  kic.txt  lib   lib64  media   opt  root  sbin  swarajitroy-data  swarajitroy-prom01-data   tmp  var
# ls swarajitroy-grafana-data

```
### 04.B Create Persistence Volume
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat grafana-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: grafana-pv-volume
spec:
  capacity:
    storage: 150Mi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /swarajitroy-grafana-data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - minikube
          
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f grafana-pv.yaml
persistentvolume/grafana-pv-volume created

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get pv
NAME                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                              STORAGECLASS    REASON   AGE
grafana-pv-volume    150Mi      RWO            Delete           Available                                      local-storage            50s
prom01-pv-volume     150Mi      RWO            Delete           Bound       default/prom01-pv-volume-claim     local-storage            5h55m
swararoy-pv-volume   250Mi      RWO            Delete           Bound       default/swararoy-pv-volume-claim   local-storage            23h

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe pv grafana-pv-volume
Name:              grafana-pv-volume
Labels:            <none>
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"PersistentVolume","metadata":{"annotations":{},"name":"grafana-pv-volume"},"spec":{"accessModes":["ReadWriteOnc...
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      local-storage
Status:            Available
Claim:
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          150Mi
Node Affinity:
  Required Terms:
    Term 0:        kubernetes.io/hostname in [minikube]
Message:
Source:
    Type:  LocalVolume (a persistent volume backed by local storage on a node)
    Path:  /swarajitroy-grafana-data
Events:    <none>

```

### 04.C Create Persistence Volume Claim
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat grafana-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pv-volume-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 150Mi
  storageClassName: local-storage

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f grafana-pvc.yaml
persistentvolumeclaim/grafana-pv-volume-claim created

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get pvc
NAME                       STATUS   VOLUME               CAPACITY   ACCESS MODES   STORAGECLASS    AGE
grafana-pv-volume-claim    Bound    grafana-pv-volume    150Mi      RWO            local-storage   6s
prom01-pv-volume-claim     Bound    prom01-pv-volume     150Mi      RWO            local-storage   40h
swararoy-pv-volume-claim   Bound    swararoy-pv-volume   250Mi      RWO            local-storage   2d10h

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe pvc grafana-pv-volume-claim
Name:          grafana-pv-volume-claim
Namespace:     default
StorageClass:  local-storage
Status:        Bound
Volume:        grafana-pv-volume
Labels:        <none>
Annotations:   kubectl.kubernetes.io/last-applied-configuration:
                 {"apiVersion":"v1","kind":"PersistentVolumeClaim","metadata":{"annotations":{},"name":"grafana-pv-volume-claim","namespace":"default"},"sp...
               pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      150Mi
Access Modes:  RWO
VolumeMode:    Filesystem
Mounted By:    <none>
Events:        <none>

```

### 04.D Create ConfigMaps holding Grafana configuration
---

Grafana has 5 aspects where some data volume is required, 

1. Grafana config (INI file)
2. Grafana data sources
3. Grafana dashboard providers
4. Grafana dashboards
5. Grafana Data

Out of these - the first 4 - we will drive through Kubernetes primitive - ConfigMap. The 5th one will be a classical data volume.

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat grafana-configMap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: swararoy-grafana-ini
  labels:
    app.kubernetes.io/name: swararoy-grafana
    app.kubernetes.io/component: grafana
data:
  grafana.ini: |
    [analytics]
    check_for_updates = true
    [grafana_net]
    url = https://grafana.net
    [log]
    mode = console
    [paths]
    data = /var/lib/grafana/data
    logs = /var/log/grafana
    plugins = /var/lib/grafana/plugins
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: swararoy-grafana-datasources
  labels:
    app.kubernetes.io/name: swararoy-grafana
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
    - access: proxy
      isDefault: true
      name: prometheus-01
      type: prometheus
      url: http://prom-01-service.default.svc.cluster.local:9090
      version: 1
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: swararoy-grafana-dashboardproviders
  labels:
    app.kubernetes.io/name: swararoy-grafana
data:
  dashboardproviders.yaml: |
    apiVersion: 1
    providers:
    - disableDeletion: false
      editable: true
      folder: ""
      name: default
      options:
        path: /var/lib/grafana/dashboards
      orgId: 1
      type: file

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f grafana-configMap.yaml
configmap/swararoy-grafana-ini created
configmap/swararoy-grafana-datasources created
configmap/swararoy-grafana-dashboardproviders created
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get configmaps
NAME                                  DATA   AGE
prometheus-01-server-configmap        1      39h
swararoy-grafana-dashboardproviders   1      8s
swararoy-grafana-datasources          1      8s
swararoy-grafana-ini                  1      8s
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe configmap swararoy-grafana-ini
Name:         swararoy-grafana-ini
Namespace:    default
Labels:       app.kubernetes.io/component=grafana
              app.kubernetes.io/name=swararoy-grafana
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"v1","data":{"grafana.ini":"[analytics]\ncheck_for_updates = true\n[grafana_net]\nurl = https://grafana.net\n[log]\nmode = c...

Data
====
grafana.ini:
----
[analytics]
check_for_updates = true
[grafana_net]
url = https://grafana.net
[log]
mode = console
[paths]
data = /var/lib/grafana/data
logs = /var/log/grafana
plugins = /var/lib/grafana/plugins

Events:  <none>

```

### 04.E Create Secrets holding Grafana configuration
---

### 04.F Create Grafana Statefulset service
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat grafana-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: grafana-01-statefulset
spec:
  serviceName: "grafana-01-statefulset"
  replicas: 1
  selector:
    matchLabels:
      app: grafana-01
  template:
    metadata:
      labels:
        app: grafana-01
    spec:
      containers:
       - name:  grafana-01
         image: grafana/grafana
         ports:
          - containerPort: 3000
         volumeMounts:
          - name: grafana-local-pv
            mountPath: /swarajitroy-grafana-data
          - name: config
            mountPath: "/etc/grafana/"
          - name: datasources
            mountPath: "/etc/grafana/provisioning/datasources/"
      volumes:
       - name: grafana-local-pv
         persistentVolumeClaim:
          claimName: grafana-pv-volume-claim

       - name: config
         configMap:
            name: swararoy-grafana-ini

       - name: datasources
         configMap:
            name: swararoy-grafana-datasources
            
  Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get svc
NAME                                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
grafana-service                     ClusterIP   None         <none>        8080/TCP   4h55m
kubernetes                          ClusterIP   10.96.0.1    <none>        443/TCP    4d11h
prom-01-service                     ClusterIP   None         <none>        9090/TCP   2d3h
victoria-metrics-headless-service   ClusterIP   None         <none>        80/TCP     3d4h
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe svc grafana-service
Name:              grafana-service
Namespace:         default
Labels:            app=grafana-01
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"grafana-01"},"name":"grafana-service","namespace":"defau...
Selector:          app=grafana-01
Type:              ClusterIP
IP:                None
Port:              http  8080/TCP
TargetPort:        3000/TCP
Endpoints:         172.17.0.8:3000
Session Affinity:  None
Events:            <none>
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get statefulsets
NAME                     READY   AGE
grafana-01-statefulset   1/1     23m
prom-01-statefulset      1/1     2d4h
victoria-metrics         1/1     2d23h
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe statefulsets grafana-01-statefulset
Name:               grafana-01-statefulset
Namespace:          default
CreationTimestamp:  Wed, 11 Nov 2020 22:26:12 +0530
Selector:           app=grafana-01
Labels:             <none>
Annotations:        kubectl.kubernetes.io/last-applied-configuration:
                      {"apiVersion":"apps/v1","kind":"StatefulSet","metadata":{"annotations":{},"name":"grafana-01-statefulset","namespace":"default"},"spec":{"...
Replicas:           1 desired | 1 total
Update Strategy:    RollingUpdate
  Partition:        824643022408
Pods Status:        1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=grafana-01
  Containers:
   grafana-01:
    Image:        grafana/grafana
    Port:         3000/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:
      /etc/grafana/ from config (rw)
      /etc/grafana/provisioning/datasources/ from datasources (rw)
      /swarajitroy-grafana-data from grafana-local-pv (rw)
  Volumes:
   grafana-local-pv:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  grafana-pv-volume-claim
    ReadOnly:   false
   config:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      swararoy-grafana-ini
    Optional:  false
   datasources:
    Type:       ConfigMap (a volume populated by a ConfigMap)
    Name:       swararoy-grafana-datasources
    Optional:   false
Volume Claims:  <none>
Events:
  Type    Reason            Age   From                    Message
  ----    ------            ----  ----                    -------
  Normal  SuccessfulCreate  24m   statefulset-controller  create Pod grafana-01-statefulset-0 in StatefulSet grafana-01-statefulset successful



```

### 04.G Test Grafana within Pods and outside
---

```

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl port-forward grafana-01-statefulset-0 3000:3000
Forwarding from 127.0.0.1:3000 -> 3000
Forwarding from [::1]:3000 -> 3000

Swarajits-MacBook-Air:~ swarajitroy$ curl http://localhost:3000
<a href="/login">Found</a>.


```

[Back to TOC](#table-of-content-toc)

## 05. AppMetric App - A prometheus exposition
---

| ID | Task | Remarks
| ----------- | ----------- | ------ |
| A | Application Overview  | |
| B | Build the application  |  |
| C | Application image  | | 
| D | Test using Docker | 

### 05.A Application Overview
---

The process of making metrics available to Prometheus is known as *Exposition*. *Exposition* to prometheus is done over HTTP. It is usually exposed under */metrics* path. 

A small application is created, which uses Java library to expose a *Counter* metric named *swararoy_counter_total* .
This counter gets incremented every *10 seconds* with a value of something between 5 and 10 (some random). It runs on port 80.

```
package org.ulearnuhelp.app.instrumentation;

import java.util.Random;

import io.prometheus.client.Counter;
import io.prometheus.client.exporter.HTTPServer;
import io.prometheus.client.hotspot.DefaultExports;


public class App 
{

    private static final Counter myCounter = Counter.build().name("swararoy_counter_total").help("sample metric").register();

    public static void main( String[] args ) throws Exception
    {
        DefaultExports.initialize(); 
        HTTPServer server = new HTTPServer(8000) ;
        System.out.println("Application metric exposition endpoint started ...") ;

        while(true) {

            double min = 5;
            double max = 10;
            Random r = new Random();
            double randomValue = min + (max - min) * r.nextDouble();
            
            myCounter.inc(randomValue);
            System.out.println("incremented counter by = " + randomValue + " and new counter value = " + myCounter.get()) ; 
            Thread.sleep(10000);
        }


    }
}

```

### 05.C Application image
---
Docker follows the naming convention to identify unofficial images. What we are creating is an unofficial image. Hence, it should follow that syntax. According to that naming convention, the unofficial image name should be named as follows: <username>/<image_name>:<tag_name>

```
Swarajits-MacBook-Air:BUILD swarajitroy$ docker info | grep -i registry
Registry: https://index.docker.io/v1/

Swarajits-MacBook-Air:BUILD swarajitroy$ docker images | grep -i appmetric
appmetric                          latest              d93a1b051840        9 days ago          105MB
Swarajits-MacBook-Air:BUILD swarajitroy$ docker tag appmetric:latest swarajitroy/appmetric:latest
Swarajits-MacBook-Air:BUILD swarajitroy$ docker images | grep appmetric
appmetric                          latest              d93a1b051840        9 days ago          105MB
swarajitroy/appmetric              latest              d93a1b051840        9 days ago          105MB

Swarajits-MacBook-Air:BUILD swarajitroy$ docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: swarajitroy
Password:
Login Succeeded

Swarajits-MacBook-Air:BUILD swarajitroy$ docker push swarajitroy/appmetric:latest
The push refers to repository [docker.io/swarajitroy/appmetric]
58d407a44faa: Pushed
b5b7e41a2428: Pushed
ceaf9e1ebef5: Mounted from library/openjdk
9b9b7f3d56a0: Mounted from library/openjdk
f1b5933fe4b5: Mounted from library/openjdk
latest: digest: sha256:95147fc2d638e5468c4a352490098875a245ddf54404454d5dc348b1d607f019 size: 1363


```

### 05.D Test using Docker
---

```
Swarajits-MacBook-Air:BUILD swarajitroy$ docker pull swarajitroy/appmetric:latest
latest: Pulling from swarajitroy/appmetric
e7c96db7181b: Already exists
f910a506b6cb: Already exists
c2274a1a0e27: Already exists
a1ea3230fcff: Already exists
131211ee82d7: Already exists
Digest: sha256:95147fc2d638e5468c4a352490098875a245ddf54404454d5dc348b1d607f019
Status: Downloaded newer image for swarajitroy/appmetric:latest
docker.io/swarajitroy/appmetric:latest

Swarajits-MacBook-Air:BUILD swarajitroy$ docker images
REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
swarajitroy/appmetric              latest              d93a1b051840        9 days ago          105MB

Swarajits-MacBook-Air:BUILD swarajitroy$ docker run -it -p 8000:8000 swarajitroy/appmetric
Application metric exposition endpoint started ...
incremented counter by = 7.049043113575238 and new counter value = 7.049043113575238

Swarajits-MacBook-Air:victoriametrics swarajitroy$ curl http://localhost:8000/metrics | grep swararoy

# HELP swararoy_counter_total sample metric
# TYPE swararoy_counter_total counter
swararoy_counter_total 50.83854533721948

```
[Back to TOC](#table-of-content-toc)


## 06. Run AppMetric endpoint Instance 01 as a deployment
---

| ID | Task | Remarks
| ----------- | ----------- | ------ |
| A | Deployment manifest with Replicaset  | |
| B | Service (ClusterIP default)  | |
| B | Test the Endpoint   |  |

### 06.A Deployment manifest with Replicaset 
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat appmetric_01-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: appmetric-01-deployment
  labels:
    app: appmetric-01-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: appmetric-01
  template:
    metadata:
      labels:
        app: appmetric-01
    spec:
      containers:
      - name: appmetric-01
        image: swarajitroy/appmetric:latest
        ports:
        - containerPort: 8000
        

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f appmetric_01-deployment.yaml
deployment.apps/appmetric-01-deployment created

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get deployments
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
appmetric-01-deployment   1/1     1            1           18s

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe deployments appmetric-01-deployment
Name:                   appmetric-01-deployment
Namespace:              default
CreationTimestamp:      Sat, 14 Nov 2020 21:14:57 +0530
Labels:                 app=appmetric-01-deployment
Annotations:            deployment.kubernetes.io/revision: 1
                        kubectl.kubernetes.io/last-applied-configuration:
                          {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"labels":{"app":"appmetric-01-deployment"},"name":"appmetric-01-d...
Selector:               app=appmetric-01
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=appmetric-01
  Containers:
   appmetric-01:
    Image:        swarajitroy/appmetric:latest
    Port:         8000/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   appmetric-01-deployment-58c7c65667 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  39s   deployment-controller  Scaled up replica set appmetric-01-deployment-58c7c65667 to 1
  
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get rs
NAME                                 DESIRED   CURRENT   READY   AGE
appmetric-01-deployment-58c7c65667   1         1         1       86s

Name:           appmetric-01-deployment-58c7c65667
Namespace:      default
Selector:       app=appmetric-01,pod-template-hash=58c7c65667
Labels:         app=appmetric-01
                pod-template-hash=58c7c65667
Annotations:    deployment.kubernetes.io/desired-replicas: 1
                deployment.kubernetes.io/max-replicas: 2
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/appmetric-01-deployment
Replicas:       1 current / 1 desired
Pods Status:    1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=appmetric-01
           pod-template-hash=58c7c65667
  Containers:
   appmetric-01:
    Image:        swarajitroy/appmetric:latest
    Port:         8000/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  108s  replicaset-controller  Created pod: appmetric-01-deployment-58c7c65667-fxl7m

```
### 06.B Service (default ClusterIP) manifest 
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat appmetric_01-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: appmetric-01-service
spec:
  ports:
  - port: 8000
    protocol: TCP
  selector:
    app: appmetric-01
 
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f appmetric_01-service.yaml
service/appmetric-01-service created

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get svc
NAME                                TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
appmetric-01-service                ClusterIP   10.98.249.1   <none>        8000/TCP   3s
grafana-service                     ClusterIP   None          <none>        8080/TCP   3d3h
kubernetes                          ClusterIP   10.96.0.1     <none>        443/TCP    7d9h
prom-01-service                     ClusterIP   None          <none>        9090/TCP   5d1h
prom-02-service                     ClusterIP   None          <none>        9090/TCP   45h
victoria-metrics-headless-service   ClusterIP   None          <none>        80/TCP     6d3h

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe service appmetric-01-service
Name:              appmetric-01-service
Namespace:         default
Labels:            <none>
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"appmetric-01-service","namespace":"default"},"spec":{"ports":[{"p...
Selector:          app=appmetric-01
Type:              ClusterIP
IP:                10.98.249.1
Port:              <unset>  8000/TCP
TargetPort:        8000/TCP
Endpoints:         172.17.0.11:8000
Session Affinity:  None
Events:            <none> 
```

### 06.C Test Endpoint
---

```
dnstools# curl http://appmetric-01-service.default.svc.cluster.local:8000/metrics | grep swararoy

# TYPE swararoy_counter_total counter
swararoy_counter_total 525.5081195440723
```
[Back to TOC](#table-of-content-toc)

## 07. Run AppMetric endpoint Instance 02 as a deployment
---

| ID | Task | Remarks
| ----------- | ----------- | ------ |
| A | Deployment manifest with Replicaset  | |
| B | Service (ClusterIP default)  | |
| C | Test the Endpoint   |  |

### 07.A Deployment manifest with Replicaset 
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat appmetric_02-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: appmetric-02-deployment
  labels:
    app: appmetric-02-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: appmetric-02
  template:
    metadata:
      labels:
        app: appmetric-02
    spec:
      containers:
      - name: appmetric-02
        image: swarajitroy/appmetric:latest
        ports:
        - containerPort: 8000
        
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f appmetric_02-deployment.yaml
deployment.apps/appmetric-02-deployment created

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get deployments
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
appmetric-01-deployment   1/1     1            1           34m
appmetric-02-deployment   1/1     1            1           25s

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe deployment appmetric-02-deployment
Name:                   appmetric-02-deployment
Namespace:              default
CreationTimestamp:      Sat, 14 Nov 2020 21:49:16 +0530
Labels:                 app=appmetric-02-deployment
Annotations:            deployment.kubernetes.io/revision: 1
                        kubectl.kubernetes.io/last-applied-configuration:
                          {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"labels":{"app":"appmetric-02-deployment"},"name":"appmetric-02-d...
Selector:               app=appmetric-02
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=appmetric-02
  Containers:
   appmetric-02:
    Image:        swarajitroy/appmetric:latest
    Port:         8000/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   appmetric-02-deployment-d6949984c (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  50s   deployment-controller  Scaled up replica set appmetric-02-deployment-d6949984c to 1
  
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get rs
NAME                                 DESIRED   CURRENT   READY   AGE
appmetric-01-deployment-58c7c65667   1         1         1       35m
appmetric-02-deployment-d6949984c    1         1         1       92s
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe rs appmetric-02-deployment-d6949984c
Name:           appmetric-02-deployment-d6949984c
Namespace:      default
Selector:       app=appmetric-02,pod-template-hash=d6949984c
Labels:         app=appmetric-02
                pod-template-hash=d6949984c
Annotations:    deployment.kubernetes.io/desired-replicas: 1
                deployment.kubernetes.io/max-replicas: 2
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/appmetric-02-deployment
Replicas:       1 current / 1 desired
Pods Status:    1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=appmetric-02
           pod-template-hash=d6949984c
  Containers:
   appmetric-02:
    Image:        swarajitroy/appmetric:latest
    Port:         8000/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  103s  replicaset-controller  Created pod: appmetric-02-deployment-d6949984c-zgt9f

```

### 07.B Service (default ClusterIP) manifest 
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat appmetric_02-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: appmetric-02-service
spec:
  ports:
  - port: 8000
    protocol: TCP
  selector:
    app: appmetric-02
    
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f appmetric_02-service.yaml
service/appmetric-02-service created    

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get svc
NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
appmetric-01-service                ClusterIP   10.98.249.1     <none>        8000/TCP   45m
appmetric-02-service                ClusterIP   10.100.111.54   <none>        8000/TCP   2m4s
grafana-service                     ClusterIP   None            <none>        8080/TCP   3d4h
prom-01-service                     ClusterIP   None            <none>        9090/TCP   5d2h
prom-02-service                     ClusterIP   None            <none>        9090/TCP   46h
victoria-metrics-headless-service   ClusterIP   None            <none>        80/TCP     6d4h

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe svc appmetric-02-service
Name:              appmetric-02-service
Namespace:         default
Labels:            <none>
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"appmetric-02-service","namespace":"default"},"spec":{"ports":[{"p...
Selector:          app=appmetric-02
Type:              ClusterIP
IP:                10.100.111.54
Port:              <unset>  8000/TCP
TargetPort:        8000/TCP
Endpoints:         172.17.0.12:8000
Session Affinity:  None
Events:            <none>

```

07.C Test Endpoint
---

```
kubectl run --restart=Never --rm -it --image infoblox/dnstools swararoy-dnstools
If you don't see a command prompt, try pressing enter.
dnstools# curl http://appmetric-02-service.default.svc.cluster.local:8000/metrics | grep swararoy

# HELP swararoy_counter_total sample metric
# TYPE swararoy_counter_total counter
swararoy_counter_total 998.590914448301

```

[Back to TOC](#table-of-content-toc)

## 08. Configure Prometheus Instance 01 to scrape AppMetric endpoint Instance 01
---

| ID | Task | Remarks
| ----------- | ----------- | ------ |
| A | Update prometheus.yml (configMap) | Include scrapping |
| B | Restart Prometheus statefulset 01  | |
| C | Test the Endpoint   |  |

### 08.A | Update prometheus.yml (configMap)
---

Following snippet is added.

```
 - job_name: 'appmetric-01'
      static_configs:
        - targets: ['appmetric-01-service.default.svc.cluster.local:8000']
```

```

global:
       scrape_interval:     15s
       evaluation_interval: 15s
       external_labels:
           openshift_instance: SDI_CAAS_QA
           datacenter: DC-01

    alerting:
       alertmanagers:
         - static_configs:
            - targets:
              # - alertmanager:9093


    scrape_configs:

    - job_name: 'prometheus'
      static_configs:
        - targets: ['localhost:9090']
    - job_name: 'appmetric-01'
      static_configs:
        - targets: ['appmetric-01-service.default.svc.cluster.local:8000']
```
and configMap was reapplied

```
kubectl apply -f prometheus_01-configMap.yaml

```
### 08.B | Restart Prometheus Statefulset 01
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl rollout restart statefulset prom-01-statefulset
```

### 08.C | Test the Endpoint
---

[Back to TOC](#table-of-content-toc)

## 09. Configure Prometheus Instance 02 to scrape AppMetric endpoint Instance 02
---

| ID | Task | Remarks
| ----------- | ----------- | ------ |
| A | Update prometheus.yml (configMap) | Include scrapping |
| B | Restart Prometheus statefulset 02  | |
| C | Test the Endpoint   |  |

### 09.A | Update prometheus.yml (configMap)
---

Following snippet is added.

```
 - job_name: 'appmetric-01'
      static_configs:
        - targets: ['appmetric-02-service.default.svc.cluster.local:8000']
```

```

global:
       scrape_interval:     15s
       evaluation_interval: 15s
       external_labels:
           openshift_instance: SDI_CAAS_QA
           datacenter: DC-01

    alerting:
       alertmanagers:
         - static_configs:
            - targets:
              # - alertmanager:9093


    scrape_configs:

    - job_name: 'prometheus'
      static_configs:
        - targets: ['localhost:9090']
    - job_name: 'appmetric-01'
      static_configs:
        - targets: ['appmetric-02-service.default.svc.cluster.local:8000']
```
and configMap was reapplied

```
kubectl apply -f prometheus_01-configMap.yaml

```

### 09.B | Restart Prometheus Statefulset 02
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl rollout restart statefulset prom-02-statefulset
```
### 09.C | Test the Endpoint
---

[Back to TOC](#table-of-content-toc)

## 10. Configure Prometheus Instance 01 to remote write to VictoriaMetric
---

| ID | Task | Remarks
| ----------- | ----------- | ------ |
| A | Update prometheus.yml (configMap) | Include remote rewrite |
| B | Restart Prometheus statefulset 02  | |
| C | Test the Endpoint  

### 10.A Update prometheus.yml (configMap) - remote rewrite
---

Add the following snippet, 

```
remote_write:
  - url: http://victoria-metrics-headless-service.default.svc.cluster.local:8428/api/v1/write
  
```
and update the configMap,

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat prometheus_01-configMap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-01-server-configmap
  labels:
    name: prometheus-01-server-configmap
data:
  prometheus.yml: |-
    global:
       scrape_interval:     15s
       evaluation_interval: 15s
       external_labels:
           openshift_instance: SDI_CAAS_QA
           datacenter: DC-01

    alerting:
       alertmanagers:
         - static_configs:
            - targets:
              # - alertmanager:9093

    remote_write:
      - url: http://victoria-metrics-headless-service.default.svc.cluster.local:8428/api/v1/write

    scrape_configs:

    - job_name: 'prometheus'
      static_configs:
        - targets: ['localhost:9090']
    - job_name: 'appmetric-01'
      static_configs:
        - targets: ['appmetric-01-service.default.svc.cluster.local:8000']

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f prometheus_01-configMap.yaml
configmap/prometheus-01-server-configmap configured

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get configmaps
NAME                                  DATA   AGE
prometheus-01-server-configmap        1      5d21h
prometheus-02-server-configmap        1      2d16h
swararoy-grafana-dashboardproviders   1      4d6h
swararoy-grafana-datasources          1      21h
swararoy-grafana-ini                  1      4d6h
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe configmap prometheus-01-server-configmap
Name:         prometheus-01-server-configmap
Namespace:    default
Labels:       name=prometheus-01-server-configmap
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"v1","data":{"prometheus.yml":"global:\n   scrape_interval:     15s \n   evaluation_interval: 15s \n   external_labels:\n   ...

Data
====
prometheus.yml:
----
global:
   scrape_interval:     15s
   evaluation_interval: 15s
   external_labels:
       openshift_instance: SDI_CAAS_QA
       datacenter: DC-01

alerting:
   alertmanagers:
     - static_configs:
        - targets:
          # - alertmanager:9093

remote_write:
  - url: http://victoria-metrics-headless-service.default.svc.cluster.local:8428/api/v1/write

scrape_configs:

- job_name: 'prometheus'
  static_configs:
    - targets: ['localhost:9090']
- job_name: 'appmetric-01'
  static_configs:
    - targets: ['appmetric-01-service.default.svc.cluster.local:8000']
Events:  <none>
```
### 10.B | Restart Prometheus Statefulset 01
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl rollout restart statefulset prom-01-statefulset
```

[Back to TOC](#table-of-content-toc)

## 11. Configure Prometheus Instance 01 to remote write to VictoriaMetric
---

| ID | Task | Remarks
| ----------- | ----------- | ------ |
| A | Update prometheus.yml (configMap) | Include remote rewrite |
| B | Restart Prometheus statefulset 02  | |
| C | Test the Endpoint  

### 11.A Update prometheus.yml (configMap) - remote rewrite
---

Add the following snippet, 

```
remote_write:
  - url: http://victoria-metrics-headless-service.default.svc.cluster.local:8428/api/v1/write
  
```
and update the configMap,

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat prometheus_02-configMap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-02-server-configmap
  labels:
    name: prometheus-02-server-configmap
data:
  prometheus.yml: |-
    global:
       scrape_interval:     15s
       evaluation_interval: 15s
       external_labels:
           openshift_instance: SDI_CAAS_PROD
           datacenter: DC-02

    alerting:
       alertmanagers:
         - static_configs:
            - targets:
              # - alertmanager:9093

    remote_write:
      - url: http://victoria-metrics-headless-service.default.svc.cluster.local:8428/api/v1/write

    scrape_configs:

    - job_name: 'prometheus'
      static_configs:
        - targets: ['localhost:9090']
    - job_name: 'appmetric-02'
      static_configs:
        - targets: ['appmetric-02-service.default.svc.cluster.local:8000']

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f prometheus_02-configMap.yaml
configmap/prometheus-02-server-configmap configured

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get configmaps
NAME                                  DATA   AGE
prometheus-01-server-configmap        1      5d21h
prometheus-02-server-configmap        1      2d17h
swararoy-grafana-dashboardproviders   1      4d6h
swararoy-grafana-datasources          1      22h
swararoy-grafana-ini                  1      4d6h
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe configmap prometheus-02-server-configmap
Name:         prometheus-02-server-configmap
Namespace:    default
Labels:       name=prometheus-02-server-configmap
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"v1","data":{"prometheus.yml":"global:\n   scrape_interval:     15s \n   evaluation_interval: 15s \n   external_labels:\n   ...

Data
====
prometheus.yml:
----
global:
   scrape_interval:     15s
   evaluation_interval: 15s
   external_labels:
       openshift_instance: SDI_CAAS_PROD
       datacenter: DC-02

alerting:
   alertmanagers:
     - static_configs:
        - targets:
          # - alertmanager:9093

remote_write:
  - url: http://victoria-metrics-headless-service.default.svc.cluster.local:8428/api/v1/write

scrape_configs:

- job_name: 'prometheus'
  static_configs:
    - targets: ['localhost:9090']
- job_name: 'appmetric-02'
  static_configs:
    - targets: ['appmetric-02-service.default.svc.cluster.local:8000']
Events:  <none>

```

### 11.B | Restart Prometheus Statefulset 02
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl rollout restart statefulset prom-02-statefulset
```
[Back to TOC](#table-of-content-toc)

## 12 End to End Demo using Grafana & PromQL 
---

```
#curl http://prom-01-service.default.svc.cluster.local:9090/api/v1/query?query=swararoy_counter_total

{"status":"success","data":
  {"resultType":"vector",
   "result":[
      {"metric":{"__name__":"swararoy_counter_total","instance":"appmetric-01-service.default.svc.cluster.local:8000","job":"appmetric-01"},"value":[1605445678.197,"13615.200835508138"]
      }]}}


#curl http://prom-02-service.default.svc.cluster.local:9090/api/v1/query?query=swararoy_counter_total

{"status":"success",
 "data":{"resultType":"vector",
 "result":[{"metric":{"__name__":"swararoy_counter_total","instance":"appmetric-02-service.default.svc.cluster.local:8000","job":"appmetric-02"},"value":[1605445876.307,"12581.609012379498"]}]}}#

```

```
# curl http://victoria-metrics-headless-service.default.svc.cluster.local:8428/api/v1/query?query=swararoy_counter_total

{"status":"success","data":
  {"resultType":"vector","result":
    [{"metric":{"__name__":"swararoy_counter_total","datacenter":"DC-01","instance":"appmetric-01-   service.default.svc.cluster.local:8000","job":"appmetric-01","openshift_instance":"SDI_CAAS_QA"},"value":[1605445554,"13615.2008355"]},
    {"metric":{"__name__":"swararoy_counter_total","datacenter":"DC-02","instance":"appmetric-02-service.default.svc.cluster.local:8000","job":"appmetric-02","openshift_instance":"SDI_CAAS_PROD"},"value":[1605445554,"12423.22608904"]}
    ]
  }
}

```

## 13 VictoriaMetric Additional 
---

### 13.1 VictoriaMetric Security - Authentication
---

In the context of an HTTP transaction, basic access authentication is a method for an HTTP user agent (e.g. a web browser) to provide a user name and password when making a request. In basic HTTP authentication, a request contains a header field in the form of Authorization: Basic <credentials>, where credentials is the Base64 encoding of ID and password joined by a single colon.
 
VictoriaMetrics supports HTTP Basic Authentication via following flags

-httpAuth.password string
        Password for HTTP Basic Auth. The authentication is disabled if -httpAuth.username is empty
-httpAuth.username string
        Username for HTTP Basic Auth. The authentication is disabled if empty. See also -httpAuth.password

We will use them as secrets https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/

| ID | Task | Remarks
| ----------- | ----------- | ------ |
| A | Base64 encode UserID and Password |  |
| B | Create Kubernetes Secret |  |
| C | Inject secret to victoriametrics container |  |

#### 13.1. A Base64 encode UserID and Password
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ echo -n 'vmetrics' | base64
dm1ldHJpY3M=
Swarajits-MacBook-Air:victoriametrics swarajitroy$ echo -n 'hello123' | base64
aGVsbG8xMjM=

```
#### 13.1. B Create Kubernetes Secret
---

```

Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat victoriametric-auth-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: vmetrics-auth-secret
data:
  username: dm1ldHJpY3M=
  password: aGVsbG8xMjM=

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl apply -f victoriametric-auth-secret.yaml
secret/vmetrics-auth-secret created

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get secrets
NAME                   TYPE                                  DATA   AGE

vmetrics-auth-secret   Opaque                                2      9s

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe secret vmetrics-auth-secret
Name:         vmetrics-auth-secret
Namespace:    default
Labels:       <none>
Annotations:
Type:         Opaque

Data
====
password:  8 bytes
username:  8 bytes


```

#### 13.1. C Inject secret to victoriametrics container
---

```
spec:
      containers:
      - name: victoria-metrics
        image: victoriametrics/victoria-metrics
        env:
        - name: HTTPAUTH_USERNAME
          valueFrom:
            secretKeyRef:
              name: vmetrics-auth-secret
              key: username
        - name: HTTPAUTH_PASSWORD
          valueFrom:
            secretKeyRef:
              name: vmetrics-auth-secret
              key: password
        args:
            - "-httpAuth.username=$(HTTPAUTH_USERNAME)"
            - "-httpAuth.password=$(HTTPAUTH_PASSWORD)"
```

The moment VictoriaMetrics is enabled for HTTP basic authentication, the prometheus will need to be updated - otherwise following error will be coming,

```
ts=2020-11-20T07:34:21.551Z caller=dedupe.go:112 component=remote level=error remote_name=e65afc url=http://victoria-metrics-headless-service.default.svc.cluster.local:8428/api/v1/write msg="non-recoverable error" count=122 err="server returned HTTP status 401 Unauthorized: "

```
The prometheus configuration has to be changed as well, to accomodate the HTTP basic authentication.the config map will be updated and restart prometheus

```
remote_write:
  - url: http://victoria-metrics-headless-service.default.svc.cluster.local:8428/api/v1/write
    basic_auth:
      username: vmetrics
      password: hello123

```

### 13.2 VictoriaMetric Security - Encryption
---

According to VictoriaMetrics website - Prefer ECDSA certs instead of RSA certs, since RSA certs are slow. There is good documentation to create ECDSA private key and CA certification out here - https://www.scottbrady91.com/OpenSSL/Creating-Elliptical-Curve-Keys-using-OpenSSL . If at all same is required for RSA - we have https://www.scottbrady91.com/OpenSSL/Creating-RSA-Keys-using-OpenSSL

| ID | Task | Remarks
| ----------- | ----------- | ------ |
| A | Check OpenSSL availability |  |
| B | Select Algorithm |  |
| C | Create the Private Key |  |
| D | Create the Public Key |  |
| E | Create the self signed certificate from the Public Key |  |

#### 13.2.A Check OpenSSL availability
---

```
Swarajits-MacBook-Air:cp-all-in-one-community swarajitroy$ which openssl
/usr/bin/openssl
Swarajits-MacBook-Air:cp-all-in-one-community swarajitroy$ openssl version
LibreSSL 2.2.7

```
#### 13.2.B Select Algorithm
---
```
Swarajits-MacBook-Air:cp-all-in-one-community swarajitroy$ openssl ecparam -list_curves | grep prime256v1
  prime256v1: X9.62/SECG curve over a 256 bit prime field


```
#### 13.2.C Create the Private Key
---
```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ openssl ecparam -name prime256v1 -genkey -noout -out vmetrics-private-key.pem

Swarajits-MacBook-Air:victoriametrics swarajitroy$ ls -l vmetrics-private-key.pem
-rw-r--r--  1 swarajitroy  staff  227 Nov 23 09:16 vmetrics-private-key.pem

Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat vmetrics-private-key.pem
-----BEGIN EC PRIVATE KEY-----
gghhdhdK3Q==
-----END EC PRIVATE KEY-----

```
#### 13.2.D Create the Public Key
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ openssl ec -in vmetrics-private-key.pem -pubout -out vmetrics-public-key.pem
read EC key
writing EC key

Swarajits-MacBook-Air:victoriametrics swarajitroy$ ls -l vmetrics-public-key.pem
-rw-r--r--  1 swarajitroy  staff  178 Nov 23 09:19 vmetrics-public-key.pem

Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat vmetrics-public-key.pem
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE5AwJo3VOtUbI1urgubYJNx8xqI7I
6DULUQIHtaQ1BlsF0wkZZI2eDXa/SPGuUCCWyq5dO49j+eodQyH7RJPK3Q==
-----END PUBLIC KEY-----

```

#### 13.2.E Create the self signed certificate
---

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ openssl req -new -x509 -key vmetrics-private-key.pem -out vmetrics-cert.pem -days 360
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) []:IN
State or Province Name (full name) []:WB
Locality Name (eg, city) []:KOLKATA
Organization Name (eg, company) []:.
Organizational Unit Name (eg, section) []:.
Common Name (eg, fully qualified host name) []:.
Email Address []:.

Swarajits-MacBook-Air:victoriametrics swarajitroy$ cat vmetrics-cert.pem
-----BEGIN CERTIFICATE-----
MIIBRjCB7gIJALAF6umWErQ9MAoGCCqGSM49BAMCMCwxCzAJBgNVBAYTAklOMQsw
CQYDVQQIDAJXQjEQMA4GA1UEBwwHS09MS0FUQTAeFw0yMDExMjMwNDAwNDhaFw0y
MTExMTgwNDAwNDhaMCwxCzAJBgNVBAYTAklOMQswCQYDVQQIDAJXQjEQMA4GA1UE
BwwHS09MS0FUQTBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABOQMCaN1TrVGyNbq
4Lm2CTcfMaiOyOg1C1ECB7WkNQZbBdMJGWSNng12v0jxrlAglsquXTuPY/nqHUMh
+0STyt0wCgYIKoZIzj0EAwIDRwAwRAIgeLGbwbh/BnP9oQLNpNX99A6aXcWa3cEK
XBWHitqs140CIBHlpdfHP4b2CHXqG8S3A1DkQYh6mmYarNqOmRbkoPdX
-----END CERTIFICATE-----

```

### 13.3 VictoriaMetric Security - vmauth
---

### 13.4 Helm Chart 
---

VictoriaMetrics has good support for HELM - https://github.com/VictoriaMetrics/helm-charts

```
Swarajits-MacBook-Air:helm-charts swarajitroy$ kubectl create namespace vemtrics-ns
namespace/vemtrics-ns created

```

The chart github repository is https://github.com/swarajitroy/kubemon-victoriametrics/tree/main/helm-charts/victoria-metrics-standalone

```
Swarajits-MacBook-Air:helm-charts swarajitroy$ helm install vmsingle victoria-metrics-standalone  -n vmetrics-ns
NAME: vmsingle
LAST DEPLOYED: Sat Nov 21 16:06:17 2020
NAMESPACE: vmetrics-ns
STATUS: deployed
REVISION: 1
TEST SUITE: None

Swarajits-MacBook-Air:helm-charts swarajitroy$ helm list -f vmsingle -n vemtrics-ns
NAME    	NAMESPACE  	REVISION	UPDATED                             	STATUS  	CHART                            	APP VERSION
vmsingle	vemtrics-ns	1       	2020-11-21 16:06:17.164853 +0530 IST	deployed	victoria-metrics-standalone-0.0.1	1.1.0

Swarajits-MacBook-Air:helm-charts swarajitroy$ helm uninstall -name vmsingle -n vmetrics-ns
release "vmsingle" uninstalled

Swarajits-MacBook-Air:helm-charts swarajitroy$ helm upgrade vmsingle victoria-metrics-standalone  -n vemtrics-ns
Release "vmsingle" has been upgraded. Happy Helming!
NAME: vmsingle
LAST DEPLOYED: Sat Nov 21 17:34:16 2020
NAMESPACE: vmetrics-ns
STATUS: deployed
REVISION: 2
TEST SUITE: None

Swarajits-MacBook-Air:helm-charts swarajitroy$ helm ls -n vemtrics-ns
NAME    	NAMESPACE  	REVISION	UPDATED                             	STATUS  	CHART                            	APP VERSION
vmsingle	vmetrics-ns	2       	2020-11-21 17:34:16.556536 +0530 IST	deployed	victoria-metrics-standalone-0.0.1	1.1.0

```

### 13.5 Kubernetes Operator
---

### 13.6 ArgoCD
---

https://levelup.gitconnected.com/integrating-argo-cd-for-your-kubernetes-project-ba6e49dfebaa

| ID | Task | Remarks
| ----------- | ----------- | ------ |
| A | Install ArgoCD | |
| B | Connect to Kubernetes |
| C | Connect to Source Repository |

#### 13.6.A Install ArgoCD
---
Argo CD is an open-source continuous delivery tool which runs on Kubernetes.

```
Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl create namespace argocd
namespace/argocd created

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl get ns
NAME                   STATUS   AGE
argocd                 Active   7s
default                Active   13d
kube-node-lease        Active   13d
kube-public            Active   13d
kube-system            Active   13d
kubernetes-dashboard   Active   13d

Swarajits-MacBook-Air:victoriametrics swarajitroy$ kubectl describe ns argocd
Name:         argocd
Labels:       <none>
Annotations:  <none>
Status:       Active

No resource quota.

No resource limits.

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

Swarajits-MacBook-Air:~ swarajitroy$ kubectl get pods -n argocd
NAME                                             READY   STATUS    RESTARTS   AGE
argocd-application-controller-6c484d79c6-82fsz   1/1     Running   1          18h
argocd-dex-server-b87b54696-ldlfs                1/1     Running   1          18h
argocd-redis-646bb7bf78-tk7gs                    1/1     Running   1          18h
argocd-repo-server-694547b4db-g4t2l              1/1     Running   1          18h
argocd-server-6987c9748c-c64zc                   1/1     Running   1          18h

Swarajits-MacBook-Air:~ swarajitroy$ kubectl get svc -n argocd
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
argocd-dex-server       ClusterIP   10.101.182.197   <none>        5556/TCP,5557/TCP,5558/TCP   18h
argocd-metrics          ClusterIP   10.109.127.94    <none>        8082/TCP                     18h
argocd-redis            ClusterIP   10.106.181.193   <none>        6379/TCP                     18h
argocd-repo-server      ClusterIP   10.108.52.81     <none>        8081/TCP,8084/TCP            18h
argocd-server           ClusterIP   10.101.172.166   <none>        80/TCP,443/TCP               18h
argocd-server-metrics   ClusterIP   10.110.46.217    <none>        8083/TCP                     18h

Swarajits-MacBook-Air:~ swarajitroy$ kubectl port-forward svc/argocd-server -n argocd 8080:443
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080

```



### 13.7 Monitoring
---

### 13.8 Backup
---

### 13.9 Alerting
---
