---
layout: article
key: k8s_prometheus_operator
tags: k8s prometheus-operator
comment: true
modify_date: 2018-8-23 10:30:00
---
# Prometheus Operator 介绍、安装、使用
Prometheus 为k8s 1.11以后推荐的集群监控平台
## 安装
``` bash


helm repo add coreos https://s3-eu-west-1.amazonaws.com/coreos-charts/stable/
helm install coreos/prometheus-operator --name prometheus-operator --namespace monitoring
#设置安装 CoreDNS
cat>kube-prometheus-values.yaml<<EOF
# Select Deployed DNS Solution
deployCoreDNS: true
deployKubeDNS: false
deployKubeEtcd: true
~
EOF

helm install coreos/kube-prometheus --name kube-prometheus --namespace monitoring -f kube-prometheus-values.yaml

# 添加 grafana 的 ingress
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
# 修改 deployments kube-prometheus-exporter-kube-state 的images地址前缀为
registry.cn-hangzhou.aliyuncs.com/google_containers
```

参考文档
----------
* [https://github.com/coreos/prometheus-operator/tree/master/helm/kube-prometheus](https://github.com/coreos/prometheus-operator/tree/master/helm/kube-prometheus)