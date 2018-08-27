---
layout: article
key: k8s_prometheus_operator
tags: k8s prometheus-operator
comment: true
modify_date: 2018-8-23 10:30:00
---
# Prometheus Operator 介绍、安装、使用
## 介绍
Prometheus 为k8s 1.11推荐的集群监控和警报平台,Operator是CoreOS开源的一套用于维护Kubernetes上Prometheus的控制器，目标是简化部署与维护Prometheus。

架构如下：

![](https://coreos.com/sites/default/files/inline-images/p1.png)

![](https://cdn-images-1.medium.com/max/880/1*u3QBSePU0ogIOiAwuBKlow.png)

主要分为五个部分：

* Operate: 系统主要控制器，根据自定义的资源（Custom Resource Definition,CRDs）来负责管理与部署；
* Prometheus Server: 由Operator 依据一个自定义资源Prometheus类型中所描述的内容而部署的Prometheus Server集，可以将这个自定义资源看作是一种特别用来管理Prometheus Server的StatefulSet资源；
* ServiceMonitor: 一个Kubernetes自定义资料，该资源描述了Prometheus Server的Target列表，Operator会监听这个资源的变化来动态更新 Prometheus Server的Scrape targets。而该资源主要透过 Selector 来依据 Labels 选取对应的 Service Endpoint,并让 Prometheus Serve 透过 Service 进行拉取 Metrics 资料；
* Service: kubernetes 中的 Service 资源，这边主要用来对应 Kubernetes 中 Metrics Server Pod,然后提供给 ServiceMonitor 选取让 Prometheus Server 拉取资料，在 Prometheus 术语中可以称为 Target,即被 Prometheus 监测的对象，如一個部署在 Kubernetes 上的 Node Exporter Service。
* Alertmanager: 接收从 Prometheus 来的 event,再根据定义的 notification 组决定要通知的方法。
## 安装
采用 helm chart 方式安装
``` bash


helm repo add coreos https://s3-eu-west-1.amazonaws.com/coreos-charts/stable/
helm install coreos/prometheus-operator --name prometheus-operator --namespace monitoring
#设置安装 CoreDNS
cat>kube-prometheus-values.yaml<<EOF
# Select Deployed DNS Solution
deployCoreDNS: true
deployKubeDNS: false
deployKubeEtcd: true
EOF

helm install coreos/kube-prometheus --name kube-prometheus --namespace monitoring -f kube-prometheus-values.yaml

# 采用域名方式访问，添加 grafana 的 ingress
cat>grafana-ingress.yaml<<EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kube-prometheus-grafana
  namespace: monitoring
spec:
  rules:
    - host: grafana.domain.com
      http:
        paths:
          - path: /
            backend:
              serviceName: kube-prometheus-grafana
              servicePort: 80
EOF
kubectl apply -f grafana-ingress.yaml

#deployments kube-prometheus-exporter-kube-state 的images无法下载，改为国内镜像
#将 images 地址前缀为 registry.cn-hangzhou.aliyuncs.com/google_containers
kubectl edit deploy kube-prometheus-exporter-kube-state -n monitoring

#关闭 grafana 匿名身份验证
kubectl edit deploy kube-prometheus-grafana -n monitoring
#将env GF_AUTH_ANONYMOUS_ENABLED改到false
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "false" 
```
## 使用
访问 grafana.domain.com，用户密码都为 admin

参考文档
----------
* [https://github.com/coreos/prometheus-operator/tree/master/helm/kube-prometheus](https://github.com/coreos/prometheus-operator/tree/master/helm/kube-prometheus)
* [https://medium.com/getamis/kubernetes-operators-prometheus-3584edd72275](https://medium.com/getamis/kubernetes-operators-prometheus-3584edd72275)
* [https://coreos.com/operators/prometheus/docs/latest/user-guides/getting-started.html](https://coreos.com/operators/prometheus/docs/latest/user-guides/getting-started.html)