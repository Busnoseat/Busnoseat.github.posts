---
title: mybatis初始化之XMLMapperBuilder
date: 2020-06-03 22:01:13
tags: mybatis
---
MyBatis 初始化的第二步，加载 Mapper 映射配置文件。而这个步骤的入口是 XMLMapperBuilder。
<!--more-->

# parse方法

```
// XMLMapperBuilder.java

public void parse() {
    // <1> 判断当前 Mapper 是否已经加载过
    if (!configuration.isResourceLoaded(resource)) {
        // <2> 解析 <mapper/> 节点
        configurationElement(parser.evalNode("/mapper"));
        // <3> 标记该 Mapper 已经加载过
        configuration.addLoadedResource(resource);
        // <4> 绑定 Mapper
        bindMapperForNamespace();
    }

    // <5> 解析待定的 <resultMap /> 节点
    parsePendingResultMaps();
    // <6> 解析待定的 <cache-ref /> 节点
    parsePendingCacheRefs();
    // <7> 解析待定的 SQL 语句的节点
    parsePendingStatements();
}
```
## 判断当前 Mapper 是否已经加载过
```
// Configuration.java

// 已加载资源( Resource )集合
protected final Set<String> loadedResources = new HashSet<>();

public boolean isResourceLoaded(String resource) {
    return loadedResources.contains(resource);
}
```

调用 Configuration#isResourceLoaded(String resource) 方法，判断当前 Mapper 是否已经加载过。

## configurationElement方法解析<mapper>节点
```
// XMLMapperBuilder.java

private void configurationElement(XNode context) {
    try {
        // <2.1> 获得 namespace 属性
        String namespace = context.getStringAttribute("namespace");
        if (namespace == null || namespace.equals("")) {
            throw new BuilderException("Mapper's namespace cannot be empty");
        }
        // <2.1> 设置 namespace 属性
        builderAssistant.setCurrentNamespace(namespace);
        // <2.2> 解析 <cache-ref /> 节点
        cacheRefElement(context.evalNode("cache-ref"));
        // <2.3> 解析 <cache /> 节点
        cacheElement(context.evalNode("cache"));
        // 已废弃！老式风格的参数映射。内联参数是首选,这个元素可能在将来被移除，这里不会记录。
        parameterMapElement(context.evalNodes("/mapper/parameterMap"));
        // <2.4> 解析 <resultMap /> 节点们
        resultMapElements(context.evalNodes("/mapper/resultMap"));
        // <2.5> 解析 <sql /> 节点们
        sqlElement(context.evalNodes("/mapper/sql"));
        // <2.6> 解析 <select /> <insert /> <update /> <delete /> 节点们
        buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
}
```

### cacheRefElement解析`cache-ref`节点
示例
```
<cache-ref namespace="com.someone.application.data.SomeMapper"/>
```

```
// XMLMapperBuilder.java
private void cacheRefElement(XNode context) {
    if (context != null) {
        // <2.2.1> 获得指向的 namespace 名字，并添加到 configuration 的 cacheRefMap 中
        configuration.addCacheRef(builderAssistant.getCurrentNamespace(), context.getStringAttribute("namespace"));
        // <2.2.2> 创建 CacheRefResolver 对象，并执行解析
        CacheRefResolver cacheRefResolver = new CacheRefResolver(builderAssistant, context.getStringAttribute("namespace"));
        try {
            cacheRefResolver.resolveCacheRef();
        } catch (IncompleteElementException e) {
            // <2.2.3> 解析失败，添加到 configuration 的 incompleteCacheRefs 中
            configuration.addIncompleteCacheRef(cacheRefResolver);
        }
    }
}
```
*   获得指向的 namespace 名字，并调用 Configuration#addCacheRef(String namespace, String referencedNamespace) 方法，添加到 configuration 的 cacheRefMap
*   创建 CacheRefResolver 对象，并调用 CacheRefResolver#resolveCacheRef() 方法，执行解析。方法中，会调用 **MapperBuilderAssistant#useCacheRef(String namespace)** 方法，获得指向的 Cache 对象。
*   解析失败，因为此处指向的 Cache 对象可能未初始化，则先调用 Configuration#addIncompleteCacheRef(CacheRefResolver incompleteCacheRef) 方法，添加到 configuration 的 incompleteCacheRefs

### cacheElement解析`cache`节点
示例
```
// 使用默认缓存
<cache eviction="FIFO" flushInterval="60000"  size="512" readOnly="true"/>

// 使用自定义缓存
<cache type="com.domain.something.MyCustomCache">
  <property name="cacheFile" value="/tmp/my-custom-cache.tmp"/>
</cache>
```
```
// XMLMapperBuilder.java

private void cacheElement(XNode context) throws Exception {
    if (context != null) {
        // <1> 获得负责存储的 Cache 实现类
        String type = context.getStringAttribute("type", "PERPETUAL");
        Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
        // <2> 获得负责过期的 Cache 实现类
        String eviction = context.getStringAttribute("eviction", "LRU");
        Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
        // <3> 获得 flushInterval、size、readWrite、blocking 属性
        Long flushInterval = context.getLongAttribute("flushInterval");
        Integer size = context.getIntAttribute("size");
        boolean readWrite = !context.getBooleanAttribute("readOnly", false);
        boolean blocking = context.getBooleanAttribute("blocking", false);
        // <4> 获得 Properties 属性
        Properties props = context.getChildrenAsProperties();
        // <5> 创建 Cache 对象
        builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
    }
}
```

### 解析resultMap节点们
```
// XMLMapperBuilder.java

// 解析 <resultMap /> 节点们
private void resultMapElements(List<XNode> list) throws Exception {
    // 遍历 <resultMap /> 节点们
    for (XNode resultMapNode : list) {
        try {
            // 处理单个 <resultMap /> 节点
            resultMapElement(resultMapNode);
        } catch (IncompleteElementException e) {
            // ignore, it will be retried
        }
    }
}

// 解析 <resultMap /> 节点
private ResultMap resultMapElement(XNode resultMapNode) throws Exception {
    return resultMapElement(resultMapNode, Collections.<ResultMapping>emptyList());
}

// 解析 <resultMap /> 节点
private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings) throws Exception {
    ErrorContext.instance().activity("processing " + resultMapNode.getValueBasedIdentifier());
    // <1> 获得 id 属性
    String id = resultMapNode.getStringAttribute("id",
            resultMapNode.getValueBasedIdentifier());
    // <1> 获得 type 属性
    String type = resultMapNode.getStringAttribute("type",
            resultMapNode.getStringAttribute("ofType",
                    resultMapNode.getStringAttribute("resultType",
                            resultMapNode.getStringAttribute("javaType"))));
    // <1> 获得 extends 属性
    String extend = resultMapNode.getStringAttribute("extends");
    // <1> 获得 autoMapping 属性
    Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");
    // <1> 解析 type 对应的类
    Class<?> typeClass = resolveClass(type);
    Discriminator discriminator = null;
    // <2> 创建 ResultMapping 集合
    List<ResultMapping> resultMappings = new ArrayList<>();
    resultMappings.addAll(additionalResultMappings);
    // <2> 遍历 <resultMap /> 的子节点
    List<XNode> resultChildren = resultMapNode.getChildren();
    for (XNode resultChild : resultChildren) {
        // <2.1> 处理 <constructor /> 节点
        if ("constructor".equals(resultChild.getName())) {
            processConstructorElement(resultChild, typeClass, resultMappings);
        // <2.2> 处理 <discriminator /> 节点
        } else if ("discriminator".equals(resultChild.getName())) {
            discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
        // <2.3> 处理其它节点
        } else {
            List<ResultFlag> flags = new ArrayList<>();
            if ("id".equals(resultChild.getName())) {
                flags.add(ResultFlag.ID);
            }
            resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
        }
    }
    // <3> 创建 ResultMapResolver 对象，执行解析
    ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, extend, discriminator, resultMappings, autoMapping);
    try {
        return resultMapResolver.resolve();
    } catch (IncompleteElementException e) {
        // <4> 解析失败，添加到 configuration 中
        configuration.addIncompleteResultMap(resultMapResolver);
        throw e;
    }
}
```
#### processConstructorElement方法处理 `<constructor /> `节点

```
// XMLMapperBuilder.java

private void processConstructorElement(XNode resultChild, Class<?> resultType, List<ResultMapping> resultMappings) throws Exception {
    // <1> 遍历 <constructor /> 的子节点们
    List<XNode> argChildren = resultChild.getChildren();
    for (XNode argChild : argChildren) {
        // <2> 获得 ResultFlag 集合
        List<ResultFlag> flags = new ArrayList<>();
        flags.add(ResultFlag.CONSTRUCTOR);
        if ("idArg".equals(argChild.getName())) {
            flags.add(ResultFlag.ID);
        }
        // <3> 将当前子节点构建成 ResultMapping 对象，并添加到 resultMappings 中
        resultMappings.add(buildResultMappingFromContext(argChild, resultType, flags));
    }
}
```

#### processDiscriminatorElement方法处理 `<discriminator />` 节点

```
// XMLMapperBuilder.java

private Discriminator processDiscriminatorElement(XNode context, Class<?> resultType, List<ResultMapping> resultMappings) throws Exception {
    // <1> 解析各种属性
    String column = context.getStringAttribute("column");
    String javaType = context.getStringAttribute("javaType");
    String jdbcType = context.getStringAttribute("jdbcType");
    String typeHandler = context.getStringAttribute("typeHandler");
    // <1> 解析各种属性对应的类
    Class<?> javaTypeClass = resolveClass(javaType);
    Class<? extends TypeHandler<?>> typeHandlerClass = resolveClass(typeHandler);
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
    // <2> 遍历 <discriminator /> 的子节点，解析成 discriminatorMap 集合
    Map<String, String> discriminatorMap = new HashMap<>();
    for (XNode caseChild : context.getChildren()) {
        String value = caseChild.getStringAttribute("value");
       // <2.1> 处，如果是内嵌的 ResultMap 的情况，则调用 #processNestedResultMappings(XNode context, List<ResultMapping> resultMappings) 方法，处理内嵌的 ResultMap 的情况。
        String resultMap = caseChild.getStringAttribute("resultMap", processNestedResultMappings(caseChild, resultMappings)); // <2.1>
        discriminatorMap.put(value, resultMap);
    }
    // <3>调用 MapperBuilderAssistant#buildDiscriminator(...) 方法，创建 Discriminator 对象。详细解析
    return builderAssistant.buildDiscriminator(resultType, column, javaTypeClass, jdbcTypeEnum, typeHandlerClass, discriminatorMap);
}
```

##### processNestedResultMappings解析内嵌的resultMap
```
// XMLMapperBuilder.java

private String processNestedResultMappings(XNode context, List<ResultMapping> resultMappings) throws Exception {
    if ("association".equals(context.getName())
            || "collection".equals(context.getName())
            || "case".equals(context.getName())) {
        if (context.getStringAttribute("select") == null) {
            // 解析，并返回 ResultMap
            ResultMap resultMap = resultMapElement(context, resultMappings);
            return resultMap.getId();
        }
    }
    return null;
}
```
该方法，会“递归”调用 #resultMapElement(XNode context, List<ResultMapping> resultMappings) 方法，处理内嵌的 ResultMap 的情况

#### buildResultMappingFromContext方法处理其他节点
```
// XMLMapperBuilder.java

private ResultMapping buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags) throws Exception {
    // <1> 获得各种属性
    String property;
    if (flags.contains(ResultFlag.CONSTRUCTOR)) {
        property = context.getStringAttribute("name");
    } else {
        property = context.getStringAttribute("property");
    }
    String column = context.getStringAttribute("column");
    String javaType = context.getStringAttribute("javaType");
    String jdbcType = context.getStringAttribute("jdbcType");
    String nestedSelect = context.getStringAttribute("select");
    String nestedResultMap = context.getStringAttribute("resultMap",
            processNestedResultMappings(context, Collections.emptyList()));
    String notNullColumn = context.getStringAttribute("notNullColumn");
    String columnPrefix = context.getStringAttribute("columnPrefix");
    String typeHandler = context.getStringAttribute("typeHandler");
    String resultSet = context.getStringAttribute("resultSet");
    String foreignColumn = context.getStringAttribute("foreignColumn");
    boolean lazy = "lazy".equals(context.getStringAttribute("fetchType", configuration.isLazyLoadingEnabled() ? "lazy" : "eager"));
    // <1> 获得各种属性对应的类
    Class<?> javaTypeClass = resolveClass(javaType);
    Class<? extends TypeHandler<?>> typeHandlerClass = resolveClass(typeHandler);
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
    // <2> 构建 ResultMapping 对象
    return builderAssistant.buildResultMapping(resultType, property, column, javaTypeClass, jdbcTypeEnum, nestedSelect, nestedResultMap, notNullColumn, columnPrefix, typeHandlerClass, flags, resultSet, foreignColumn, lazy);
}
```

#### resultMapResolver.resolve()解析

```
// ResultMapResolver.java

public ResultMap resolve() {
    return assistant.addResultMap(this.id, this.type, this.extend, this.discriminator, this.resultMappings, this.autoMapping);
}
```
在 #resolve() 方法中，会调用 MapperBuilderAssistant#addResultMap(...) 方法，创建 ResultMap 对象。详细解析


### sqlElement(List<XNode>) 解析 `<sql />` 节点们
```
// XMLMapperBuilder.java

private void sqlElement(List<XNode> list) throws Exception {
    if (configuration.getDatabaseId() != null) {
        sqlElement(list, configuration.getDatabaseId());
    }
    sqlElement(list, null);
    // 上面两块代码，可以简写成 sqlElement(list, configuration.getDatabaseId());
}

private void sqlElement(List<XNode> list, String requiredDatabaseId) throws Exception {
    // <1> 遍历所有 <sql /> 节点
    for (XNode context : list) {
        // <2> 获得 databaseId 属性
        String databaseId = context.getStringAttribute("databaseId");
        // <3> 获得完整的 id 属性，格式为 `${namespace}.${id}` 。
        String id = context.getStringAttribute("id");
        id = builderAssistant.applyCurrentNamespace(id, false);
        // <4> 判断 databaseId 是否匹配
        if (databaseIdMatchesCurrent(id, databaseId, requiredDatabaseId)) {
            // <5> 添加到 sqlFragments 中
            sqlFragments.put(id, context);
        }
    }
}
```

#### databaseIdMatchesCurrent方法，判断databaseId是否匹配

```
// XMLMapperBuilder.java

private boolean databaseIdMatchesCurrent(String id, String databaseId, String requiredDatabaseId) {
    // 如果不匹配，则返回 false
    if (requiredDatabaseId != null) {
        return requiredDatabaseId.equals(databaseId);
    } else {
        // 如果未设置 requiredDatabaseId ，但是 databaseId 存在，说明还是不匹配，则返回 false
        // mmp ，写的好绕
        if (databaseId != null) {
            return false;
        }
        // skip this fragment if there is a previous one with a not null databaseId
        // 判断是否已经存在
        if (this.sqlFragments.containsKey(id)) {
            XNode context = this.sqlFragments.get(id);
            // 若存在，则判断原有的 sqlFragment 是否 databaseId 为空。因为，当前 databaseId 为空，这样两者才能匹配。
            return context.getStringAttribute("databaseId") == null;
        }
    }
    return true;
}
```


### buildStatementFromContext(...) 方法，解解析 `<insert/delete/update/select/>` 节点们
```
// XMLMapperBuilder.java

private void buildStatementFromContext(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
        buildStatementFromContext(list, configuration.getDatabaseId());
    }
    buildStatementFromContext(list, null);
    // 上面两块代码，可以简写成 buildStatementFromContext(list, configuration.getDatabaseId());
}

private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    // <1> 遍历 <select /> <insert /> <update /> <delete /> 节点们
    for (XNode context : list) {
        // <1> 创建 XMLStatementBuilder 对象，执行解析
        final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
        try {
            statementParser.parseStatementNode();
        } catch (IncompleteElementException e) {
            // <2> 解析失败，添加到 configuration 中
            configuration.addIncompleteStatement(statementParser);
        }
    }
}
```


## 标记该 Mapper 已经加载过
```
调用 Configuration#addLoadedResource(String resource) 方法，标记该 Mapper 已经加载过。
// Configuration.java

public void addLoadedResource(String resource) {
    loadedResources.add(resource);
}
```

## 调用 #bindMapperForNamespace() 方法，绑定 Mapper

```
// XMLMapperBuilder.java

private void bindMapperForNamespace() {
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
        // <1> 获得 Mapper 映射配置文件对应的 Mapper 接口，实际上类名就是 namespace 。嘿嘿，这个是常识。
        Class<?> boundType = null;
        try {
            boundType = Resources.classForName(namespace);
        } catch (ClassNotFoundException e) {
            //ignore, bound type is not required
        }
        if (boundType != null) {
            // <2> 不存在该 Mapper 接口，则进行添加
            if (!configuration.hasMapper(boundType)) {
                // Spring may not know the real resource name so we set a flag
                // to prevent loading again this resource from the mapper interface
                // look at MapperAnnotationBuilder#loadXmlResource
                // <3> 标记 namespace 已经添加，避免 MapperAnnotationBuilder#loadXmlResource(...) 重复加载
                configuration.addLoadedResource("namespace:" + namespace);
                // <4> 添加到 configuration 中
                configuration.addMapper(boundType);
            }
        }
    }
}
```

## parsePendingResultMaps解析待定的 ` <resultMap /> ` 节点
```
// XMLMapperBuilder.java

private void parsePendingResultMaps() {
    // 获得 ResultMapResolver 集合，并遍历进行处理
    Collection<ResultMapResolver> incompleteResultMaps = configuration.getIncompleteResultMaps();
    synchronized (incompleteResultMaps) {
        Iterator<ResultMapResolver> iter = incompleteResultMaps.iterator();
        while (iter.hasNext()) {
            try {
                // 执行解析
                iter.next().resolve();
                // 移除
                iter.remove();
            } catch (IncompleteElementException e) {
                // ResultMap is still missing a resource...
                // 解析失败，不抛出异常
            }
        }
    }
}
```

## parsePendingCacheRefs解析待定的 ` <cache-ref /> `  节点

```
// XMLMapperBuilder.java

private void parsePendingCacheRefs() {
    // 获得 CacheRefResolver 集合，并遍历进行处理
    Collection<CacheRefResolver> incompleteCacheRefs = configuration.getIncompleteCacheRefs();
    synchronized (incompleteCacheRefs) {
        Iterator<CacheRefResolver> iter = incompleteCacheRefs.iterator();
        while (iter.hasNext()) {
            try {
                // 执行解析
                iter.next().resolveCacheRef();
                // 移除
                iter.remove();
            } catch (IncompleteElementException e) {
                // Cache ref is still missing a resource...
            }
        }
    }
}

```

## 解析待定的 SQL 语句的节点

```
// XMLMapperBuilder.java

private void parsePendingStatements() {
    // 获得 XMLStatementBuilder 集合，并遍历进行处理
    Collection<XMLStatementBuilder> incompleteStatements = configuration.getIncompleteStatements();
    synchronized (incompleteStatements) {
        Iterator<XMLStatementBuilder> iter = incompleteStatements.iterator();
        while (iter.hasNext()) {
            try {
                // 执行解析
                iter.next().parseStatementNode();
                // 移除
                iter.remove();
            } catch (IncompleteElementException e) {
                // Statement is still missing a resource...
            }
        }
    }

 ```
【相关文献】
