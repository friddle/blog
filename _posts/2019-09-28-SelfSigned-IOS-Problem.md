---
title: "关于自签名证书到IOS13不支持问题"
categories:
  - Tech
tags:
  - Grpc
  - IOS
---

### 关于IOS13后自签名证书不合法的问题

苹果官方文档。明确表明证书新的需求。文档为 [https://support.apple.com/zh-cn/HT210176](!https://support.apple.com/zh-cn/HT210176)  
主要麻烦点集中于。生成
>  TLS 服务器证书必须在证书的“使用者备用名称”扩展中显示服务器的 DNS 名称。证书的 CommonName 中的 DNS 名称不再受信任。


##### rsakey的生成 
rsa key的生成:`openssl genrsa -out private/friddle.pem 4096`

##### 准备SSL配置脚本 
```
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
C = CN
ST = HuNan
L = Changsha
O = DrinkBird
OU = DrinkBird
CN = www.friddle.me


[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = appapi.friddle.me
DNS.2 = appapi2.friddle.me
```
##### 生成site命令
```
openssl req -new -x509 -key private/friddle.pem -out ./private/sites/appapi.friddle.pem -days 824 -config openssl.cfg -extensions 'v3_req'
```