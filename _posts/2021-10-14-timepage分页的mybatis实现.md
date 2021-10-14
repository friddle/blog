---
title: Page通过时间分页的Mybatis实现
categories:
  - Tech
tags:
  - Mybtais Spring
date: 2021-10-14 02:56 +0000
---

#  时间分页需求诞生
需求来自于我有多个数据源（或者说多个表）合并成一个数据给前端展示。   
但是数据量大到肯定需要分页。但是因为是多个数据源（表的结构根本不一致根本无法实现连表）所以就想了一个办法。做时间分页。    
因为时间是统一和固定的。所以分布的系统中统一按时间进行分页返回比较合适.   

当然还有更加完善但是工作量巨大的方式。内存维护一个队列。并监听各表数据库操作然后实现update数据变动到内存队列.这种工作量还好,维护起来还是挺麻烦的.

#  灵感来源
来源于有时候用微博直接按年代区分的。需要自己点击年代。那为何不干脆直接用时间分页呢.

#  设计思路
## 用法
首先这个用法肯定是能抄袭Mybtais-Plus的PaginationInterceptor(分页插件 | MyBatis-Plus)
使用方式肯定是类似的
```kotlin
   fun testPage(page:Page<T>):List<T>
   fun testTimePage(@Param("create_time")page: TimePage<T>):List<T>
```


## 思路
   
1. 在Mybatis的准备期间.查询到当前SQL最新/最旧时间/和总共数量。赋值到Page然后并作为下次生成分页的依据
2. 默认SQL添加基于时间的分割查询,类似于`AND push_time BETWEEN {ts '2021-07-06 10:53:04.7'} AND {ts '2021-10-14 10:53:04.7'} ORDER BY push_time DESC `
3. 假如多数据源的时候进行Page合并.保证一致性的Page数据
4. 返回时候通过这次的结果算出下次推荐的分页时间区间
   
## 推荐下次分页算法逻辑   

具体实现逻辑为
1. 先使用整个查询所有数据的最新实现和最旧时间,算出总共的时间长度
2. 用时间长度/推荐的个数。可以得出每一次返回理想的条数大概的时间。在拿查询时间相减一份时间
比如在查询推送的时候。可得出最老一条时间为`2021-10`,最新一条时间为`2021-08`.总共数量为20,那就(2021-10)-(2021-08))/20(期待每条推荐数)
3. 由于实际上数据不是在时间分布上线性的。肯定是一个正态或者多个正态分布.那么可以技术修正成为按这次查询。假如数据量过多。那么就相应的减少一份的时间间隔。假如数据量太少。则相应的
4. 利用现有的查询数据进行修正.大致逻辑是。修正Rate=1/((这次查询的时间/一份时间)/(这次查询的数量/推荐的数量)),修正Rate*onePiece可得到修正后得到理想的数据
   


丑陋实现版本
```kotlin

    val timeMaxRate=5;
    val timeMinRate=1/timeMaxRate.toDouble()
    private fun getPeriodTimeByShowCount():Long //返回为Seconds
    {
        var seconds=
            (this.newestDateTime?:LocalDateTime.now()).toEpochSecond(ZoneOffset.UTC)-
                    (this.oldestDateTime?:LocalDateTime.now().minusYears(10)).toEpochSecond(ZoneOffset.UTC)

        if(this.recommendShowCount==0L){this.recommendShowCount=10}
        //总数都没有返回0
        if(this.totalCount==0L){return 0L}

        // 理论查询次数= (总数/推荐数量)
        // 一份=总时间/理论查询时间
        var onePiece=(seconds/(this.totalCount/this.recommendShowCount.toDouble())).toLong()
        //假如总量没有
        if(onePiece==0L){return onePiece}

        //要判断下不是第一次跑...
        if(this.queryStartTime!=null&&this.queryEndTime!=null){
            //这次查询总时长,假如查询时长为0则没有修正的必要性
            val querySeconds=Math.abs(this.queryStartTime!!.toEpochSecond(ZoneOffset.UTC)-this.queryEndTime!!.toEpochSecond(ZoneOffset.UTC))
            if(querySeconds==0L){return onePiece}

            //假如数量为空:则返回最多5倍的
            if(this.showCount==0L){return timeMaxRate*onePiece}


            //修正查询时间=(查询时间/一份时间)
            val fixedOnePieceRate=querySeconds/onePiece
            //修正查询数量(当前)
            val fixedShowCount=this.showCount/this.recommendShowCount.toDouble()
            //当前数量
            var rate= (1/(fixedShowCount/fixedOnePieceRate))
            //限制最高和最小(5,1/5)
            rate=if(rate>5){timeMaxRate.toDouble()}else{rate}
            rate=if(rate<timeMinRate){timeMinRate}else{rate}
            onePiece=(onePiece*rate).toLong()
        }
        return onePiece
    }

```


# 优势
多个数据的时候可以实现完美分页. 时间是天然的多数据源分割的利器   
由于大部分逻辑通过组件实现。其实后端人员的心智负担并没有添加多少


# 缺点
查询数据时候需要查询最新和最老时间以及总数。后面查询可以缓存。但是第一次查询必然会很慢。要及时优化索引。
返回数据并不稳定有。有一定概率为空。所以需要前端做更多的兼容性
多数据源的时候没办法自动生成理想的查询时间间隔,需要数据源汇总后重新计算应该的时间间隔.意思是需要调用两次

# 优化点
更新机制    


#   具体细节实现
由于有些地方写的不是很好。所以暂时就不开源了。想要的可以联系我的。这里挑几个点节点说下

#   TimePage结构分析
```kotlin
    class TimePage<T>:Serializable {
    //query参数 开始时间和结束时间
    var queryStartTime: LocalDateTime?=null
    var queryEndTime:LocalDateTime?=null
    //推荐查询数量。后面有用到
    var recommendShowCount:Long=10
    //决定是从新到老还是老到新
    var isAsc:Boolean=false  
    //数据库里面查询最老的时间，最新的时间和总体数量
    var oldestDateTime:LocalDateTime?=null
    var newestDateTime:LocalDateTime?=null
    var totalCount:Long=0L
    //设置records后这边的赋值。给前端用。
    var showCount:Long=0
    var hasNext:Boolean=false
    var records:List<T> =arrayListOf()
}
```


# 具体代码.仅供参考
```kotlin
   @Bean
   @Order(20)
    fun timePageInterceptor(): TimePageInterceptor {
        return TimePageInterceptor()
    }


    @Intercepts(Signature(type = StatementHandler::class, method = "prepare", args = [java.sql.Connection::class, java.lang.Integer::class]))
class TimePageInterceptor:SqlParserHandler(), Interceptor {
    val logger= LoggerFactory.getLogger(TimePageInterceptor::class.java)

    private val sqlParser: ISqlParser? = null
    private var dialectType: String? = null
    private var dialectClazz: String? = null
    private var localPage=false

    override fun intercept(invocation: Invocation?): Any? {
        val statementHandler = PluginUtils.realTarget(invocation!!.target) as StatementHandler
        val metaObject = SystemMetaObject.forObject(statementHandler)
        //this.sqlParser(metaObject)
        val mappedStatement = metaObject.getValue("delegate.mappedStatement") as MappedStatement
        if (SqlCommandType.SELECT != mappedStatement.sqlCommandType) {
            return invocation!!.proceed()
        }
        val boundSql = metaObject.getValue("delegate.boundSql") as BoundSql
        val paramObj = boundSql.parameterObject
        // 判断参数里是否有page对象

        // 判断参数里是否有page对象
        var key="create_time"
        var page: TimePage<*>? = null
        if (paramObj is TimePage<*>) {
            page = paramObj
        } else if (paramObj is Map<*, *>) {
            for (arg in paramObj) {
                if (arg.value is TimePage<*>) {
                    page = arg.value as TimePage<*>
                    key=arg.key as String
                    break
                }
            }
        }

        if(page==null){
            return invocation!!.proceed()
        }
        if((page.queryStartTime==null||page.queryEndTime==null) && !page.autoGenerateQueryTime)
        {
                return invocation!!.proceed()
        }
        page is TimePage<*>
        /**
         * 不需要分页的场合
         * timePage->threadGetLocal
         */
        val connection = invocation!!.args[0] as java.sql.Connection
        var originalSql = boundSql.sql
        if(page.oldestDateTime==null||page.newestDateTime==null){
            val sqlInfo=getLastDateTimeSql(originalSql,key?:"")
            queryLastTime(sqlInfo.sql,mappedStatement,boundSql, page, connection)
            //lastDateTime为空
            if(page.oldestDateTime==null)
            {
                //logger.debug("page lastDateTime is empty")
                return invocation!!.proceed()
            }
        }
        //在这里进行处理。。。。
        if(page.autoGenerateQueryTime&&(page.queryStartTime==null||page.queryEndTime==null)){
            page.generateDefaultQueryTime()
        }
        originalSql=buildTimePaginationSql(page,originalSql,key).sql
        /*
         * <p> 禁用内存分页 </p>
         * <p> 内存分页会查询所有结果出来处理（这个很吓人的），如果结果变化频繁这个数据还会不准。</p>
         */
        metaObject.setValue("delegate.boundSql.sql", originalSql)
        metaObject.setValue("delegate.rowBounds.offset", RowBounds.NO_ROW_OFFSET)
        metaObject.setValue("delegate.rowBounds.limit", RowBounds.NO_ROW_LIMIT)
        return invocation!!.proceed()
    }

    fun buildTimePaginationSql(page:TimePage<*>, originalSql:String,key:String):SqlInfo{
        val sqlInfo = SqlInfo.newInstance()
        return try {
            val selectStatement = CCJSqlParserUtil.parse(originalSql) as Select
            val plainSelect = selectStatement.selectBody as PlainSelect
            val where=addWhereWithTime(plainSelect.where,page,key)
            if(where!=null){
                plainSelect.where=where
            }
            plainSelect.orderByElements=addOrderBy(plainSelect.orderByElements,page,key)
            sqlInfo.sql = selectStatement.toString()
            sqlInfo
        } catch (e: Throwable) {
            // 无法优化使用原 SQL
            logger.error("add time page failed:",e)
            sqlInfo.sql = originalSql
            sqlInfo
        }
        return sqlInfo
    }

    //
    fun getLastDateTimeSql(originalSql:String,key:String): SqlInfo {
        val sqlInfo = SqlInfo.newInstance()
        return try {
            val selectStatement = CCJSqlParserUtil.parse(originalSql) as Select
            val plainSelect = selectStatement.selectBody as PlainSelect
            val distinct = plainSelect.distinct
            val groupBy = plainSelect.groupByColumnReferences
            val orderBy = plainSelect.orderByElements

            // 添加包含groupBy 不去除orderBy
            if (CollectionUtils.isEmpty(groupBy) && CollectionUtils.isNotEmpty(orderBy)) {
                plainSelect.orderByElements = null
                sqlInfo.isOrderBy = false
            }
            //#95 Github, selectItems contains #{} ${}, which will be translated to ?, and it may be in a function: power(#{myInt},2)
            for (item in plainSelect.selectItems) {
                if (item.toString().contains("?")) {
                    sqlInfo.sql = SqlUtils.getOriginalCountSql(selectStatement.toString())
                    return sqlInfo
                }
            }
            // 包含 distinct、groupBy不优化
            if (distinct != null || CollectionUtils.isNotEmpty(groupBy)) {
                sqlInfo.sql = SqlUtils.getOriginalCountSql(selectStatement.toString())
                return sqlInfo
            }
            // 优化 SQL
            plainSelect.selectItems = getSelectItems(key)
            plainSelect.limit=null
            sqlInfo.sql = selectStatement.toString()
            sqlInfo
        } catch (e: Throwable) {
            // 无法优化使用原 SQL
            sqlInfo.sql = SqlUtils.getOriginalCountSql(originalSql)
            sqlInfo
        }
    }

    fun  getSelectItems(key:String):List<SelectItem>{
        val selectElect= Column("$key")
        val expressionList = ExpressionList()
        val expressions:List<Expression> =arrayListOf<Expression>(selectElect);
        expressionList.expressions=expressions
        val minDate=net.sf.jsqlparser.expression.Function()
        minDate.name = "min"
        minDate.parameters=expressionList

        val maxDate=Function()
        maxDate.name="max"
        maxDate.parameters=ExpressionList().apply{this.expressions=arrayListOf<Expression>(Column("$key"))}

        val count=Function()
        count.name="count"
        count.parameters=ExpressionList().apply{
            this.expressions=arrayListOf<Expression>(LongValue(1))
        }

        return arrayListOf(SelectExpressionItem(minDate),SelectExpressionItem(maxDate),SelectExpressionItem(count))
    }


    //cachePageBy應該是有問題的。需要參數也村下去。具體怎麼做看下
    fun queryLastTime(sql:String,mappedStatement:MappedStatement, boundSql:BoundSql, page:TimePage<*>, connection:java.sql.Connection)
    {
        if(page.cachePage)
        {
            page.records= arrayListOf();
            var sql_=sql+(boundSql.parameterObject as Map<String,Any?>).filter{it.key.contains("param")}.filter{it.value !is TimePage<*>}.map{it.value.toString()}.joinToString(",")
            sql_=URLEncoder.encode(sql_,"UTF-8")
            val forceRefresh=page.autoGenerateQueryTime
            logger.debug("force ${forceRefresh} refresh page sql_:${sql_}")
            //
            var page_=cache(sql_,forceRefresh=forceRefresh)
            //缓存时间部分。复用平台框架所以有时间开源的
            page_= page_?:queryLastTime_(sql,mappedStatement,boundSql,page,connection) 
            page.oldestDateTime=page_.oldestDateTime
            page.newestDateTime=page_.newestDateTime
            page.totalCount=page_.totalCount
        }
        else{
           queryLastTime_(sql,mappedStatement,boundSql,page,connection)
        }
    }

    //sql本地缓存
    fun queryLastTime_(sql:String, mappedStatement:MappedStatement, boundSql:BoundSql, page:TimePage<*>, connection:java.sql.Connection):TimePage<*>{
        try {
            connection.prepareStatement(sql).use {
                    statement ->
                val parameterHandler: DefaultParameterHandler =
                    MybatisDefaultParameterHandler(mappedStatement, boundSql.parameterObject, boundSql)
                parameterHandler.setParameters(statement)
                var lastDateTime: LocalDateTime = LocalDateTime.now()
                var newestDateTime: LocalDateTime = LocalDateTime.now()
                var totalNum=0L
                statement.executeQuery().use { resultSet ->
                    if (resultSet.next()) {
                        //应该是获得相应的类型
                        lastDateTime = resultSet.getTimestamp(1)?.toLocalDateTime()?:LocalDateTime.now()
                        newestDateTime = resultSet.getTimestamp(2)?.toLocalDateTime()?:LocalDateTime.now()
                        totalNum=resultSet.getLong(3)
                    }
                }
                page.oldestDateTime=lastDateTime
                page.newestDateTime=newestDateTime
                page.totalCount=totalNum
            }
            return page
        } catch (e: Exception) {
            throw MybatisPlusException("Error: Method queryTime Error ${sql}", e)
        }
        return page
    }

    override fun plugin(target: Any?): Any {
        if(target is StatementHandler)
        {
            return Plugin.wrap(target,this)
        }
        return target!!
    }

    override fun setProperties(prop: Properties) {
        val dialectType: String = prop.getProperty("dialectType")
        val dialectClazz: String = prop.getProperty("dialectClazz")
        val localPage: String = prop.getProperty("localPage")

        if (StringUtils.isNotEmpty(dialectType)) {
            this.dialectType = dialectType
        }
        if (StringUtils.isNotEmpty(dialectClazz)) {
            this.dialectClazz = dialectClazz
        }
        if (StringUtils.isNotEmpty(localPage)) {
            this.localPage = Boolean.valueOf(localPage)
        }
    }

    fun addWhereWithTime(where:Expression?, page:TimePage<*>,key:String):Expression?{
        //orderBy
        var reverse=false
        if(page.queryStartTime==null||page.queryEndTime==null){
            return where
        }
        if(page.queryStartTime!!>page.queryEndTime!!){reverse=true}
        val timePack:Pair<Expression,Expression> =if(page.isDate){
            Pair(page.queryStartTime!!.toDateValue(),page.queryEndTime!!.toDateValue())
        }else{
            Pair(page.queryStartTime!!.toDateTimeValue(),page.queryEndTime!!.toDateTimeValue())
        }

        val between=Between().apply{
            this.betweenExpressionStart=if(reverse){timePack.second!!}else{timePack.first!!}
            this.betweenExpressionEnd=if(reverse){timePack.first!!}else{timePack.second!!}
            this.leftExpression=Column(key)
        }
        if(where==null)
        {
            return between
        }else{
          return AndExpression(where,between)
        }
    }

    fun addOrderBy(orderBy:List<OrderByElement>?,page:TimePage<*>,key:String):List<OrderByElement>{
        val orderByElement=OrderByElement().apply{
            this.isAsc=page.isAsc
            this.expression=Column(key)
        }
        //有先這個
        if(orderBy==null)
        {
            return arrayListOf(orderByElement)
        }
        else{
            return arrayListOf<OrderByElement>(orderByElement,*orderBy.toTypedArray())
        }
    }


    fun LocalDateTime.toDateValue():DateValue
    {
        val timeStr=this.format(DateTimeFormatter.ofPattern("YYYY-MM-dd"))
        return DateValue("\'${timeStr}\'")
    }

    fun LocalDateTime.toDateTimeValue():TimestampValue
    {
        val timeStr=this.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss.SSSSSS"))
        return TimestampValue("\'${timeStr}\'")
    }
}
```


```kotlin
val logger= LoggerFactory.getLogger(TimePage::class.java)
class TimePage<T:Comparator<T>>:Serializable {
    //query参数
    var queryStartTime: LocalDateTime?=null
    var queryEndTime:LocalDateTime?=null
    //这边就是
    var recommendShowCount:Long=10

    var isDate:Boolean =false
    var isAsc:Boolean=false
    //key假如有annotation就不要了
    var cachePage:Boolean=false
    var autoGenerateQueryTime:Boolean=false

    //返回参数
    var oldestDateTime:LocalDateTime?=null
    var newestDateTime:LocalDateTime?=null
    var totalCount:Long=0L

    //展示参数
    var showCount:Long=0
    var hasNext:Boolean=false

    constructor(
        queryStartTime:LocalDateTime?=null,
        queryEndTime:LocalDateTime?=null,
        isDate:Boolean=false,
        pageCache:Boolean=false,
        autoGenerateQuery:Boolean=false
    )
    {
        this.queryStartTime=queryStartTime
        this.queryEndTime=queryEndTime
        this.isDate=isDate
        this.cachePage=pageCache
        this.autoGenerateQueryTime=autoGenerateQuery

    }
    constructor()
    {

    }
    var records:List<T> =arrayListOf()
    set(values)
    {
        field=values
        this.showCount=values.size.toLong()
        //逆序并最后时间小
        if(this.queryEndTime !=null&&oldestDateTime!=null&&newestDateTime!=null){
            //
            if(this.queryEndTime!!.isAfter(oldestDateTime) && !this.isAsc){
                this.hasNext=true
            }
            if(this.queryEndTime!!.isBefore(newestDateTime) && this.isAsc){
                this.hasNext=true
            }
        }
    }


    fun getRecommendQueryTime():Pair<LocalDateTime, LocalDateTime> {
        val onePiece = getPeriodTimeByShowCount()
        val firstEndTime=if(!this.isAsc){this.newestDateTime?:LocalDateTime.now()}else{this.oldestDateTime?:LocalDateTime.now().minusYears(3)}
         var queryEndTime=this.queryEndTime?:firstEndTime
         queryEndTime=if(!this.isAsc){
             if(queryEndTime>firstEndTime){firstEndTime}
             else {
                 queryEndTime
             }
         }else{
             if(queryEndTime<firstEndTime){firstEndTime}else {
                 queryEndTime
             }
         }
        //不能这样写
        logger.debug("queryEndTime:${queryEndTime} firstEndTime:${firstEndTime} onepice:${onePiece} asc:${this.isAsc}")
        return Pair(queryEndTime, countTimeByOrder(queryEndTime, onePiece, this.isAsc))
    }


    //通过recommendShowCount算出来
    // (          1分=(结束时间-开始时间)/(总数/10)    )
    // (          getCount/(showCount/lastFix)(比如20/10)  一份/偏移量 (20/10)                  )
    // (          记得Date应该要加1            )
    //     开始时间( queryBeginTime or newest)-（by desc) 一份/偏移量)
    val timeMaxRate=5;
    val timeMinRate=1/timeMaxRate.toDouble()
    private fun getPeriodTimeByShowCount():Long //返回为Seconds
    {
        var seconds=
            (this.newestDateTime?:LocalDateTime.now()).toEpochSecond(ZoneOffset.UTC)-
                    (this.oldestDateTime?:LocalDateTime.now().minusYears(2)).toEpochSecond(ZoneOffset.UTC)
        if(this.recommendShowCount==0L){this.recommendShowCount=10}
        //总数都没有返回一个屁
        if(this.totalCount==0L){return 0L}

        // 理论查询次数= (总数/推荐数量)
        // 一份=总时间/理论查询时间
        var onePiece=(seconds/(this.totalCount/this.recommendShowCount.toDouble())).toLong()
        //假如总量没有
        if(onePiece==0L){return onePiece}

        //要判断下不是第一次跑...
        if(this.queryStartTime!=null&&this.queryEndTime!=null){
            //这次查询总时长,假如查询时长为0则没有修正的必要性
            val querySeconds=Math.abs(this.queryStartTime!!.toEpochSecond(ZoneOffset.UTC)-this.queryEndTime!!.toEpochSecond(ZoneOffset.UTC))
            if(querySeconds==0L){return onePiece}

            //假如数量为空:则返回最多5倍的
            if(this.showCount==0L){return timeMaxRate*onePiece}


            //修正查询时间=(查询时间/一份时间)
            val fixedOnePieceRate=querySeconds/onePiece
            //修正查询数量(当前)
            val fixedShowCount=this.showCount/this.recommendShowCount.toDouble()
            //当前数量
            var rate= (1/(fixedShowCount/fixedOnePieceRate))
            //限制最高和最小(5,1/5)
            rate=if(rate>5){timeMaxRate.toDouble()}else{rate}
            rate=if(rate<timeMinRate){timeMinRate}else{rate}
            onePiece=(onePiece*rate).toLong()
        }
        return onePiece
    }

    private fun countTimeByOrder(originTime:LocalDateTime,onePiece:Long,isAsc:Boolean):LocalDateTime {
        if(isAsc){
            return originTime.plusSeconds(onePiece)
        }
        else{
            return originTime.plusSeconds(0-onePiece)
        }
    }


    //假如两边为0/null->通过recommendShowCount算出来 queryStartTime->大
    fun generateDefaultQueryTime()
    {
        if((this.queryStartTime==null||this.queryEndTime==null)&&this.totalCount!=0L)
        {
            val onePiece=getPeriodTimeByShowCount()
            var defaultQueryBeginTime=if(!this.isAsc){this.newestDateTime?:LocalDateTime.now()}else{this.oldestDateTime?:LocalDateTime.now().minusYears(3)}
            var defaultQueryEndTime=countTimeByOrder(defaultQueryBeginTime, onePiece, this.isAsc)
            this.queryStartTime=defaultQueryBeginTime
            this.queryEndTime=defaultQueryEndTime
        }
    }


}

fun<T:Comparator<T>,V:Comparator<V>> TimePage<T>.transform(pageCache:Boolean? =null):TimePage<V>
{
    val timePage=TimePage<V>()
    timePage.queryStartTime=this.queryStartTime
    timePage.queryEndTime=this.queryEndTime
    timePage.isDate=this.isDate;
    timePage.isAsc=this.isAsc
    timePage.totalCount=this.totalCount
    timePage.oldestDateTime=this.oldestDateTime;
    timePage.autoGenerateQueryTime=this.autoGenerateQueryTime
    timePage.newestDateTime=this.newestDateTime;
    if(!(pageCache==null))
    {
        timePage.cachePage=pageCache
    }
    //false true false
    return timePage
}

inline fun <reified T:Comparator<T>> Common.TimePageMessage.toMybatisTimePage(): TimePage<T> {
    val timePage=TimePage<T>()
    val queryStartTime_:LocalDateTime?=if(this.queryStartTime<=0L){null}else{this.queryStartTime.toLocalDateTime()}
    val queryEndTime_:LocalDateTime?=if(this.queryEndTime<=0L){null}else{this.queryEndTime.toLocalDateTime()}
    val recommendCount:Long=if(this.recommendQueryCount==0L){30}else{this.recommendQueryCount}
    timePage.recommendShowCount=recommendCount
    timePage.queryStartTime=queryStartTime_
    timePage.queryEndTime=queryEndTime_
    timePage.isDate=this.isDate;
    timePage.isAsc=this.isAsc
    timePage.autoGenerateQueryTime=this.autoGenerateQuery
    return timePage
}

inline fun <reified T:Comparator<T>> TimePage<T>.toMessage(): Common.TimePageMessage {
    val timePageMessage= Common.TimePageMessage.newBuilder()
    timePageMessage.isAsc=this.isAsc
    timePageMessage.hasNext=this.hasNext
    timePageMessage.queryShowCount=this.showCount?.toLong()?:0L
    timePageMessage.totalCount=this.totalCount
    timePageMessage.recommendQueryCount=this.recommendShowCount
    timePageMessage.endTime=this.oldestDateTime?.toLongValue()?:0
    timePageMessage.newestTime=this.newestDateTime?.toLongValue()?:0
    timePageMessage.autoGenerateQuery=this.autoGenerateQueryTime

    queryStartTime?.let{
        timePageMessage.queryStartTime=this.queryStartTime!!.toLongValue()
    }
    queryEndTime?.let{
        timePageMessage.queryEndTime=this.queryEndTime!!.toLongValue()
    }
    val (recommendStartTime,recommendEndTime)=this.getRecommendQueryTime()
    timePageMessage.recommendQueryNextBeginTime=recommendStartTime.toLongValue()
    timePageMessage.recommendQueryNextEndTime=recommendEndTime.toLongValue()
    return timePageMessage.build()
}



fun<T:Comparator<T>> List<TimePage<T>>.merge():TimePage<T>
{
    val timePage=TimePage<T>()
    timePage.queryStartTime=this.first().queryStartTime
    timePage.queryEndTime=this.first().queryEndTime
    timePage.isDate=this.first().isDate;
    timePage.isAsc=this.first().isAsc
    //timePage.key=this.first().key;
    timePage.oldestDateTime=(this.mapNotNull { it.oldestDateTime }.sortedBy { it }.firstOrNull()?:LocalDateTime.now().minusYears(3))
    timePage.newestDateTime=(this.mapNotNull { it.newestDateTime }.sortedByDescending { it }.firstOrNull()?:LocalDateTime.now())
    //totalCount must not b zro
    timePage.totalCount=this.map { it.totalCount?:0 }.fold(0L){x,y->x+y}
    timePage.records=this.flatMap { it.records }.sortedWith( Comparator{x,y->x.compare(x,y)} )
    return timePage
}
```
