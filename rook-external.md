# External Storage Cluster
Running Ceph in external mode, Rook will provide the configuration for the CSI driver and other basic
resources that allows your applications to connect to Ceph in the external cluster.

## Source Ceph cluster
提取Ceph cluster信息，用于Rook连接。
#### 创建users和keys
#### 创建用于k8s的rbd和cephfs的pool
```shell
root@pvg03s1cphap01:~# ceph osd pool create rbd-k8s ssd
root@pvg03s1cphap01:~# ceph osd pool application enable rbd-k8s rbd

root@pvg03s1cphap01:~# ceph osd pool create cephfs-k8s ssd
pool 'cephfs-k8s' created

root@pvg03s1cphap01:~# ceph fs add_data_pool cephfs cephfs-k8s
added data pool 19 to fsmap
root@pvg03s1cphap01:~# ceph fs ls
name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data cephfs-k8s ]

```

#### 创建users和keys
```shell
root@pvg03s1cphap01:~# python3 create-external-cluster-resources.py --rbd-data-pool-name rbd-k8s --cephfs-filesystem-name cephfs --cephfs-data-pool-name cephfs-k8s --namespace rook-ceph-external --format bash --ceph-conf /etc/ceph/ceph.conf
WARNING: Multiple data pools detected: ['cephfs_data', 'cephfs-k8s']
Using the data-pool: 'cephfs-k8s'

export NAMESPACE=rook-ceph-external
export ROOK_EXTERNAL_FSID=6c8ff8fc-2e5f-11ed-8406-c99615413fba
export ROOK_EXTERNAL_USERNAME=client.healthchecker
export ROOK_EXTERNAL_CEPH_MON_DATA=pvg03s1cphap01=10.120.94.6:6789
export ROOK_EXTERNAL_USER_SECRET=AQCaZy1jGIDhNxAAOhSPIghUH652kHA2gcHgSg==
export ROOK_EXTERNAL_DASHBOARD_LINK=https://10.120.94.6:8443/
export CSI_RBD_NODE_SECRET=AQCaZy1jLrEgOBAAThFoOFm1pKxO6R8JCIvLww==
export CSI_RBD_NODE_SECRET_NAME=csi-rbd-node
export CSI_RBD_PROVISIONER_SECRET=AQCaZy1jJypaOBAALO7xaE7WEB1zJxle18PKFA==
export CSI_RBD_PROVISIONER_SECRET_NAME=csi-rbd-provisioner
export CEPHFS_POOL_NAME=cephfs-k8s
export CEPHFS_METADATA_POOL_NAME=cephfs_metadata
export CEPHFS_FS_NAME=cephfs
export CSI_CEPHFS_NODE_SECRET=AQCaZy1j5CyVOBAA43fBNooviQOvqGZ0dPXONg==
export CSI_CEPHFS_PROVISIONER_SECRET=AQCaZy1jSDHQOBAAlsrSGJh/DP08xUbwqLroKg==
export CSI_CEPHFS_NODE_SECRET_NAME=csi-cephfs-node
export CSI_CEPHFS_PROVISIONER_SECRET_NAME=csi-cephfs-provisioner
export MONITORING_ENDPOINT=10.120.94.6
export MONITORING_ENDPOINT_PORT=9283
export RBD_POOL_NAME=rbd-k8s
export RGW_POOL_PREFIX=default

```

### 在k8s集群中deploy rook
#### 创建common resources
包含namespace 
#### 创建operator
#### 创建cluster CRD
```shell
 kubectl create -f common.yaml
 kubectl create -f crds.yaml -f operator.yaml
 kubectl create -f common-external.yaml
 kubectl create -f cluster-external.yaml

```
#### import ceph集群
```shell
root@juazhou01:~# . import-external-cluster.sh
secret/rook-ceph-mon created
configmap/rook-ceph-mon-endpoints created
secret/rook-csi-rbd-node created
secret/rook-csi-rbd-provisioner created
secret/rook-csi-cephfs-node created
secret/rook-csi-cephfs-provisioner created
secret/rgw-admin-ops-user created
storageclass.storage.k8s.io/ceph-rbd created
storageclass.storage.k8s.io/cephfs created
root@juazhou01:~#

```
#### 查看cluster以及storage class状态
```shell

root@juazhou01:~# kubectl get CephCluster -n rook-ceph-external
NAME                 DATADIRHOSTPATH   MONCOUNT   AGE   PHASE       MESSAGE                          HEALTH      EXTERNAL
rook-ceph-external                                84s   Connected   Cluster connected successfully   HEALTH_OK   true


root@juazhou01:~# kubectl get sc -n rook-ceph-external
NAME       PROVISIONER                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
ceph-rbd   rook-ceph.rbd.csi.ceph.com      Delete          Immediate           true                   9m55s
cephfs     rook-ceph.cephfs.csi.ceph.com   Delete          Immediate           true                   9m55s

```

#### 基于这两个storageclass创建Persistent Volume

##### CephFS
测试以下内容：

* 创建PVC
* Pod挂载PV
* 数据写入
* PVC Clone--从已有PVC中clone一个新的PVC

```shell
root@juazhou01:~/sctest# cat cephfs-pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc01
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: cephfs

root@juazhou01:~/sctest# kubectl create -f cephfs-pvc.yaml
persistentvolumeclaim/cephfs-pvc01 created
root@juazhou01:~/sctest#
root@juazhou01:~/sctest#
root@juazhou01:~/sctest# kubectl get pvc
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cephfs-pvc01   Bound    pvc-93bf70bb-43d6-4004-bb87-81314b714512   2Gi        RWO            cephfs         3s

```
创建一个nginx
```shell
root@juazhou01:~/sctest# cat cephfs-nginx-pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: cephfs-nginx-pod
spec:
  containers:
    - name: web-server
      image: nginx
      volumeMounts:
        - name: nginxpvc
          mountPath: /var/lib/www/html
  volumes:
    - name: nginxpvc
      persistentVolumeClaim:
        claimName: cephfs-pvc01
        readOnly: false

root@juazhou01:~/sctest# kubectl create -f cephfs-nginx-pod.yaml
pod/cephfs-nginx-pod created

```
进入pod，往/var/lib/www/html中写入数据
```shell

root@cephfs-nginx-pod:/# cat /var/lib/www/html/test.html
sdfjaslfjelkawrjlkjdfklaj

```

#### PVC clone
clone source: cephfs-pvc01
```shell
root@juazhou01:~/sctest# cat cephfs-pvc01-clone.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc01-clone
spec:
  storageClassName: cephfs
  dataSource:
    name: cephfs-pvc
    kind: PersistentVolumeClaim
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi

root@juazhou01:~/sctest# kubectl create -f cephfs-pvc01-clone.yaml
persistentvolumeclaim/cephfs-pvc01-clone created

```
查看pvc状态
```shell

root@juazhou01:~/sctest# kubectl get pvc
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cephfs-pvc01         Bound    pvc-93bf70bb-43d6-4004-bb87-81314b714512   2Gi        RWO            cephfs         27m
cephfs-pvc01-clone   Bound    pvc-a3210d2b-f6ee-49c6-8aca-e494b61a1031   2Gi        RWX            cephfs         12s


clone的pvc已经处于bound状态。

创建一个pod，挂载clone的pvc，进行数据验证
```shell
root@juazhou01:~/sctest# cat cephfs-nginxi02-pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: cephfs-nginx02-pod
spec:
  containers:
    - name: web-server
      image: nginx
      volumeMounts:
        - name: nginxpvc
          mountPath: /var/lib/www/html
  volumes:
    - name: nginxpvc
      persistentVolumeClaim:
        claimName: cephfs-pvc01-clone
        readOnly: false

```
```shell

root@juazhou01:~/sctest# kubectl create -f cephfs-nginxi02-pod.yaml
pod/cephfs-nginx02-pod created

```
查看clone的数据是否已经存在
```shell

root@juazhou01:~/sctest# kubectl exec -it cephfs-nginx02-pod -- /bin/bash

root@cephfs-nginx02-pod:/# cat /var/lib/www/html/test.html
sdfjaslfjelkawrjlkjdfklaj
root@cephfs-nginx02-pod:/#

```
与源pvc数据一致。

#### Ceph RBD
测试以下内容：

* 创建PVC
* Pod挂载PV
* 数据写入
* PVC Clone--从已有PVC中clone一个新的PVC

##### 创建PVC
```shell

root@juazhou01:~/sctest# cat cephrbd-pvc01.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephrbd-pvc01
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: ceph-rbd
root@juazhou01:~/sctest#
root@juazhou01:~/sctest# kubectl create -f cephrbd-pvc01.yaml
persistentvolumeclaim/cephrbd-pvc01 created
root@juazhou01:~/sctest# kubectl get pvc
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cephfs-pvc01         Bound    pvc-93bf70bb-43d6-4004-bb87-81314b714512   2Gi        RWO            cephfs         61m
cephfs-pvc01-clone   Bound    pvc-a3210d2b-f6ee-49c6-8aca-e494b61a1031   2Gi        RWX            cephfs         34m
cephrbd-pvc01        Bound    pvc-447d27f3-112a-49e8-93cf-e53b4d9b94a2   2Gi        RWO            ceph-rbd       4s

```
#### 创建一个Mysql Pod，挂载pvc cephrbd-pvc01
```shell
root@juazhou01:~/sctest# cat mysql-deployment.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
          # Use secret in real usage
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: cephrbd-pvc01
          
          

root@juazhou01:~/sctest# kubectl create -f mysql-deployment.yaml
service/mysql created
deployment.apps/mysql created

```

