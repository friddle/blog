---
title: "关于自签名证书到IOS13不支持问题"
categories:
  - Tech
tags:
  - Grpc
  - IOS
---

### 关于IOS13后自签名证书不合法的问题

苹果官方文档。明确表明证书新的需求。文档为 [https://support.apple.com/zh-cn/HT210176](https://support.apple.com/zh-cn/HT210176)  
主要麻烦点集中于。生成
>  TLS 服务器证书必须在证书的“使用者备用名称”扩展中显示服务器的 DNS 名称。证书的 CommonName 中的 DNS 名称不再受信任。


##### rsakey的生成 
rsa key的生成:`openssl genrsa -out private/friddle.pem 4096`
> 要2048以上的长度

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
keyUsage = keyEncipherment, dataEncipherment,nonRepudiation, digitalSignature  
extendedKeyUsage = serverAuth,clientAuth,codeSigning, emailProtection #必须添加serverAuth和codeSigning
subjectAltName = @alt_names
basicConstraints = CA:FALSE


[alt_names]
DNS.1 = appapi.friddle.me
DNS.2 = appapi2.friddle.me
```
##### 生成site命令
```
openssl req -new -x509 -key private/friddle.pem -out ./private/sites/appapi.friddle.pem -days 824 -config openssl.cfg -extensions 'v3_req'
```

>  注意必须是825天以+sha256的加密规则
>  必须添加extendKeyUsage加上 serverAuth,codeSigning,digitalSignature 

#### dart的grpc问题
由证书造成的dart问题。可以通过加onBadCertificate的重构覆盖掉就行了
以现在非常稀少的flutter+grpc框架。恩。这个还是挺好的

```
  ClientChannel get releaseChannel =>  ClientChannel('appapi.friddle.me',
      port: 443,
      options: ChannelOptions(
          credentials: ChannelCredentials.secure(
              certificates: _sslKey, password: null, authority: "appap.friddle.me", onBadCertificate: (X509Certificate certificate, String host){
                return true;
          })));

```