---
layout: article
key: minio_thumbor
tags: minio thumbor
comment: true
modify_date: 2018-11-13 14:30:00
---
公司一直使用 [seaweedfs](https://github.com/) 做文件存储与图片服务器，seaweedfs 很好用且性能不错，但是缺乏文件管理方案、大文件存储、对k8s的支持，为了满足以上需求我准备用OSS替换现有的文件存储方案，[MINIO](https://docs.minio.io/) 是一个基于Apache License v2.0开源协议的对象存储服务。它兼容亚马逊S3云存储服务接口，非常适合于存储大容量非结构化的数据，例如图片、视频、日志文件、备份数据和容器/虚拟机镜像等，而一个对象文件可以是任意大小，从几kb到最大5T不等。

MINIO 本身并不支持图片处理，需要 [Thumbor](https://thumbor.readthedocs.io/en/latest/index.html) 这类型的图片处理服务配合才能构成一个图片服务器，
Thumbor 是一个小型的图片处理服务，具有缩放、裁剪、翻转、滤镜等功能。

# MINIO
Minio 的安装和使用都非常简单，提供完善各平台安装示例与JAVA、JavaScript、Python、Golang、.NET SDK文档，有完整的中文文档对我这种英文渣非常友好，而且它是GO语言写的，正好我有学习GO的计划有空还可以研究一下源码
## 安装
k8s 安装官方提供了 [Helm Chart](https://docs.minio.io/cn/deploy-minio-on-kubernetes.html) ，配制 value.yaml
``` yaml
image:
  repository: minio/minio
  tag: RELEASE.2018-10-18T00-28-58Z
  pullPolicy: IfNotPresent

mode: standalone

accessKey: "admin"
secretKey: "123456"
configPath: "/root/.minio/"
mountPath: "/export"

persistence:
  enabled: true

  ## A manually managed Persistent Volume and Claim
  ## Requires persistence.enabled: true
  ## If defined, PVC must be created manually before volume will be bound
  # existingClaim:

  ## minio data Persistent Volume Storage Class
  ## If defined, storageClassName: <storageClass>
  ## If set to "-", storageClassName: "", which disables dynamic provisioning
  ## If undefined (the default) or set to null, no storageClassName spec is
  ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
  ##   GKE, AWS & OpenStack)
  ##
  ## Storage class of PV to bind. By default it looks for standard storage class.
  ## If the PV uses a different storage class, specify that here.
  storageClass: minio
  accessMode: ReadWriteOnce
  size: 20Gi

  ## If subPath is set mount a sub folder of a volume instead of the root of the volume.
  ## This is especially handy for volume plugins that don't natively support sub mounting (like glusterfs).
  ##
  subPath: ""

ingress:
  enabled: true
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  path: /
  hosts:
    - minio.domain.vc
  tls: 
  - secretName: minio-oss-tls
    hosts:
      - minio.domain.vc

## Node labels for pod assignment
## Ref: https://kubernetes.io/docs/user-guide/node-selection/
##
nodeSelector: {}

resources:
  requests:
    memory: 256Mi
    cpu: 250m

```
执行安装命令
``` bash
helm install --name my-release -f values.yaml stable/minio
```
Helm 官方 Chart 库存在墙的问题可能无法安装，可以去 Github 把源码 clone 下来使用 [https://github.com/helm/charts/tree/master/stable/minio](https://github.com/helm/charts/tree/master/stable/minio)

Ingress Https 参考 [https://jamesdeng.github.io/2018/09/17/K8s-cert-manager.html](https://jamesdeng.github.io/2018/09/17/K8s-cert-manager.html) 进行处理

## 使用
官网例子：
``` java
import java.io.IOException;
import java.security.NoSuchAlgorithmException;
import java.security.InvalidKeyException;

import org.xmlpull.v1.XmlPullParserException;

import io.minio.MinioClient;
import io.minio.errors.MinioException;

public class FileUploader {
  public static void main(String[] args) throws NoSuchAlgorithmException, IOException, InvalidKeyException, XmlPullParserException {
    try {
      // 使用Minio服务的URL，端口，Access key和Secret key创建一个MinioClient对象
      MinioClient minioClient = new MinioClient("https://play.minio.io:9000", "Q3AM3UQ867SPQQA43P2F", "zuf+tfteSlswRu7BJ86wekitnifILbZam1KYY3TG");

      // 检查存储桶是否已经存在
      boolean isExist = minioClient.bucketExists("asiatrip");
      if(isExist) {
        System.out.println("Bucket already exists.");
      } else {
        // 创建一个名为asiatrip的存储桶，用于存储照片的zip文件。
        minioClient.makeBucket("asiatrip");
      }

      // 使用putObject上传一个文件到存储桶中。
      minioClient.putObject("asiatrip","asiaphotos.zip", "/home/user/Photos/asiaphotos.zip");
      System.out.println("/home/user/Photos/asiaphotos.zip is successfully uploaded as asiaphotos.zip to `asiatrip` bucket.");
    } catch(MinioException e) {
      System.out.println("Error occurred: " + e);
    }
  }
}
```
# Thumbor
## 安装
Github 上的 Thumbor Docker Image 都没有最新6.6.0版本的，我 fork 一个项目改了一下 [https://github.com/jamesDeng/thumbor](https://github.com/jamesDeng/thumbor)

使用 Docker-compose examples
``` yaml
version: '3'
services:
  thumbor:
    image: registry.cn-shenzhen.aliyuncs.com/thinker-open/thumbor:6.6.0
    environment:
      #- ALLOW_UNSAFE_URL=False
      - DETECTORS=['thumbor.detectors.feature_detector','thumbor.detectors.face_detector']
      - AWS_ACCESS_KEY_ID=admin # put your AWS_ACCESS_KEY_ID here
      - AWS_SECRET_ACCESS_KEY=123456 # put your AWS_SECRET_ACCESS_KEY here
      # Is needed for buckets that demand the new signing algorithm (v4)
      # - S3_USE_SIGV4=true
      # - TC_AWS_REGION=eu-central-1
      - TC_AWS_REGION=us-east-1
      - TC_AWS_ENDPOINT=https://minio.domain.vc
      - TC_AWS_ENABLE_HTTP_LOADER=False
      - TC_AWS_ALLOWED_BUCKETS=False
      # loader
      - LOADER=tc_aws.loaders.s3_loader
      - TC_AWS_LOADER_BUCKET=test-images
      # STORAGE
      - STORAGE=tc_aws.storages.s3_storage
      - TC_AWS_STORAGE_BUCKET=thumbor-storage
      - TC_AWS_STORAGE_ROOT_PATH=storage
      #RESULT_STORAGE
      - RESULT_STORAGE=tc_aws.result_storages.s3_storage
      - TC_AWS_RESULT_STORAGE_BUCKET=thumbor-storage
      - TC_AWS_RESULT_STORAGE_ROOT_PATH=result_storage
      - RESULT_STORAGE_STORES_UNSAFE=True
      - STORAGE_EXPIRATION_SECONDS=None
      - RESULT_STORAGE_EXPIRATION_SECONDS=None
    restart: always
    ports:
      - "8002:8000" 
    volumes:
      # mounting a /data folder to store cached images
      - /Users/james/james-docker/thumbor/data:/data
    restart: always
    networks:
      - app
  nginx:
    image: registry.cn-shenzhen.aliyuncs.com/thinker-open/thumbor-nginx:1.0.1
    links:
      - thumbor:thumbor
    ports:
      - "8001:80" # nginx cache port (with failover to thumbor)
    hostname: nginx
    restart: always
    networks:
      - app
volumes:
  data:
    driver: local
networks:
  app:
    driver: bridge
```
启动
``` bash
docker-compose -f thumbor-prod.yaml up -d
```

配制中 RESULT_STORAGE 也是采用 tc_aws.result_storages.s3_storage，我看大部分文章的 RESULT_STORAGE 都是 thumbor.result_storages.file_storage，然后配制Nginx进行去读取本地存储，我是考虑是如果集群部署那么每个图片的处理结果在所有实例上都有可能存在，这样很浪费空间。

注意例子中的 TC_AWS_STORAGE_BUCKET=thumbor-storage、TC_AWS_RESULT_STORAGE_BUCKET=thumbor-storage 的 bucket 都需要手动去 Minio 创建,[安装 mc](https://docs.minio.io/cn/minio-client-quickstart-guide.html)后执行
```
mc config host add minio-oss https://minio.domain.vc admin 123456 S3v4
mc mb minio-oss/thumbor-storage
```

注意例子中的 TC_AWS_LOADER_BUCKET 必须指定，该 docker 镜像有问题并不支持在访问URL中写入Bucket的方式，我按网上方案并没有解决

## nginx 说明
对 Thumbor url 语法进行改进，去掉 unsafe，加入自定的尾部截取语法
``` conf
 server {
        listen 80 default;
        server_name localhost;
        
        # This CORS configuration will be deleted if envvar THUMBOR_ALLOW_CORS != true
        add_header 'Access-Control-Allow-Origin' '*';                                   # THUMBOR_ALLOW_CORS
        add_header 'Access-Control-Allow-Credentials' 'true';                           # THUMBOR_ALLOW_CORS
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS';    # THUMBOR_ALLOW_CORS
        add_header 'Access-Control-Allow-Headers' 'Accept,Authorization,Cache-Control,Content-Type,DNT,If-Modified-Since,Keep-Alive,Origin,User-Agent,X-Mx-ReqToken,X-Requested-With'; # THUMBOR_ALLOW_CORS
         
        location / {
            proxy_pass   http://thumbor/unsafe$request_uri;
            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }

        location ~* ^([\s\S]+)_([\d-]+x[\d-]+)$ {
            proxy_pass   http://thumbor;
            rewrite ~*/([\s\S]+)_([\d-]+x[\d-]+)$ /unsafe/$2/$1 break;
            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
        
        location = /healthcheck {
            proxy_pass http://thumbor$request_uri;
            access_log off;
        }

        location ~ /\.ht { deny  all; access_log off; error_log off; }
        location ~ /\.hg { deny  all; access_log off; error_log off; }
        location ~ /\.svn { deny  all; access_log off; error_log off; }
        location = /favicon.ico { deny  all; access_log off; error_log off; }
    }
```

## 使用
如 bucket test-images 中有图片 test1.jpg，进行裁剪

官方方式

http://localhost:8001/200x200/test1.jpg

自定的尾部截取语法

http://localhost:8001/test1.jpg_200x200

其它相关 Thumbor 语法去官网查询

## k8s 安装
写了一个 helm chart [https://github.com/jamesDeng/helm-charts/tree/master/thumbor](https://github.com/jamesDeng/helm-charts/tree/master/thumbor),配制 value.yaml 
``` yaml
# Default values for thumbor.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

thumbor:
  image:
    repository: registry.cn-shenzhen.aliyuncs.com/thinker-open/thumbor
    tag: 6.6.0
    pullPolicy: Always
  resources: 
    limits:
      cpu: 1000m
      memory: 1024Mi
    requests:
      cpu: 500m
      memory: 128Mi

nginx:
  image:
    repository: registry.cn-shenzhen.aliyuncs.com/thinker-open/thumbor-nginx
    tag: 1.0.1
    pullPolicy: Always
  resources: 
    limits:
      cpu: 500m
      memory: 256Mi
    requests:
      cpu: 100m
      memory: 128Mi

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  path: /
  hosts:
    - image.domain.vc
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

nodeSelector: {}

thumborConfig: 
  THUMBOR_PORT: 8000
  ALLOW_UNSAFE_URL: "True"
  LOG_LEVEL: "DEBUG"
  DETECTORS: "['thumbor.detectors.feature_detector','thumbor.detectors.face_detector']"
  AWS_ACCESS_KEY_ID: admin
  AWS_SECRET_ACCESS_KEY: 123456
  # aws:eu-central-1 minio:us-east-1
  TC_AWS_REGION: us-east-1  
  TC_AWS_ENDPOINT: https://minio.domain.vc
  TC_AWS_ENABLE_HTTP_LOADER: "False"
  LOADER: "tc_aws.loaders.s3_loader"
  TC_AWS_LOADER_BUCKET: test-images
  TC_AWS_LOADER_ROOT_PATH: ""
  STORAGE: "tc_aws.storages.s3_storage"
  TC_AWS_STORAGE_BUCKET: thumbor-storage
  TC_AWS_STORAGE_ROOT_PATH: storage
  RESULT_STORAGE: "tc_aws.result_storages.s3_storage"
  TC_AWS_RESULT_STORAGE_BUCKET: thumbor-storage
  TC_AWS_RESULT_STORAGE_ROOT_PATH: result_storage
  RESULT_STORAGE_STORES_UNSAFE: "True"
  STORAGE_EXPIRATION_SECONDS: None
  RESULT_STORAGE_EXPIRATION_SECONDS: None
```
clone 下来进入thumbor目录执行
```
helm install --name thumbor -f values.yaml .
```
部署成功使用 http://image.domain.vc/200x200/test1.jpg 访问

参考文档
----------
* [https://github.com/helm/charts/tree/master/stable/minio](https://github.com/helm/charts/tree/master/stable/minio)
* [https://github.com/APSL/docker-thumbor](https://github.com/APSL/docker-thumbor)
* [https://blog.frognew.com/2017/08/image-process-service-with-thumbor.html](https://blog.frognew.com/2017/08/image-process-service-with-thumbor.html)
* [https://github.com/MinimalCompact/thumbor](https://github.com/MinimalCompact/thumbor)