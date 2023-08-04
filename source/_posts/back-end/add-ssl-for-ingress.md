---
title: 给Ingress上配置ssl证书
date: 2022-06-18 13:04:18
tags: [k8s, ingress, ssl]
categories: [后端]
---

#  给Ingress上配置ssl证书

## 创建secret

```bash
kubectl create secret tls [secretName]  --cert=[pem文件路径] --key=[key文件路径] --namespace [namespace] -o yaml --dry-run=client > ingress-default-cert.yaml


kubectl apply -f ingress-default-cert.yaml
```

## Ingress添加证书

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress  
metadata: 
    name: [ingress-name] 
    namespace: [namespace] # ingress要和secret在同一个名称空间下 
    annotations: 
        kubernetes.io/ingress.class: traefik 
        traefik.frontend.rule.type: PathPrefixStrip 
        # http 重定向到 https 
        ingress.kubernetes.io/ssl-redirect: "True" 
spec: 
    tls: 
        - hosts: 
            - xxxx.xxxx # 这里是下面要配置https的域名 
            - xxxx.xxxx # 这里是下面要配置https的域名 
        secretName: 
            [secret-name]: 
    rules:
    - host: xxx.xxx.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: [service-name]
                port:
                  number: 8080

```


## 参考资料

1. [为k8s集群配置SSL证书](https://mars-tian.github.io/2022/01/10/%E4%B8%BAk8s%E9%9B%86%E7%BE%A4%E9%85%8D%E7%BD%AESSL%E8%AF%81%E4%B9%A6/)
2. [k3s配置ingress使用ssl证书](https://ubisoft-potato.github.io/2019/11/03/k3s-pei-zhi-ingress-shi-yong-ssl-zheng-shu/)