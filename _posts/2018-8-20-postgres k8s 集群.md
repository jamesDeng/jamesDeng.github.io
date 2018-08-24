---
layout: article
key: k8s_postgres_cluster
tags: k8s postgres
comment: true
modify_date: 2018-8-24 10:30:00
---
# postgres集群方案
总结一下在k8s集群上部署一套高可用、读写分离的postgres集群的方案，大体上有以下三种方案可以选择：
### [Patroni](https://github.com/zalando/patroni)
只了解了一下官网说明，大体上应该是一套可以进行自定义模版
### [Crunchy](https://github.com/CrunchyData/crunchy-containers)
将现有的开源框架分别整理成Dokcer镜像并做了配制的自定义，通过这些镜像组合提供各种类型的postgres集群解决方案。

官方推荐解决方案
![](https://crunchydata.github.io/crunchy-containers/containers.png?raw=true)
### [stolon](https://github.com/sorintlab/stolon)
Stolon是专门用于部署在云上的PostgresSql高可用集群，与Kubernetes集成较好。

架构图：

![架构图](https://blog.lwolf.org/img/2017/03/architecture_small.png)

## 选择
主要在stolon和Crunchy对比：

stolon

* 高可用做的很好，故障转移比较靠谱；
* 并不支持读写分离；
* 存储是在etcd或者consul，未做性能测试不知道效果怎么样；

Cruuchy

* 可以自定义集群的方案，比较灵活；
* 通过[pgpool](http://www.pgpool.net/mediawiki/index.php/Main_Page)进行读写分离
* 高可用方案依赖于k8s，Master如启动多个Pod需要采用 Access Modes 为 ReadWriteMany 的Volume，支持ReadWriteMany的Volume类型并不多而且多半是网盘性能指标都不高，再者就依赖k8s对pod的故障转移了；
* 其它备份、恢复和管理都提供了相应的组件，可以选择性安装
* 对各类组件都自定义的docker镜像有耦合问题

对比后Cruuchy更加满足我的需求，Master的高可用问题今后我也可以再找更加好的解决方案替换掉Master的部署（或者Cruuchy本身就有解决方案只是我还不会....）
# Cruuchy 部署
相关部署都是阿里云的k8s集群上，我的集群部署请参考[阿里云 k8s 1.10 版本安装
](https://jamesdeng.github.io/k8s/2018/06/01/k8s-1.10-%E9%98%BF%E9%87%8C%E4%BA%91%E5%AE%89%E8%A3%85.html)

整个安装做了一个[Helm chart](https://github.com/jamesDeng/helm-charts/tree/master/pgsql-primary-replica),直接使用命令安装
``` bash
git clone https://github.com/jamesDeng/helm-charts.git

cd pgsql-primary-replica

helm install ./ --name pgsql-primary-replica --set primary.pv.volumeId=xxxx
```
primary.pv.volumeId 为阿里云盘ID

## 说明
组成结构还不完整，其它部分还需要慢慢完善，目前为：

* pgpool-II deployment
* primary deployment
* replica statefulset

create the following in your Kubernetes cluster:

 * Create a service named *primary*
 * Create a service named *replica*
 * Create a service named *pgpool*
 * Create a deployment named *primary*
 * Create a statefulset named *replica*
 * Create a deployment named *pgpool*
 * Create a persistent volume named *primary-pv* and a persistent volume claim of *primary-pvc*
 * Create a secret pgprimary-secret
 * Create a secret pgpool-secrets
 * Create a configMap pgpool-pgconf

## 自定义配制文件
根目录下：

* pool-configs
    - pgpool.conf
    - pool_hba.conf
    - pool_passwd
* primary-configs
    - pg_hba.conf
    - postgresql.conf
    - setup.sql

如果使用repo的方式安装，由于没有找到helm install替换这些文件的方式，目前有两个解决方案：

1. install完成后手动修改相应的configMap和secret的内容
2. helm fetch pgsql-primary-replica --untar 下源文件，修改相应的配件文件，采用 helm install ./ 的方式安装

## postgres 用户名和密码修改
如果是通过pgpool连接数据库的话，必须在pool_passwd配制postgres的用户名和密码
### 部署前
可以设置默认用户postgres和PG_USER对应用户的密码

1. 修改values.yaml中PG_ROOT_PASSWORD或者PG_PASSWORD的值，
2. 使用pgpool的pg_md5 指令生成密码（pg_md5 --md5auth --username= ）
3. 修改pool-configs/pool_passwd中的用户名和密码

### 部署后修改
先获取postgres的用户名和md5密码
``` bash
$ kubectl get pod -n thinker-production
NAME                       READY     STATUS    RESTARTS   AGE
pgpool-67987c5c67-gjr5h    1/1       Running   0          3d
primary-6f9f488df8-6xjjd   1/1       Running   0          3d
replica-0                  1/1       Running   0          3d

# 进入primary pod
$ kubectl exec -it primary-6f9f488df8-6xjjd  -n thinker-production  sh

$ psql

#执行sql查找md5
$ select usename,passwd from pg_shadow;  
   usename   |               passwd                
-------------+-------------------------------------
 postgres    | md5dc5aac68d3de3d518e49660904174d0c
 primaryuser | md576dae4ce31558c16fe845e46d66a403c
 testuser    | md599e8713364988502fa6189781bcf648f
(3 rows)
```
pool_passwd 的格式为
``` bash
postgres:md5dc5aac68d3de3d518e49660904174d0c
```
k8s secret 的values为bash46,转换
``` bash
$ echo -n 'postgres:md5dc5aac68d3de3d518e49660904174d0c' | base64
cG9zdGdyZXM6bWQ1ZGM1YWFjNjhkM2RlM2Q1MThlNDk2NjA5MDQxNzRkMGM=
```
编辑 pgpool-secrets 替换 pool_passwd 值
``` bash
kubectl edit secret pgpool-secrets  -n thinker-production
```
编辑 pgpool deploy,触发滚动更新
``` bash
kubectl edit deploy pgpool  -n thinker-production
# 一般可以修改一下 terminationGracePeriodSeconds 的值
```
## 自定义Values.yaml
Values.yaml的值请参考源码
``` bash
helm install ./ --name pgsql-primary-replica -f values.yaml
```
参考文档
--------------
* [https://crunchydata.github.io/crunchy-containers/](https://crunchydata.github.io/crunchy-containers/)
* [http://www.pgpool.net/docs/pgpool-II-3.5.4/doc/pgpool-zh_cn.html](http://www.pgpool.net/docs/pgpool-II-3.5.4/doc/pgpool-zh_cn.html)
* [https://blog.lwolf.org/post/how-to-deploy-ha-postgressql-cluster-on-kubernetes/](https://blog.lwolf.org/post/how-to-deploy-ha-postgressql-cluster-on-kubernetes/)