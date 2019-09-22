---
title: "Nginx不能获取变量的坑"
categories:
  - Tech
tags:
  - link
  - Post Formats
link: http://blog.friddle.me/Tech/NginxCannotGetVariable
---

最近做Docker镜像.需要获取环境变量.但是在网上查的都是这种写法.包括StackOverflow.但是就是没有返回.真的是一筹莫展.      
本地lua写这句话是正确的.就是在Nginx不返回任何值.

```
http {
  ...
  server {
    location / {
      set_by_lua $api_key 'return os.getenv("API_KEY")';
      ...
    }
  }
}
```

然后果然在用Google深度搜索后,发现agentzh大大果然又是明灯指导我们.
给出了文档地址
[nginx](!http://nginx.org/en/docs/ngx_core_module.html#env)    
明确告诉我们.在运行状态的时候Nginx已经果断把所有环境变量移除了.    
好的被坑了.

那就只能在`init_by_lua`里面使用了.
```
http
{ 

	init_by_lua_block
	{
		test=os.getenv("PATH");
	}
    server{
     ....
       location /
       { 

            set_by_lua $test 'return test';
       }
    }
}

```
解决这个问题真的是已经准备看源代码了.恩.幸亏有明灯.