# [译]利用StatefulSets部署PostgreSQL集群 

原文：[http://blog.kubernetes.io/2017/02/postgresql-clusters-kubernetes-statefulsets.html](http://blog.kubernetes.io/2017/02/postgresql-clusters-kubernetes-statefulsets.html)

本文介绍了利用Kubernetes的StatefulSets特性部署PostgreSQL集群的示例。

## 创建Kubernetes环境

StatefulSets 是Kubernetes1.5的新特性（在此之前称作PetSets）。因此运行此示例需要Kubernetes1.5.0及以上的版本。使用kubeadm部署介绍[见此](http://linoxide.com/containers/setup-kubernetes-kubeadm-centos)

## 安装NFS

本示例中使用NFS作为Persistent Volumes。当然也可以使用其他共享文件系统，如ceph、gluster。

示例中的脚本假设NFS服务运行在本地，并且hostname能够被正确解析。

在CentOS 7上脚本如下：

```sh
sudo setsebool -P virt_use_nfs 1
sudo yum -y install nfs-utils libnfsidmap
sudo systemctl enable rpcbind nfs-server
sudo systemctl start rpcbind nfs-server rpc-statd nfs-idmapd
sudo mkdir /nfsfileshare
sudo chmod 777 /nfsfileshare/
sudo vi /etc/exports
sudo exportfs -r
```

`/etc/exports`文件中，除了指定可用IP，还应该包含类似以下的一行，

```
/nfsfileshare 192.168.122.9(rw,sync)
```

到此，NFS应该正常运行在测试环境中了

## 克隆Crunchy PostgreSQL Container Suite

从GitHub上克隆Crunchy Containers仓库

```
cd $HOME
git clone https://github.com/CrunchyData/crunchy-containers.git
cd crunchy-containers/examples/kube/statefulset
```

拉取 Crunchy PostgreSQL容器镜像

```
docker pull crunchydata/crunchy-postgres:centos7-9.5-1.2.6
```

## 运行示例

开始前，设置下环境变量

```shell
export BUILDBASE=$HOME/crunchy-containers
export CCP_IMAGE_TAG=centos7-9.5-1.2.6
```

`BUILDBASE`是克隆的仓库的地址，`CCP_IMAGE_TAG`是镜像标签。

运行示例

```
./run.sh
```

这个脚本会创建以下几个Kubernetes对象：

-  Persistent Volumes (pv1, pv2, pv3)
-  Persistent Volume Claim (pgset-pvc)
-  Service Account (pgset-sa)
-  Services (pgset, pgset-master, pgset-replica)
-  StatefulSet (pgset)
-  Pods (pgset-0, pgset-1)

至此，已经有两个Pods已经运行在Kubernetes环境中了：

```
$ kubectl get pod
NAME      READY     STATUS    RESTARTS   AGE
pgset-0   1/1       Running   0          2m
pgset-1   1/1       Running   1          2m
```

部署描述如下：

![](https://lh5.googleusercontent.com/tGg-37a7SoVQR9Zn3R209iKbkegX5XqRQdRa5ZD6q-vpm1hWqtBxnhOBiGw2uHHkZ5lc_VBKrSEEP29BmAzoWc1xydV7G4I8kaQqVZoYOdRCvBf755Rxf9aj-pm7FhfmgECBW3gR)

## 刚发生了什么

在这示例中创建了一个StatefulSet，StatefulSet继而创建了两个pod。pod中的容器运行PostgreSQL 数据。对于PostgreSQL 集群，我们需要一个容器作为Master，另一个作为Replica。那么容器如何决定谁主谁备？

此时就需要了StateSet 机制来运作。StateSet 机制给集合中的每个pod分配了一个唯一有序的值，该值从0开始。每个容器在初始化期间，检查被分配的值，若为0则作为Master，否则作为Replica。

通过配置，PostgreSQL 备机经过Master的专用Service来访问Master数据库。在示例中，分别为Master和Replica两角色创建了两个独立的Service。一旦Replica连接成功，Replica开始从Master同步状态。

在容器初始化期间，Master容器会用Service Account](https://kubernetes.io/docs/user-guide/service-accounts/) (pgset-sa)来改变容器标签，以匹配Master的Service的选择器。

## 部署图

现在部署图如下所示

![](https://lh3.googleusercontent.com/5NthdAnA243jN_gXVlwZsg74jkGgCwQZh1yq78-8E0L7wuDgpdqH_AaeUvQd9RtXIlOV0cAWv1P0a_2oeVJN8fHstf9Iev1c-swGIqojIw0pXrVuqAqpCF3M5hw6sdTmx_1-Bg27)

在部署中，Master和Replica各有一个独立的Service。Replica已经连接上Master并且开始从Master同步状态。

Crunchy PostgreSQL容器还支持其他集群部署模式，通过设置容器的`PG_MODE `环境变量来选择不同的模式。在此示例中，设置为`PG_MODE=set`。

## 测试示例

如果系统中未安装psql 客户端，可以通过以下命令安装

```
sudo yum -y install postgresql
```

现假设环境中，DNS已经解析到Kub DNS，并且环境中的DNS搜索路径已近配置并且符合可用的Kube命名空间和域。Master的Server名字是`pgset-master`，Replica的Service叫`pgset-replica`。

通过以下命令测试Mater

```
psql -h pgset-master -U postgres postgres -c 'table pg_stat_replication'
```

正常情况下，上面的命令返回输出，显示一个Replica连接到Master。

下面测试Replica

```
psql -h pgset-replica -U postgres postgres  -c 'create table foo (id int)'
```

上面的命令应该失败，因为在PostgreSQL集群中，Replica是只读的。

下面扩展集群

```
kubectl scale statefulset pgset --replicas=3
```

命令成功运行后创建了一个新的Replica`pgset-2`

![](https://lh5.googleusercontent.com/w82XRPd9LqwgcoY3wJrilJEULxZyub6HLcFk332--1fd94-Vte4YlDFvspLM9syNCdT47PISJlEDo7jSPmiflFv-ZZKmrY6Jm6sJWMki0RfJigf6a6IEPNeyy1PJ_5Mhd4NW4rHm)

## 持久化

查看NFS挂载目录

```
$ ls -l /nfsfileshare/
total 12
drwx------ 20   26   26 4096 Jan 17 16:35 pgset-0
drwx------ 20   26   26 4096 Jan 17 16:35 pgset-1
drwx------ 20   26   26 4096 Jan 17 16:48 pgset-2
```

StatefulSets中的每个容器绑定到单独的NFS Persistent Volume Claim (pgset-pvc) 。因为NFS和PVC是可共享的，每个pod都可以往NFS路径上写。

此容器被设计成在共享路径上创建一个子目录，目录名称是pod的名称以防保持唯一。

## 结论

对于实现了集群的容器来说，StatefulSets是个令人振奋的特性。分配的有序值提供了一个简单的机制，使得集群很容易决定主备。

