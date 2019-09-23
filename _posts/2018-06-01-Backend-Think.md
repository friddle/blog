---
title: "对于中型系统的后端思考"
categories:
  - Manage
tags:
  - Spring
  - SSL
---

## 想想看一个总结吧。       
从0做了一个中型系统。感觉经验上的收获还是挺多的。    
以前都是涉及边缘比较难的业务。不太想触及无聊的中层业务。感觉挺烦的。确实挺烦的。不过还是挺有收获的。。   

## 工具链
首先还是得感谢框架的kotlin/spring/grpc工具的idea+deepin。
这个工具链很满意没啥吐槽的。尤其感谢kotlin。对于大部分业务逻辑来说。真的少写好多代码。基本上以我这种拖拉的性格。还是在1天时间从3个表左右的数据库设计到接口完成。在本人的推动下。团队（我们团队在长沙。技术能力大家懂的）也开始慢慢习惯用Kotlin了。有效减少无用逻辑。

Spring也没吐槽的。太成熟了。而且虽然是重。但是感觉真的分层设计的好。真的就能很容易的屏蔽和发现问题。话说现在我解决Spring问题都是直接去看源代码的。不得不承认。Spring和Tomcat发展到现在。核心的架构还是改的很少。这个真的很重要。我在构建自自己的系统也在想这方面的问题。  

说点细节的思考把。在业务复杂度慢慢失控的几个点。感觉对的方向和错误的方向。都集中于核心几个点。 
  
当然不能说搞个完美的系统架构。以前觉得系统只要和核心业务很近。就改的比较少。实际上。业务重心是会变的。说几个点吧。

### 1.表的设计要原子化。一致化。

这个绝对不能偷懒。业务层可以偷懒。可以写点很tough的逻辑。但是在数据库层面是绝对不能偷懒的。因为复杂的数据关系就会造成复杂的业务逻辑。到后面。数据大家都不敢动。业务层也就不敢动。上线后谁都不想大改数据库。因为数据弄乱复原比业务弄乱更麻烦。

表的设计一定要原子化。不能为了偷懒随意加字段。该创建表的时候一定要创建表。还有非流程内的其他数据可以用一个字段。存Json来记录。举一个简单的列子。

本来是支付购买会员的。但是后来业务上改成了支付购买实物商品赠送会员。（因为可恶的Apple）。队友提议直接在Order表里面加个userAddress和一个物流状态。暂时能很快的支持线上的要求（时间紧）。

然后想了下。被我否决了。我认为几个点吧。一个是。这样逻辑关系又会弄乱。一旦以后有需求去查询我的商品信息的时候去查我的订单。这在怎样说不过去。我说代码可以先写成固定的。但是数据库一定要搞个原子化的表出来。一个是我的物品栏表。一个是发货状态地址表。业务就先写死了下单后就默认添加到用户栏就发货。有时间就拆分。果然在后期的需求的时候。这个设计就不会因为数据的混乱而不好改。只需要改业务逻辑就对了。

数据集一定要原子化。就是一个完整的行为的数据集。最后可以支撑一个单独的业务逻辑。一个单独的解释。不要想一个表满足一个业务需求的多个方面。而是多个表满足一个业务需求。多对多的关系最好用关联表链接。不要做冗余字段。

### 2.业务层尽量还是用Service层沟通+缓存系统,少用表关联

连表查询确实容易简单很多逻辑。开始也喜欢用连表。毕竟少一层逻辑代码也开心一层。但是实际上不是这样的。    
因为把业务逻辑写在xml层。会造成很多业务逻辑无法复用。而且没办法做到很好的解耦。一旦一个小的逻辑要改会造成很多xml都面临要改。    
当然全部写在数据层也有问题。造成一个业务多次查表。所以这个时候需要缓存系统的存在。缓存系统可以优秀缓解业务压力。  
当然清理缓存也会有清理缓存的问题。有时候清理缓存和生成缓存并不在同样的位置。具体怎么解决。等下再讲。


### 3.抽象任务。
有一种类型的任务叫规则任务：     
意味着我习惯把一串复杂的规则归纳统计。然后做成数据库数据。做总控开关。  

比如新创建用户我默认送她7天会员。对于我来说就是一个规则。这个规则是经常变的。比如会换成3天？。比如不送会员了。送其他东西。

生成一个数据库的表。然后记录相关的规则。每一条数据都作为一个规则。并把参数配置，状态以及一些注释丢到里面。然后以这个表的数据作为总控开关。相当灵活。这样做到了充分的解耦。    

说道这里就说到实现。实现的话下面详细述说

### 4.善用Spring的Bean管理。
分层是一个运行效率低的东西。毕竟多层调用不方便。但是实际上真的有利于代码的组织。
Aop估计大家都知道。但是Bean管理估计大家用的不多。

针对上面说的即拆即拔很简单。
```kotlin
  interafce IRule
  {
      fun ruleId():Int
      fun doAction(params:List<*>):Pair<Boolean,String>
  }
```
定义好这样一个类。然后实现一个简单的继承
```kotlin
  @Service
  class RuleOne
  {
    open fun ruleId():{return 1}
    open fun doAction(params:List<*>){return Pair(success,"do actions")}
  }
```
然后在调用的时候可以直接这样
```
 context.getBeansOfType(IRule::class.java).filter{it.ruleId()==ruleId}.doActions(params)  
```
这样的话就可以做到即拆即拔了。只要实现了相应接口就直接进入了业务逻辑中相当方便。
这样的逻辑适合在支付的时候实现一套平台接口（微信，支付宝）。主体逻辑不需要任何的改变。相应的依赖什么的都不需要。完全抽离开来了。
这是一个小技巧。这只是善用Spring的Bean管理的一个小方面。SpringBean管理还有很多方便的东西。这个都挺重要的。

### 5.消息管理和微量型的Message的实现。
我们以前用的消息管理都是RabitMQ等大型的。这次这个项目我不是很想用这套。规则挺多的。

说一下具体的需求来源。
我们当时有一个业务统计阅读数据。实时的。很少更新。     
但是更新后需要清理缓存。而这个阅读行为不在统计业务的模块里面。
而这个清理数据暂时被我丢到阅读这个模块的逻辑代码里面。但是有一天重新看到这个代码的时候我就觉得不太好。不好解耦。
一秒钟想到了消息队列。毕竟阅读这个行为是个消息。而这个消息发生。各个地方的代码都会因为这个动作进行相应的联动。完全的解耦。一旦这个模块放弃了。相应的不会影响到另一个业务逻辑。。

然后想了一下。毕竟还没有到大型项目。要用到RabitMq等逻辑。要用到错误机制。
只是业务内的充分解耦。所以自己实现了一套简单的实现。以后扩大后就用Mq实现来代替当前实现。

定义MessageConsumer
```
open interface IMessageConsumer
{
    fun messageType():Int
    fun receiverMessage(t:List<Any?>,callback:(()->Pair<Boolean,String>)?):Pair<Boolean,String>
}
```
实现一个
```
    @Bean
    fun clearCache(): IMessageConsumer {
        return object:IMessageConsumer
        {
            override fun messageType(): Int {
                return MESSAGE_TYPE_BUY_VIP
            }

            override fun receiverMessage(t: List<Any?>, callback: (() -> Pair<Boolean, String>)?): Pair<Boolean, String> {
                return Pair(true,"")
            }

        }
    }
```

一个简单的实现（当然具体实现还是添加了一些东西的）
```
@Service
open class MessageQueue {
    
    @Autowired
    lateinit var applicationContext: ApplicationContext
    var logger=LoggerFactory.getLogger(MessageQueue::class.java)
    var threadPool= Executors.newWorkStealingPool()
    var consumers= arrayListOf<IMessageConsumer>()

     fun sendMessage(type:Int,t:List<Any?>):Pair<Boolean,String>
    {
        val typeConsumer=applicationContext
           .getBeansOfType(IMessageConsumer::class.java)
           .values.filter { it.messageType()== type}


        for(consumer:IMessageConsumer in typeConsumer)
        {
            threadPool.execute {
                try {
                    val result=consumer.receiverMessage(t,callback)
                    if(!result.first) {
                        throw Exception("task run failed:"+result.second)
                    }
                } catch (e:Exception) {                  
                    logger.error("consumer error:", e)
                }
            }
        }
        return Pair(true,"")
    }
}
```
具体用什么Pool大家自己去考虑去。FixedThreadPool也不错。


### 6.SpringContextHolder和Utils
有些时候涉及到数据库和配置的工具类没法写成静态方法。这个时候一个小技巧就是巧用SpringContextHolder 这个是我从别人那里拷过来的。这个思路真的挺不错的

```
@Component
object SpringContextHolder:ApplicationContextAware{
     private var applicationContext: ApplicationContext?=null;

    override fun setApplicationContext(applicationContext: ApplicationContext) {
        this.applicationContext=applicationContext
    }

    fun getApplicationContext():ApplicationContext?
    {
        return applicationContext
    }

    fun <T> getBean(beanName:String): T? {
        if(applicationContext==null)return null
        return applicationContext!!.getBean(beanName) as T
    }

    fun <T> getBean(classz:Class<T>):T?{
        if(applicationContext==null)return null
        return applicationContext!!.getBean(classz)
    }
}
```
工具类实现。
```
object AreaUtil
{
    fun getAreaByCode(code:String): QqArea?
    {
        return SpringContextHolder
              .getBean(IAreaService::class.java)!!.getAreaByCode(code)
    }
}
```

调用
```
    AreaUtil.getAreaByCode("xxxxxxx")
```
这是利用Spring的事件监听来实现的。当启动完成后把ApplicationContext绑到SpringContextHolder这个静态类里面。然后所有工具类都可以用这个工具类获得动态Bean了。工具类还是这样调用舒服点。






    

