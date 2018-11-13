---
layout: article
key: gitlab_cicd
tags: gitlab CI/CD 
comment: true
modify_date: 2018-9-10 14:30:00
---
在devops开发流程中，ci/cd是很非常重要的步骤，它赋予了我们快速迭代的可能:

* ci(持续构建) 代码提交后触发自动化的单元测试，代码预编译，构建镜像，上传镜像等．
* cd(持续发布) 持续发布则指将构建好的程序发布到各种环境，如预发布环境，正式环境．

GitLab CI/CD 通过在项目内 .gitlab-ci.yaml 配置文件读取 CI 任务并进行相应处理；GitLab CI 通过其称为 GitLab Runner 的 Agent 端进行 build 操作，结构如下：
![](https://docs.gitlab.com/ee/ci/img/cicd_pipeline_infograph.png)

## GitLab Runner
gitlab 的CI/CD任务需要GitLab Runner运维，Runner可以是虚拟机、VPS、裸机、docker容器、甚至一堆容器。GitLab和Runners通过API通信，所以唯一的要求就是运行Runners的机器可以联网，，参考[官方文档](https://docs.gitlab.com/runner/install/linux-repository.html)进行安装
``` bash
#机器为 CentOS 7.4
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash

yum install gitlab-runner

gitlab-runner status
```
Runner 通过注册的方式添加需要服务的Project、Group或者整个gitlab,在Project/Group->CI/CD->Runners settings查找需要的registration token
![](https://raw.githubusercontent.com/jamesDeng/jamesDeng.github.io/master/images/gitlab-cicd/runners-settings.png)
执行[官方文档](https://docs.gitlab.com/runner/register/index.html)的设置步骤
``` bash
$ gitlab-runner register

Running in system-mode.                            
                                                   
Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
# 输入 URL http://gitlab.domain.com/
Please enter the gitlab-ci token for this runner:
#输入 registration token
Please enter the gitlab-ci description for this runner:
#输入描述
Please enter the gitlab-ci tags for this runner (comma separated):
#输入tags
Registering runner... succeeded                     runner=9jHBE4P6
Please enter the executor: ssh, virtualbox, docker+machine, docker-ssh+machine, docker, docker-ssh, parallels, shell, kubernetes:
#输入执行类型，我输入的是 docker
Please enter the default Docker image (e.g. ruby:2.1):
#默认运行的镜像
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!

#查看注册
$ gitlab-runner list
```
在 Project/Group->CI/CD->Runners settings 查看是否可以使用
![](https://raw.githubusercontent.com/jamesDeng/jamesDeng.github.io/master/images/gitlab-cicd/runners-registration.png)
## .gitlab-ci.yml
.gitlab-ci.yml是用来配置CI在我们的项目中做些什么工作，它位于项目的根目录。

语法参考[官方文档的翻译](https://segmentfault.com/a/1190000010442764#articleHeader9)

我司项目目前有 java,react,android,ios 类型项目，该篇介绍 java
### java build
项目使用maven构建，发布需要打包成docker images,采用[dockerfile-maven-plugin](https://jamesdeng.github.io/2018/08/15/%E6%9C%8D%E5%8A%A1docker%E5%8C%96.html)方式做 docker images bulid and push
``` yaml
image: maven:3.5.4-jdk-8

# 注意使用 docker:dind 需要设置 /etc/gitlab-runner/config.toml 的 privileged = true，
services:
  - docker:dind

variables:
  # This will supress any download for dependencies and plugins or upload messages which would clutter the console log.
  # `showDateTime` will show the passed time in milliseconds. You need to specify `--batch-mode` to make this work.
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=WARN -Dorg.slf4j.simpleLogger.showDateTime=true -Djava.awt.headless=true"
  # As of Maven 3.3.0 instead of this you may define these options in `.mvn/maven.config` so the same config is used
  # when running from the command line.
  # `installAtEnd` and `deployAtEnd` are only effective with recent version of the corresponding plugins.
  MAVEN_CLI_OPTS: "--batch-mode --errors --fail-at-end --show-version -DinstallAtEnd=true -DdeployAtEnd=true"
  #用于支持 docker bulid
  DOCKER_HOST: tcp://docker:2375
  DOCKER_DRIVER: overlay2

# Cache downloaded dependencies and plugins between builds.
# To keep cache across branches add 'key: "$CI_JOB_NAME"'
cache:
  paths:
    - .m2/repository

stages:
  - build
  - deploy

# Validate JDK8
validate:jdk8:
  script:
    - 'mvn $MAVEN_CLI_OPTS test-compile'
  only:
    - dev-4.0.1
  stage: build

# 目前只做到打包
deploy:jdk8:
  # Use stage test here, so the pages job may later pickup the created site.
  stage: deploy
  script:
    - 'mvn $MAVEN_CLI_OPTS deploy'
  only:
    - dev-4.0.1

```
注意使用docker:dind 需要在 docker run 加入 --privileged，Runner 设置方式为修改 /etc/gitlab-runner/config.toml 的 privileged = true
``` toml
concurrent = 1
check_interval = 0

[[runners]]
  name = "kms"
  url = "http://example.org/ci"
  token = "3234234234234"
  executor = "docker"
  [runners.docker]
    tls_verify = false
    image = "alpine:3.4"
    privileged = true
    disable_cache = false
    volumes = ["/cache"]
  [runners.cache]
    Insecure = false

```

#### dockerfile-maven-plugin push 支持
dockerfile-maven-plugin 的 push 需要在 maven settings.xml 文件中设置 docker 的用户名和密码，这里需要重做 maven images ,为了安全还需要对 settings.xml 中的密码进行加密，并且 dockerfile-maven-plugin 高于 1.4.3 才支持这个设置

进行 settings-security.xml settings.xml 文件准备
``` bash
#生成 master 密码
$ mvn --encrypt-master-password 123456
{40hwd7X9T1wHfRTKWlUIbHyjacbV4iV/hSfODlRcH/E=}

$ cat>settings-security.xml<<EOF
<settingsSecurity>
  <master>{40hwd7X9T1wHfRTKWlUIbHyjacbV4iV/hSfODlRcH/E=}</master>
</settingsSecurity>
EOF

#生成 docker 密码 
$ mvn --encrypt-password 123456
{QdP5NFvxYhYHILE6k8tDEff+CZzWq2N3mCkPPUcljSA=}

#修改 settings.xml 中 docker 密码
<server>
  <id>registry.cn-shenzhen.aliyuncs.com</id>
  <username>username</username>
  <password>{QdP5NFvxYhYHILE6k8tDEff+CZzWq2N3mCkPPUcljSA=}</password>
</server>
```
完成 dockerfile
``` yaml
FROM maven:3.5.4-jdk-8

COPY settings.xml /root/.m2/

COPY settings-security.xml /root/.m2/
```
docker build 成自己的 maven images,并修改 .gitlab-ci.yml 文件中使用的 images
#### 私有 docker images 支持
如果 docker images 是需要login的，先生成 username:password 的 base64 值
``` bash
echo -n "my_username:my_password" | base64

# Example output to copy
bXlfdXNlcm5hbWU6bXlfcGFzc3dvcmQ=
```
在 Project/Group->CI/CD->Variable添加 DOCKER_AUTH_CONFIG，内容如下：
``` json
{
    "auths": {
        "registry.example.com": {
            "auth": "bXlfdXNlcm5hbWU6bXlfcGFzc3dvcmQ="
        }
    }
}
```
### k8s helm deploy
公司项目基于 k8s helm 进行发布，这里介绍怎样基于 k8s helm 做 CD

首先需要做一个 kubectl and helm 的镜像
``` yaml
FROM alpine:3.4

#注意 kubectl 和 helm 的版本
ENV KUBE_LATEST_VERSION=v1.10.7
ENV HELM_VERSION=v2.9.1
ENV HELM_FILENAME=helm-${HELM_VERSION}-linux-amd64.tar.gz

RUN apk add --update ca-certificates \
 && apk add --update -t deps curl  \
 && apk add --update gettext tar gzip \
 && curl -L https://storage.googleapis.com/kubernetes-release/release/${KUBE_LATEST_VERSION}/bin/linux/amd64/kubectl -o /usr/local/bin/kubectl \
 && curl -L https://storage.googleapis.com/kubernetes-helm/${HELM_FILENAME} | tar xz && mv linux-amd64/helm /bin/helm && rm -rf linux-amd64 \
 && chmod +x /usr/local/bin/kubectl \
 && apk del --purge deps \
 && rm /var/cache/apk/*
```
运行 build 需要使用代理解决墙的问题
``` bash
#HTTP_PROXY=http://192.168.1.22:1087 为 vpn 代理
docker build --build-arg HTTP_PROXY=http://192.168.1.22:1087 -t helm-kubectl:v2.9.1-1.10.7 .
```
kubectl 执行指令需要 k8s api 的config,找一台 k8s master 执行指令将 config 转换成 base64，创建 gitlab ci/cd 的 “kube_config” variable 值为 config base64
``` bash
cat ~/.kube/config | base64 
```
加入 deploy-test job 用于发布测试环境,考虑目前开发模式下测试环境不会更新版本号，在 deploy.yaml 的 spec/template/metadata/annotations 下添加 update/version 注解,并在 chart 配制 values
``` yaml
spec:
  selector:
    matchLabels:
      app: admin
  replicas: 1
  revisionHistoryLimit: 3
  template:
    metadata:
      labels:
        app: admin
      annotations:
        update/version : "{{ .Values.admin.updateVersion }}"
```
在 job 使用 helm upgrade --set admin.updateVersion= 可以达到重新部署测试环境的目的
``` yaml
deploy-test:
  # Use stage test here, so the pages job may later pickup the created site.
  stage: deploy-test
  image: registry.cn-shenzhen.aliyuncs.com/thinker-open/helm-kubectl:v2.9.1-1.10.7
  before_script:
    - mkdir -p /etc/deploy
    - echo ${kube_config} | base64 -d > ${KUBECONFIG}
    - kubectl config use-context kubernetes-admin@kubernetes
    - helm init --client-only --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
    - helm repo add my-repo http://helm.domain.com/
    - helm repo update
  script:
    - helm upgrade cabbage-test --reuse-values --set admin.updateVersion=`date +%s` thinker/cabbage
  environment:
    name: test
    url: https://test.example.com
  only:
  - dev-4.0.1
```
加入 deploy-prod job 用于发布生产环境，拉 tag 时执行，使用 tag name 更新 image 的版本 
``` yaml
deploy-prod:
  # Use stage test here, so the pages job may later pickup the created site.
  stage: deploy-prod
  image: registry.cn-shenzhen.aliyuncs.com/thinker-open/helm-kubectl:v2.9.1-1.10.7
  before_script:
    - mkdir -p /etc/deploy
    #使用 kube_config variable
    - echo ${kube_config} | base64 -d > /etc/deploy/config
    #注意 kubernetesContextName 为 ~/.kube/config 文件中的 context name
    - kubectl config use-context kubernetesContextName
    #墙问题使用阿里云源
    - helm init --client-only --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
    #加入需要的 helm repo 
    - helm repo add my-repo http://helm.domain.com/
    - helm repo update
  script:
    # 这里只做 upgrade,CI_COMMIT_REF_NAME 是  branch or tag name 
    - helm upgrade cabbage-test --reuse-values --set admin.image.tag=${CI_COMMIT_REF_NAME} thinker/cabbage
  environment:
    name: test
    url: https://prod.example.com
  only:
  - tags
```
完整例子
``` yaml
image: maven:3.5.4-jdk-8

services:
  - docker:dind

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=WARN -Dorg.slf4j.simpleLogger.showDateTime=true -Djava.awt.headless=true"
  MAVEN_CLI_OPTS: "--batch-mode --errors --fail-at-end --show-version -DinstallAtEnd=true -DdeployAtEnd=true"
  DOCKER_HOST: tcp://docker:2375
  DOCKER_DRIVER: overlay2
  KUBECONFIG: /etc/deploy/config

# Cache downloaded dependencies and plugins between builds.
# To keep cache across branches add 'key: "$CI_JOB_NAME"'
cache:
  paths:
    - .m2/repository

stages:
  - build 
  - deploy-test
  - deploy-prod

# mvn deploy,生成 images 并 push 到阿里云
build:jdk8:
  script:
    - 'mvn $MAVEN_CLI_OPTS deploy'
  only:
    - dev-4.0.1
  stage: build

deploy-test:
  # Use stage test here, so the pages job may later pickup the created site.
  stage: deploy-test
  image: registry.cn-shenzhen.aliyuncs.com/thinker-open/helm-kubectl:v2.9.1-1.10.7
  before_script:
    - mkdir -p /etc/deploy
    - echo ${kube_config} | base64 -d > ${KUBECONFIG}
    - kubectl config use-context kubernetes-admin@kubernetes
    - helm init --client-only --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
    - helm repo add my-repo http://helm.domain.com/
    - helm repo update
  script:
    - helm upgrade cabbage-test --reuse-values --set admin.updateVersion=`date +%s` thinker/cabbage
  environment:
    name: test
    url: https://test.example.com
  only:
  - dev-4.0.1

deploy-prod:
  # Use stage test here, so the pages job may later pickup the created site.
  stage: deploy-prod
  image: registry.cn-shenzhen.aliyuncs.com/thinker-open/helm-kubectl:v2.9.1-1.10.7
  before_script:
    - mkdir -p /etc/deploy
    - echo ${kube_config} | base64 -d > ${KUBECONFIG}
    - kubectl config use-context kubernetes-admin@kubernetes
    - helm init --client-only --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
    - helm repo add my-repo http://helm.domain.com/
    - helm repo update
  script:
   #CI_COMMIT_REF_NAME 是  branch or tag name 
    - helm upgrade cabbage-test --reuse-values --set admin.image.tag=${CI_COMMIT_REF_NAME} thinker/cabbage
  environment:
    name: test
    url: https://prod.example.com
  only:
  - tags

```
#### 总结
采用 helm 有几个问题：

*  项目第一次 install 应该由运维统一执行，如果放在 job 里做怎样处理配制文件是个问题
*  如果迭代过程中出现在配制文件项的添加和更新要怎么处理
*  helm chart 文件的编写与项目开发怎么协调

参考文档
----------
* [https://docs.gitlab.com/ee/ci/docker/using_docker_build.html](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html)
* [https://about.gitlab.com/2017/09/21/how-to-create-ci-cd-pipeline-with-autodeploy-to-kubernetes-using-gitlab-and-helm/](https://about.gitlab.com/2017/09/21/how-to-create-ci-cd-pipeline-with-autodeploy-to-kubernetes-using-gitlab-and-helm/)