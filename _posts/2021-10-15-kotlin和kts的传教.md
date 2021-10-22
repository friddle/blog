---
title: Kotlin Script推广
categories:
  - Tech
tags:
  - Kotlin Java Script
date: 2021-10-14 02:56 +0000
---

此文章就两个目的啊。推广Kotlin Script和常见问题的处理办法  

## 为什么建议使用KScript
假如让你写个脚本。你优先选择的是什么?是Shell,是Python。毕竟基础操作shell够了。复杂的操作Python也够了。   
但是你想在跑项目中的代码并用熟悉的java/Kotlin语言来执行一些操作。这个时候假如用Java和Kscript直接写代码而复用业务逻辑代码。会不会很有想象力。   
这样就不需要打包成一个jar然后线上运行。
假如刚好又没有自己环境的serverless之就很好用了

## 如何使用
安装[sdkman](https://sdkman.io/).或者直接执行`curl -s "https://get.sdkman.io" | bash`
`sdk install kotlin`
`sdk install kscript`

具体项目地址为:`https://github.com/holgerbrandl/kscript`


## HelloWorld
```kotlin
#!/usr/bin/env kscript
println("Hello from Kotlin!")
for (arg in args) {
    println("arg: $arg")
}
```
直接保存并执行 `kscript ./hello.kts`,`./hello.kts`
输出
```
Hello from Kotlin!
```

## 基础使用

#### Kotlin注解

```kotlin
#!/usr/bin/env kscript
@file:MavenRepository("aliyun","https://maven.aliyun.com/repository/public")
//@file:CompilerOpts("-jvm-target 1.8")
@file:DependsOn("com.github.holgerbrandl:kutils:0.12")
@file:DependsOn("com.google.code.gson:gson:2.8.6")
@file:KotlinOpts("-J'--add-opens=java.base/java.util=ALL-UNNAMED'")
@file:EntryPoint("Foo.bar") 
@file:Include("util.kt")

println("Hello from Kotlin!")
for (arg in args) {
    println("arg: $arg")
}
```
1. @file:MavenRepository是设置Maven的Repository。这里可以换成常见Maven镜像   
2. @file:DependsOn是需要的依赖。     
3. @file:KotlinOpts是Kotlin编译的参数 `-J'xxxx'` 是Jvm参数  
4. @file:EntryPoint("Foo.bar") 是在多文件的时候制定启动点
5. @file:Include是包括相应文件
  
基本上这些注解对于大部分开发都是够了的

### 其他工具
`kscript --interactive CountRecords.kts`
可以直接进行命令行交互

`kscript --add-bootstrap-header some_script.kts`
添加kts转换到shell开发

`kscript --idea CountRecords.kts`
用idea开发kts脚本

### 小建议

#### 一些问题
Java11后模块化后一些默认依赖没有。假如出现了这个问题
```txt
java.lang.ClassNotFoundException: java.sql.SQL
```
这里正确解决方式是
```
@file:KotlinOpts("-J'--add-opens=java.base/java.util=ALL-UNNAMED'")
```
这边相应的类和模块需要自己查询对应



### 推荐使用库
```kotlin
//excel处理
@file:DependsOn("org.apache.poi:poi:5.0.0")
@file:DependsOn("org.apache.poi:poi-ooxml:5.0.0")

//gson
@file:DependsOn("com.google.code.gson:gson:2.8.6")
//邮箱
@file:DependsOn("net.axay:simplekotlinmail-client:1.3.2")
@file:DependsOn("net.axay:simplekotlinmail-core:1.3.2")
//httpClient
@file:DependsOn("com.squareup.okhttp3:okhttp:3.9.1")
@file:DependsOn("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.4.3")
```
  
