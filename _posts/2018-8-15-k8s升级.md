---
layout: article
key: k8s_upgrade
tags: k8s
comment: true
modify_date: 2018-8-15 10:30:00
---
# k8s 升级
目前公司k8s集群是1.11.1版本，尝试了升级到1.11.2，为今后升级探个道
## 基本情况
公司目前集群部署在阿里云上是3主2从,升级根据[官方文档](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade-1-11/)
## 升级相关控制组件
任意一点Master上安装新版Kubeadm
``` bash
export VERSION=$(curl -sSL https://dl.k8s.io/release/stable.txt) # or manually specify a released Kubernetes version
export ARCH=amd64 # or: arm, arm64, ppc64le, s390x
curl -sSL https://dl.k8s.io/release/${VERSION}/bin/linux/${ARCH}/kubeadm > /usr/bin/kubeadm
chmod a+rx /usr/bin/kubeadm
```
执行命令查看一下各组件当前版本与升级版本
``` bash
kubeadm upgrade plan
```
确认无误执行升级,注意升级的版本号
``` bash
kubeadm upgrade apply v1.11.2
```
## 升级Master and node安装包
新版本只需要升级kubelet and kubeadm,该操作需要在所有的节点上执行

排除节点，使节点不参加调度
``` bash
如果nodename是hostname就可以直接执行以下
export nodename=`hostname`
kubectl drain $nodename --ignore-daemonsets
```
执行升级
``` bash
yum upgrade -y kubelet kubeadm --disableexcludes=kubernetes
```
如遇墙的问题，可以使用阿里镜像升级
``` bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
## 重启与配制kubelet
所有node都需要执行下面的步骤

如果是从节点，需要升级kubelet config:
``` bash
这一步Master node是不需要执行的
sudo kubeadm upgrade node config --kubelet-version $(kubelet --version | cut -d ' ' -f 2)
```
由于升级kubelte后，/etc/sysconfig/kubelet 文件被清空，如果有相关配制需要重新写入/etc/sysconfig/kubelet
``` bash
各家配制不一样，注意替换配制内容
cat <<EOF > /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--enable-controller-attach-detach=false --cgroup-driver=cgroupfs --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.1"
EOF
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```
检查一下kubelet状态
``` bash
systemctl status kubelet
```
确认kubelet状态正常后，重新加入node
``` bash
kubectl uncordon $nodename
```
最后确认节点状态
``` bash
kubectl get nodes
```