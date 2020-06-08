---
title: mybatis初始化之XMLMapperBuilder
date: 2020-06-03 22:01:13
tags: mybatis
---
MyBatis 初始化的第二步，加载 Mapper 映射配置文件。而这个步骤的入口是 XMLMapperBuilder。
<!--more-->

## parse方法

```
// XMLMapperBuilder.java

public void parse() {
    // <1> 判断当前 Mapper 是否已经加载过
    if (!configuration.isResourceLoaded(resource)) {
        // <2> 解析 `<mapper />` 节点
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
### 1 判断当前 Mapper 是否已经加载过
调用 Configuration#isResourceLoaded(String resource) 方法，判断当前 Mapper 是否已经加载过。

### 2 configurationElement方法解析<mapper>节点
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

#### 2.2 cacheRefElement解析cache-ref节点
```
private void cacheRefElement(XNode context) {
    if (context != null) {
        // <1> 获得指向的 namespace 名字，并添加到 configuration 的 cacheRefMap 中
        configuration.addCacheRef(builderAssistant.getCurrentNamespace(), context.getStringAttribute("namespace"));
        // <2> 创建 CacheRefResolver 对象，并执行解析
        CacheRefResolver cacheRefResolver = new CacheRefResolver(builderAssistant, context.getStringAttribute("namespace"));
        try {
            cacheRefResolver.resolveCacheRef();
        } catch (IncompleteElementException e) {
            // <3> 解析失败，添加到 configuration 的 incompleteCacheRefs 中
            configuration.addIncompleteCacheRef(cacheRefResolver);
        }
    }
}
```

### 3 标记该 Mapper 已经加载过
调用 Configuration#addLoadedResource(String resource) 方法，标记该 Mapper 已经加载过。

### 4 调用 #bindMapperForNamespace() 方法，绑定 Mapper

### 5 解析待定的 <resultMap /> 节点

### 6 解析待定的 <cache-ref /> 节点

### 7 解析待定的 SQL 语句的节点
【相关文献】
本文学自