---
title: "购买SSL证书流水记"
categories:
  - Tech
tags:
  - Nginx
  - SSL
---

### 关于购买证书时候的考虑。

   开始想选择沃通的证书。但是最近负面新闻爆表。Chrome和Firefox直接要屏蔽吊沃通的证书。所以直接说Goodbye了。   
   然后就选择了Godaddy的证书。虽然网页介绍的信息等于无。每次打电话过去咨询。人家好听的客服妹子听了我的描述基本就是:`先生你等几分钟。我跟美国Godaddy那边的技术团队交流下` 知道为什么国外的互联网搞不赢中国的原因了。    
   最后还是选了Godaddy的泛域名证书。然后就是辛苦的配置之旅了
    
### 关于SSL生成时候的操作

生成私钥证书和CRT：  
`openssl genrsa -out friddle.key 1024`    
`openssl req -new -sha256 -key ./friddle.key -out ./friddle.csr`   

这一部分填的东西多。不同的网站估计会生成不同的csr。
而且又容易填错。我就写了一个脚本。通过改脚本生成比手动打靠谱。

```
#!/usr/bin/expect
#generate openssl to 

set website_prefix [lindex $argv 0]
spawn openssl req -new -sha256 -key ./friddle.key -out ./$website_prefix.friddle.csr
expect -re "Country.*$"
send "CN\r"
expect -re "State.*$"
send "Hunan\r"
expect -re "Locality.*$"
send "Changsha\r"
expect -re "Organization Name.*$"
send "Friddle Co., Ltd\r"
expect -re "Organizational Unit Name.*$"
send  "Friddle\r"
expect -re "Common.*$"
send  "$website_prefix.friddle.com\r"
expect -re  "Email.*$"
send  "friddle@friddle.me\r"
expect -re  "A challenge.*$"
send  "\r"
expect -re  "An optional.*$"
send  "\r"
expect eof
exit 
```

一般来说COMMON_NAME对应的是你需要保护的域名地址。比如假如你买的泛域名证书就需要填 `*.friddle.me`

生成CRT后就可以把证书提交给Godaddy了。接下来就是配置你域名的TXT了或者邮箱发邮件就行验证。这点还是比较简单。
然后在把Godaddy生成的服务器证书文件下过来。

### 提前准备。
1.   各种服务开HTTPS端口配置。       
2.   各种CDN的HTTPS配置。    

基本上大部分服务都跑的SLB。所以很幸运。大部分只要在SLB上加上去就行了。但是有些服务还是得自己配置。

配置CDN的时候。阿里的同一个CDN地址都默认可以直接http和https的双向配置。而七牛的。就需要手动发工单过去。简直了。
难得吐槽一翻。提工单这种很影响效率的。

### 配置Nginx

基础的Nginx的基础配置。基本上加上去就ok了。
```
   server {
        listen       443 ssl;
        server_name  *.friddle.me;

        ssl_certificate       cert/friddle.crt;
        ssl_certificate_key   cert/friddle.key;
   }
```

假如想全局跳转所有Http的访问请求到HTTPS则可以配置
```
    server {
        listen       80;
        server_name  *.friddle.me;
        rewrite  ^ https://$host$request_uri? permanent;
    } 
```

同时对后端的跳转最好做一下redirect的修改
```
      location / {
            proxy_set_header client-real-url $scheme://$host$request_uri;
            proxy_set_header Host $host;
            proxy_set_header X-Real-Ip $remote_addr;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_redirect ^ https://$host:$request_uri?;
            proxy_pass http://xxx;
        } 
```

最好注意下.转发有个`X-Forwarded-Proto` 这是一个规范.  
这个规范对于后端库来说可以意识到最远程的访问请求是Https或者Http开头的.这个比较重要.  
因为默认后端跳转代码一般会从后端Get默认的协议.而没加这个头.Tomcat或者后端程序做Http的.



### 性能问题
服务都直接配置在阿里云的SLB上。所以不存在性能问题。