---
layout: article
key: helm_private_resp
tags: k8s helm
comment: true
modify_date: 2018-8-7 10:30:00
---
# Helm 私服
对于准备用Helm来安装与管理线上服务的公司，搭建一个公司内部的Helm Chart Repositry是必不可少的，本例使用[ChartMuseum](https://github.com/helm/chartmuseum)来搭建。
## 本地安装
### 安装
``` bash
# on Linux
curl -LO https://s3.amazonaws.com/chartmuseum/release/latest/bin/linux/amd64/chartmuseum

# on macOS
curl -LO https://s3.amazonaws.com/chartmuseum/release/latest/bin/darwin/amd64/chartmuseum

# on Windows
curl -LO https://s3.amazonaws.com/chartmuseum/release/latest/bin/windows/amd64/chartmuseum

chmod +x ./chartmuseum
mv ./chartmuseum /usr/local/bin
```
### 启动
ChartMuseum 启动方式主要在于对存储的选择，ChartMuseum支持很多种存储方式:本地、Amazon S3、Google Cloud Storage、Alibaba Cloud OSS Storage 等等，这里我选择了阿里OSS。

在阿里云开通OSS并新建存储空间 my-oss-bucket，新建子用户并授与AliyunOSSFullAccess权限，拿到新建用户的AccessKey And Secret

写入AccessKey And Secret到环境变量
``` bash
export ALIBABA_CLOUD_ACCESS_KEY_ID=xxx
export ALIBABA_CLOUD_ACCESS_KEY_SECRET=xxx
```
执行启动命令
``` bash
chartmuseum --debug --port=8080 \
  --storage="alibaba" \
  --storage-alibaba-bucket="my-oss-bucket" \
  --storage-alibaba-prefix="" \
  --storage-alibaba-endpoint="oss-cn-beijing.aliyuncs.com"
```
## 使用
本地添加repository
``` bash
helm repo add my-repository http://localhost:8080
```
查找该repository下的charts
``` bash
helm search my-repository/
```
安装 chart
``` bash
helm install my-repository/mychart
```
## 上传Chart
github上有介绍curl命令方式上传这里略过，直接介绍通过[helm-push](https://github.com/chartmuseum/helm-push)插件方式上传
### helm-push
helm-push是helm命令的插件，安装过后就可以直接执行helm push
``` bash
helm plugin install https://github.com/chartmuseum/helm-push
```
### 上传
``` bash
helm push mychart/ my-repository
```
### 权限控制
支持的权限控制有Basic Auth、Token，这里选择Basic Auth

首先启动命令要添加AUTH_ANONYMOUS_GET=true，并将用户名和密码设置在环境变量中
``` bash
export HELM_REPO_USERNAME="myuser"
export HELM_REPO_PASSWORD="mypass"
chartmuseum --debug --port=8080 \
  --storage="alibaba" \
  --storage-alibaba-bucket="my-oss-bucket" \
  --storage-alibaba-prefix="" \
  --storage-alibaba-endpoint="oss-cn-beijing.aliyuncs.com" \
  --AUTH_ANONYMOUS_GET=true
```
在repo url中添加username/password，如：http://myuser:mypass@my.chart.repo.com
## 其它安装方式
### docker
``` bash
docker run --rm -it \
  -p 8080:8080 \
  -e PORT=8080 \
  -e DEBUG=1 \
  -e STORAGE="alibaba" \
  -e STORAGE_ALIBABA_BUCKET="my-oss-bucket" \
  -e STORAGE_ALIBABA_PREFIX="" \
  -e STORAGE_ALIBABA_ENDPOINT="oss-cn-beijing.aliyuncs.com" \
  -e ALIBABA_CLOUD_ACCESS_KEY_ID="xxx" \
  -e ALIBABA_CLOUD_ACCESS_KEY_SECRET="xxx" \
  chartmuseum/chartmuseum:latest
```
### Helm Chart
既然我们使用k8s和Helm,用该方式安装肯定是极好的，[如果不会用可以查看一下人家的Chart怎么写的](https://github.com/kubernetes/charts/tree/master/stable/chartmuseum)

先写一个custom.yaml用来配制chart
``` yaml
env:
  open:
    #开启api
    DISABLE_API: false
    ALLOW_OVERWRITE: true
    STORAGE: alibaba
    STORAGE_ALIBABA_BUCKET: my-oss-bucket
    STORAGE_ALIBABA_PREFIX:
    STORAGE_ALIBABA_ENDPOINT: oss-cn-shenzhen-internal.aliyuncs.com
  secret:
    ALIBABA_CLOUD_ACCESS_KEY_ID: "xxx" ## alibaba OSS access key id
    ALIBABA_CLOUD_ACCESS_KEY_SECRET: "xxx" ## alibaba OSS access key secret
    BASIC_AUTH_USER: myuser
    BASIC_AUTH_PASS: mypass
ingress:
#开启 ingress
  enabled: true
## Chartmuseum Ingress annotations
##
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"

## Chartmuseum Ingress hostnames
## Must be provided if Ingress is enabled
##
  hosts:
    #访问的域名
    helm.xxx.xxx:
        - /charts
        - /api/charts
        - /api/prov
        - /index.yaml
```
执行安装
``` bash
helm repo add stable https://kubernetes-charts.storage.googleapis.com
helm install -f custom.yaml stable/chartmuseum 
```

参考文档
===
[官网](https://github.com/helm/chartmuseum)