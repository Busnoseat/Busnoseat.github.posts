---
title: mybatis的原理
date: 2021-04-22 22:37:48
tags: mybatis
---

mybatis虽然一直用，但是执行过程一直不是很理解，最近在几位大佬的博客里学习并整理以下内容。<!--more-->mybatis总的来说执行流程总共分为3个步骤
* 解析配置文件
* 创建sqlsession对象
* 调用SqlSession中的api，传入Statement Id和参数，底层最后调用jdbc执行SQL语句，封装结果返回


## 使用方式
### 传统工作模式
用最原始的方式，摒除了各种包装，也最能体现mybatis最初的实现流程。
```
public static void main(String[] args) {
// 解析配置文件
InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(inputStream);

// 获取sqlsession对象
SqlSession sqlSession = factory.openSession();
String name = "tom";

// 调用sqlsession的api
List<User> list = sqlSession.selectList("com.demo.mapper.UserMapper.getUserByName",params);
}
```


### 使用Mapper接口
```
public static void main(String[] args) {

//前三步都相同
InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(inputStream);
SqlSession sqlSession = factory.openSession();

//这里不再调用SqlSession 的api，而是获得了接口对象，调用接口中的方法。
UserMapper mapper = sqlSession.getMapper(UserMapper.class);
List<User> list = mapper.getUserByName("tom");
}
```
由于面向接口编程的趋势，MyBatis也实现了通过接口调用mapper配置文件中的SQL语句，相比较传统工作模式做了些许封装。获取到sqlsession后，调用getMapper接口，通过动态代理，统一生成MapperProxy对象，并调用exectuor执行语句，executor执行语句最终也是调用sqlsession的api， 传入Statement Id和参数，内部进行复杂的处理，最后调用jdbc执行SQL语句，封装结果返回。


### spring整合mybatis

## 初始化配置文件
初始化配置文件的本质是创建configuration对象，将解析的xml数据封装到Configuration内部的属性中。xml中的标签有**properties（属性），settings（设置），typeAliases（类型别名），typeHandlers（类型处理器），objectFactory（对象工厂），mappers（映射器）**等，Configuration也有对应的对象属性来封装它们。

### 解析的内容
```
// XMLConfigBuilder.java

   private void parseConfiguration(XNode root) {
        try {
            // 解析 <settings /> 标签
            Properties e = this.settingsAsPropertiess(root.evalNode("settings"));
            // 解析 <properties /> 标签
            this.propertiesElement(root.evalNode("properties"));
            // 加载自定义 VFS 实现类
            this.loadCustomVfs(e);
            // 解析 <typeAliases /> 标签
            this.typeAliasesElement(root.evalNode("typeAliases"));
            // 解析 <plugins /> 标签
            this.pluginElement(root.evalNode("plugins"));
            // 解析 <objectFactory /> 标签
            this.objectFactoryElement(root.evalNode("objectFactory"));
            // 解析 <objectWrapperFactory /> 标签
            this.objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
            // 解析 <reflectorFactory /> 标签
            this.reflectionFactoryElement(root.evalNode("reflectionFactory"));
            // 赋值 <settings /> 到 Configuration 属性
            this.settingsElement(e);
            // 解析 <environments /> 标签
            this.environmentsElement(root.evalNode("environments"));
            // 解析 <databaseIdProvider /> 标签
            this.databaseIdProviderElement(root.evalNode("databaseIdProvider"));
            // 解析 <typeHandlers /> 标签
            this.typeHandlerElement(root.evalNode("typeHandlers"));
            // 解析 <mappers /> 标签
            this.mapperElement(root.evalNode("mappers"));
        } catch (Exception var3) {
            throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + var3, var3);
        }
    }
 ```

### 传统模式里入口
SqlsessionFactoryBuilder调用build的时候，会生成XMLConfigBuilder。然后XMLConfigBuilder调用parse方法解析配置文件
```
// SqlSessionFactoryBuilder.java

 public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
        SqlSessionFactory var5;
        try {
            XMLConfigBuilder e = new XMLConfigBuilder(reader, environment, properties);
            var5 = this.build(e.parse());
        ...
    }
```
### spring整合mybatis的入口
SqlSessionFactoryBean实现InitializingBean接口，所以项目启动后，当所有bean创建完成会自动调用SqlSessionFactoryBean的afterPropertiesSet，里面就有生成XMLConfigBuilder。然后XMLConfigBuilder调用parse方法解析配置文件

```
// SqlSessionFactoryBean.java

   public void afterPropertiesSet() throws Exception {
        ...
        this.sqlSessionFactory = this.buildSqlSessionFactory();
    }

    protected SqlSessionFactory buildSqlSessionFactory() throws IOException {
        XMLConfigBuilder xmlConfigBuilder = null;
        if(xmlConfigBuilder != null) {
            try {
                xmlConfigBuilder.parse();
                if(LOGGER.isDebugEnabled()) {
                    LOGGER.debug("Parsed configuration file: \'" + this.configLocation + "\'");
                }
            } catch (Exception var22) {
                throw new NestedIOException("Failed to parse config resource: " + this.configLocation, var22);
            } finally {
                ErrorContext.instance().reset();
            }
        }

       ....

```

## 执行流程
### 获取mapper接口
sqlsession调用getMapper的时序图如下：
![image](/asset/article/20210424/1.png)
* 在调用getMapper之后，会去Configuration对象中获取Mapper对象，因为在项目启动的时候就会把Mapper接口加载并解析存储到Configuration对象
```
DefaultSqlSession.java
  public <T> T getMapper(Class<T> type) {
        return this.configuration.getMapper(type, this);
    }
```
* 通过Configuration对象中的MapperRegistry对象属性，继续调用getMapper方法
```
Configuration.java
    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        return this.mapperRegistry.getMapper(type, sqlSession);
    }
```
* 根据type类型，从MapperRegistry对象中的knownMappers获取到当前类型对应的代理工厂类，然后通过代理工厂类生成对应Mapper的代理类
```
MapperRegister.java
    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        MapperProxyFactory mapperProxyFactory = (MapperProxyFactory)this.knownMappers.get(type);
        if(mapperProxyFactory == null) {
            throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
        } else {
            try {
                return mapperProxyFactory.newInstance(sqlSession);
            } catch (Exception var5) {
                throw new BindingException("Error getting mapper instance. Cause: " + var5, var5);
            }
        }
    }
```
* 最终获取到我们接口对应的代理类MapperProxy对象,而MapperProxy可以看到实现了InvocationHandler，使用的就是JDK动态代理。
```
MapperProxyFactory.java
    protected T newInstance(MapperProxy<T> mapperProxy) {
        return Proxy.newProxyInstance(this.mapperInterface.getClassLoader(), new Class[]{this.mapperInterface}, mapperProxy);
    }

    public T newInstance(SqlSession sqlSession) {
        MapperProxy mapperProxy = new MapperProxy(sqlSession, this.mapperInterface, this.methodCache);
        return this.newInstance(mapperProxy);
    }
```

### 寻找sql
* sql语句的时序图
![image](/asset/article/20210424/2.png)

* 调用被代理对象的方法之后实际上执行的就是代理对象的invoke方法
```
MapperProxy.java
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        //如果调用的方法是继承自object的 那么直接调用即可 如toString()
        if(Object.class.equals(method.getDeclaringClass())) {
            try {
                return method.invoke(this, args);
            } catch (Throwable var5) {
                throw ExceptionUtil.unwrapThrowable(var5);
            }
        } else {
        //否则如果是mapper里自定义的方法，则走这个分支
            MapperMethod mapperMethod = this.cachedMapperMethod(method);
            return mapperMethod.execute(this.sqlSession, args);
        }
    }
 ```   


 * 接下来，是构造一个MapperMethod对象,这个对象封装了Mapper接口中对应的方法信息以及对应的sql语句信息

 ```  
 public class MapperMethod {
    //记录sql语句以及语句类型
    private final MapperMethod.SqlCommand command;
    //记录mapper接口中方法信息
    private final MapperMethod.MethodSignature method;
 ```      

 ### 执行sql
 * 执行sql的语句图
 ![image](/asset/article/20210424/3.png)

 * MapperMethod的execute方法会根据sql的语句类型和返回值类型选择不同的执行方法
 ![image](/asset/article/20210424/4.png)

 * 绕了一圈 还是用SqlSession封装好的接口
 ![image](/asset/article/20210424/5.png)

 * sqlsession并不直接操作数据库，mybatis对数据库的操作统一有执行器executor来完成
 ![image](/asset/article/20210424/6.png)

 * executor参数准备完毕后，调用doQuery方法开始查询数据库，这里面代码几乎和手写jdbc连接查询数据库是一致的

 ```  
 BatchExecutor.java
     public <E> List<E> doQuery(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
        Statement stmt = null;
        List var10;
        try {
            this.flushStatements();
            Configuration configuration = ms.getConfiguration();
            StatementHandler handler = configuration.newStatementHandler(this.wrapper, ms, parameterObject, rowBounds, resultHandler, boundSql);
            Connection connection = this.getConnection(ms.getStatementLog());
            stmt = handler.prepare(connection, this.transaction.getTimeout());
            //入参设置
            handler.parameterize(stmt);
            var10 = handler.query(stmt, resultHandler);
        } finally {
            this.closeStatement(stmt);
        }

        return var10;
    }
 ```  

* statementHandler调用query前会调用parameterize进行入参设置

* statementHandler的query里，封装PreparedStatement，执行executor查询数据
 ```  
StatementHandler.java
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
   PreparedStatement ps = (PreparedStatement)statement;
   ps.execute();
   return this.resultSetHandler.handleResultSets(ps);
    }
 ```  
* 最后调用ResultSetHandler的handleResultSets来进行出参转换

## 工作流程图
![image](/asset/article/20210424/7.jpg)