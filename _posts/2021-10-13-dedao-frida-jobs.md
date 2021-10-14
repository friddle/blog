---
title: frida-js的Retrofit思路和尝试
date: 2021-10-13 15:42 +0000
link: https://zhuanlan.zhihu.com/p/315762467
categories:
  - Tech
tags:
  - Flutter
---

# frida-js的Retrofit思路和尝试

开始博客又搞来搞去。本来准备在知乎上了。不过可以在这里写。然后知乎上再拷贝一份    
Retrofit是一个很牛逼的框架/Frida也是。我作为新手。通过hack某个App一周多。也算正式入门了    

### 查看接口数据
而我hack的App采用了Rxjava2+Retrofit2+Kotlin. 我们有两个思路。一个是搞定Sign另一个是frida-js调用Retrofit来模拟实现。   
一般来说肯定是搞定Sign是最好的。然后通过就可以自己写代码了   

### 前期准备Sign的Fail
用大神Frida-DexDump的项目成功脱壳后。然后用jeb(比jadx好用) 然后打开dex后整个人ORZ了。因为kotlin做了很多优化。所以根本无法反编译。。而我的能力根本不行。根本看不出是调用了那个方法。。满屏的vtaboff...
所以失败

### Retrofit的Hack
因为接口用了Retrofit2。所以思路是怎么Hacking:Retrofit。
而网上搜索了下。方法确实比较少。没有我这种新手博客和文章。所以这就是为什么写这篇文章的原因。
Retrofit的调用    
别人代码 Retrofit调用代码。。  
```
Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(API.BASE_URL)//基础URL 建议以 / 结尾
                .addConverterFactory(GsonConverterFactory.create())//设置 Json 转换器
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())//RxJava 适配器
                .build();
        RxWeatherService rxjavaService = retrofit.create(RxWeatherService .class);
        rxjavaService .getMessage("北京")
                .subscribeOn(Schedulers.io())//IO线程加载数据
                .observeOn(AndroidSchedulers.mainThread())//主线程显示数据
                .subscribe(new Subscriber<WeatherEntity>() {
                    @Override
                    public void onCompleted() {

                    }

                    @Override
                    public void onError(Throwable e) {

                    }

                    @Override
                    public void onNext(WeatherEntity weatherEntity) {
                       Log.e("TAG", "response == " + weatherEntity.getData().getGanmao());
                    }
                });
```
获得Retrofit2.Retrrofit/Retrofit2.Api的调用对象
实际上尝试了三个思路。两个通用思路。一个是偷懒思路。结果都成功了。这算比较容易的 。

##### 第一个版本:
```
var api_invoke=[]
var api_invoke_temp_name=[]
var StartUpListener=function(){
     var retrofitClass="retrofit2.Retrofit"
     Java.use(retrofitClass).create.overload("java.lang.Class").implementation=function(classes_)
     {
        var result=this.create.overload("java.lang.Class").call(this,classes_)
        api_invoke.push({name:classes_.getName(),obj:result})
        api_invoke_temp_name.push({name:classes_.getName(),proxy_name:result.getClass().getName()})
        return result;
    };

    //A
    let proxy_name=api_invoke_temp_name.filter(item=>item.name==='ApiName')[0].proxy_name
    Java.choose(proxy_name,{
                onMatch:function (instance) {
                    //开始调用接口代码

                },
                onComplete:function () {
                }
})
```

这个思路是燕过拔毛形的。就是修改Retrofit.create监听并保存生成的对象。缺点就是很明确的。需要在启动的时候运行.
删程序后在运行 `objection -g com.friddle --startup-command='import ./test1.js'`

##### 第二种方法
```
var Api;
var rTrendSearchApi=Java.use("com.friddle.api.SearchApi")
 Java.choose(retrofitClass,{
                onMatch:function (instance) {
                    try{
                        let http_url=instance._baseUrl.value.toString()
                        if(http_url.indexOf('app.friddle.com')!==-1)
                        {    
                            //在这里创建
                                let trendApiObj=Java.cast(retrofit.create(rTrendSearchApi.class),rTrendSearchApi);    
                                  Api=trendApiObj;                      
                        }
                    }catch (e) {
                        console.log(e)
                    }         
    }
``` 
这样方法比较通用。而且什么时候都可以调用.
Java.choose去找相应的方法。只是自己要过滤。比如拿host来过滤就显的非常简单
第三钟方式。是hack的App有一个整体的静态类管理所有的Retrofit。所以没有可参考性。是在监控Builder类的时候发现。
object:android hooking watch class retrofit2.Retrofit$Builder
监控Retrofit2的接口和调用
Retrofit2的create方法

```
@SuppressWarnings("unchecked") // Single-interface proxy creation guarded by parameter safety.
  public <T> T create(final Class<T> service) {
    validateServiceInterface(service);
    return (T)
        Proxy.newProxyInstance(
            service.getClassLoader(),
            new Class<?>[] {service},
            new InvocationHandler() {
              private final Platform platform = Platform.get();
              private final Object[] emptyArgs = new Object[0];

              @Override
              public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                  throws Throwable {
                // If the method is a method from Object then defer to normal invocation.
                if (method.getDeclaringClass() == Object.class) {
                  return method.invoke(this, args);
                }
                args = args != null ? args : emptyArgs;
                return platform.isDefaultMethod(method)
                    ? platform.invokeDefaultMethod(method, service, proxy, args)
                    : loadServiceMethod(method).invoke(args);
              }
            });
  }
  ```
Retrofit2是解析了接口的Annotation的Interface然后直接使用了官方的Proxy包（我的记忆还停留在cgilib年代，没想到都内置了）那就核心就是这句。
两个都做hack就可以打出日志
```
function json_output(obj)
{
    try{
        const gson = Java.use('com.google.gson.Gson');
        return (gson.$new().toJson(obj)).toString()
    }catch (e) {
        return "error:"+e
    }
}
function Retrofit_Listener() {
    var rRetrofitClass = Java.use("retrofit2.Retrofit")
    rRetrofitClass.loadServiceMethod.implementation=function(_method)
    {
        var invokeClass=_method.getDeclaringClass();
        var invokeParams=_method.getParameters()
        console.log("class_name:"+invokeClass.getName()+
            "\tmethod_name:"+_method.getName()
            +"\tinvoke_params:"+invokeParams.map(function(item,index){
                return "index_"+index.toString()+":"+item.getType().getName()
            }).join(",")
        )
        var result=this.loadServiceMethod.call(this,_method);
        return result;

    }
    var rHttpServiceMethod=Java.use("retrofit2.HttpServiceMethod")
    rHttpServiceMethod.invoke.overload("[Ljava.lang.Object;").implementation=function (classes_) {
        let params_text=",";
        for(var s_class of classes_)
        {
            params_text=params_text+","+json_output(s_class)
        }
        var result=this.invoke.overload("[Ljava.lang.Object;").call(this,classes_)
        console.log("invoke_params:"+params_text)
        return result
    };





调用接口
function OpenStrictMode()
{
  Java.perform(function(){
    let rStrictMode=Java.use('android.os.StrictMode')
    let rStrictThreadMode=Java.use('android.os.StrictMode$ThreadPolicy$Builder')
    rStrictMode.setThreadPolicy(rStrictThreadMode.$new().permitAll().build())
    return
  })
}

function json_output(obj)
{
  const gson = Java.use('com.google.gson.Gson');
  return (gson.$new().toJson(obj)).toString()
}


function json_from(obj_classz,value)
{
  let rFastJson=Java.use("com.alibaba.fastjson.JSON")
  let rValueClass=Java.use(obj_classz)
  let result=Java.cast(rFastJson.parseObject(value,rValueClass.class),rValueClass)
  return result
}


var result=""
function invokeDynamic(body) {
  OpenStrictMode()
  let apiName = body.apiName;
  let methodName = body.methodName;
  let objs = body.params;
  Java.perform(function () {
    var rApiObj = Java.use(apiName)
    var rRestClient = Java.use("com.friddle.RestClient")
    //c ,e ->app.dewu.com g->m.dewu.com(这里update版本后就要更新)
    try {
      let retrofit = rRestClient.x();
      let ApiObj = Java.cast(retrofit.create(rApiObj.class), rApiObj)
      let newObjs=objs.map(function(item){
        if(item.type==='string')
        {
          return item.value;
        }
        if(item.type==='int')
        {
          return item.value
        }
        else {
          let transform=json_from(item.type,JSON.stringify(item.value))
          return transform
        }
      });
      let methodObservable = ApiObj[methodName].call(ApiObj, ...newObjs)
      result = json_output(methodObservable.blockingSingle())
      return result;
    } catch (e) {
      console.log("error"+e)
      result = "error" + e
    }
  })
  console.log("result_finish"+result)
  return result
}

function depositSearch(searchWord,page){
  var params={
    apiName:"com.friddle.DepositService",
    methodName:"getSearchList",
    params:[
      {type:"string",value:searchWord},
      {type:"int",value:20},
      {type:"int",value:parseInt(page)}
    ]
  }
  return this.invokeDynamic(params)
}
```
因为直接在方法中调用。默认走的是主线程。Android默认不允许走主线程。所以先要设置主线程打开 OpenStrictMode    
为了使用方便。obj转json然后json再用gson转。做了两层转（其实写后端引擎就是这样偷懒的）。用fastjson是因为gson默认会把Number转成Double....    
本来是通过dex生成/registerClass的方式回调获得（因为真的不是Android开发,所以不熟悉），然后没有数据输出。突然发现可以调用blockingSingle方式就直接返回了。瞬间解决。真好
dex生成方式    

## frida-js一些脚本

#### arida项目
arida-page 这个项目还是挺方便的。就是环境搞了好久。后面干脆自己手动安装的依赖，没有使用conda   
搞完后就用fastapi调试接口。 然后在加上element+vue就可以直接提供接口服务了。这里就不展示了。给个截图

![image](assets/images/luoxiaohei.jpg)