## 前言

本文记录k3s使用letsencrypt配置ssl证书和续签。

本文使用的k3s版本为： v1.23.6+k3s1。

## 安装 cert-manager

```bash
# 使用官网提供的配置文件一键安装
# 如果拉取 github 资源有困难，可以从网络通畅的位置下载好粘贴过去
$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.yaml
```

如果一切正常，k3s

```
$ kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-6544c44c6b-tlhw8              1/1     Running   0          51s
cert-manager-cainjector-5687864d5f-rldsg   1/1     Running   0          51s
cert-manager-webhook-785bb86798-g4g7g      1/1     Running   0          51s
```

## 部署 Issuing Certificates

创建letsencrypt.yaml文件

```yml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  namespace: cert-manager
  name: letsencrypt
spec:
  acme:
    email: <YOUR EMAIL> # replace this
    privateKeySecretRef:
      name: prod-issuer-account-key
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
      - http01:
          ingress:
            class: traefik
        selector: {}
```

> 请自行替换<YOUR EMAIL>为个人的邮件地址

部署

```
$ kubectl apply -f letsencrypt.yaml
```

如果一切正常:

```
$ kubectl describe clusterissuer letsencrypt
Name:         letsencrypt
Namespace:    
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         ClusterIssuer
...
Spec:
  Acme:
    Email:            user@example.com
    Preferred Chain:  
    Private Key Secret Ref:
      Name:  prod-issuer-account-key
    Server:  https://acme-v02.api.letsencrypt.org/directory
    Solvers:
      http01:
        Ingress:
          Class:  traefik
      Selector:
Status:
  Acme:
    Last Registered Email:  mingguobin@live.com
    Uri:                    https://acme-v02.api.letsencrypt.org/acme/acct/715199687
  Conditions:
    Last Transition Time:  2022-09-03T12:20:23Z
    Message:               The ACME account was registered with the ACME server
    Observed Generation:   1
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
```

## 给Ingress配置ssl

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: hoteler-namespace
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
  name: hoteler-web
spec:
  tls:
  - secretName: hoteler-web-tls
    hosts:
      - hoteler.damingerdai.com
  # ingressClassName: traefik
  rules:
  - host: hoteler.damingerdai.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hoteler-web
            port:
              number: 80
```

改动如下：

1. 添加annotations的metadata

```yml
annotations:
  cert-manager.io/cluster-issuer: letsencrypt
```

2. 添加tls

```yml
tls:
- secretName: hoteler-web-tls
  hosts:
    - hoteler.damingerdai.com
```

secretName可以按照自己的习惯来取，cert-manager会自动创建

```
$ kubectl apply -f ingress.yaml -n hoteler-namespace
$ kubectl get ingress -n hoteler-namespace
hoteler-web                 <none>   hoteler.damingerdai.com         204.44.75.77   80, 443   115d
```

可以看到hoteler-web除了80端口，还新加了443端口


## 查看证书状态

```shell
$ kubectl describe certificate -n hoteler-namespace
...
Status:
  Conditions:
    Last Transition Time:  2022-09-03T12:27:15Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2022-12-02T11:27:13Z
  Not Before:              2022-09-03T11:27:14Z
  Renewal Time:            2022-11-02T11:27:13Z
  Revision:                1
Events:
  Type    Reason     Age   From                                       Message
  ----    ------     ----  ----                                       -------
  Normal  Issuing    30m   cert-manager-certificates-trigger          Issuing certificate as Secret does not exist
  Normal  Generated  30m   cert-manager-certificates-key-manager      Stored new private key in temporary Secret resource "hoteler-web-tls-pfxrp"
  Normal  Requested  30m   cert-manager-certificates-request-manager  Created new CertificateRequest resource "hoteler-web-tls-v24f6"
  Normal  Issuing    29m   cert-manager-certificates-issuing          The certificate has been successfully issued
```

## 效果

![](assets/back-end/hoteler-web-certificate.png)

![Hoteler Web](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/back-end/hoteler-web-certificate.png)

## 参考资料

1. [k3s 使用 Letsencrypt 和 Traefik 完成 https 入口部署](https://www.frytea.com/technology/k8s/k3s-uses-letsencrypt-and-traefik-to-deploy-the-https/)
2. [K3S与Docker常用命令](http://t.zoukankan.com/dream2true-p-13064701.html)
3. [Securing NGINX-ingress](https://cert-manager.io/docs/tutorials/acme/nginx-ingress/)
4. [Documenting "context deadline exceeded" errors relating to the webhook](https://github.com/cert-manager/cert-manager/issues/2319)