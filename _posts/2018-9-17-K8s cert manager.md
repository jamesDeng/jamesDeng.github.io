---
layout: article
key: K8s_cert_manager
tags: k8s cert-manager 
comment: true
modify_date: 2018-9-17 14:30:00
---
cert-manager 是k8s的本地证书管理控制器，它可以自动注册和续签 Let’s Encrypt, HashiCorp Vault 的https证书
![](https://cert-manager.readthedocs.io/en/latest/_images/high-level-overview.png) 
## 安装
使用 helm 安装，这里可以查找[charts repository](https://github.com/helm/charts/tree/master/stable/cert-manager)
```
helm install \
    --name cert-manager \
    --namespace kube-system \
    stable/cert-manager
```

有墙的问题，建议把源码下载后安装
```
git clone https://github.com/helm/charts.git
cd charts/stable/cert-manager
helm install \
    --name cert-manager \
    --namespace kube-system \
    ./
```
## 使用
### 设置 Issuer
先添加用于设定从那里取得证书的 Issuer，k8s 提供了Issuer和ClusterIssuer两种，前者在单个namespace使用，后者是可以在整个集里中使用的
``` bash
cat>issuer.yaml<<EOF
apiVersion: certmanager.k8s.io/v1alpha1 
# Issuer/ClusterIssuer
kind: Issuer 
metadata:   
    name: cert-issuer
    namespace: default
spec:
    acme:
        # The ACME server URL
        server: https://acme-v02.api.letsencrypt.org/directory
        # 用于AGME的注册
        email: my@163.com
        # 用于存储向AGME注册的 private key
        privateKeySecretRef:
            name: cert-issuer-key
        # 启用 http-01 challenge
        http01: {}
EOF
kubectl apply -f issuer.yaml
```
如果要设定 ClusterIssuer 把yaml中的kind设置成ClusterIssuer即可
### 添加 secret
生成签名密钥并创建 k8s secret
``` bash
# Generate a CA private key
$ openssl genrsa -out ca.key 2048

# Create a self signed Certificate, valid for 10yrs with the 'signing' option set
$ openssl req -x509 -new -nodes -key ca.key -subj "/CN=kube-ingress" -days 3650 -reqexts v3_req -extensions v3_ca -out ca.crt

$ kubectl create secret tls my-tls \
   --cert=ca.crt \
   --key=ca.key \
   --namespace=default
```
### 取得 Certificate
``` bash
cat>my-cert.yaml<<EOF
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: my-cert
  namespace: default
spec:
  secretName: my-tls
  issuerRef:
    name: cert-issuer
    ## 如果使用 ClusterIssuer 需要添加 kind: ClusterIssuer
    #kind: ClusterIssuer
  commonName: ""
  dnsNames:
  - www.example.com
  acme:
    config:
    - http01:
        ingress: ""
      domains:
      - www.example.com
EOF

#创建
kubectl apply -f my-cert.yaml

#查看进度
kubectl describe certificate my-cert -n default

```

参考文档
----------
* [https://cert-manager.readthedocs.io/en/latest/tutorials/acme/http-validation.html](https://cert-manager.readthedocs.io/en/latest/tutorials/acme/http-validation.html)