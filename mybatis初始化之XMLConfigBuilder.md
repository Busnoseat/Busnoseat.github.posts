---
title: mybatis初始化之XMLConfigBuilder
date: 2020-06-02 11:00:25
tags: mybatis
---

在 MyBatis 初始化过程中，会加载 mybatis-config.xml 配置文件。而这个步骤的入口是 XMLConfigBuilder。
<!--more-->

# parse方法
```
// XMLConfigBuilder.java

    public Configuration parse() {
            // 若已解析，抛出 BuilderException 异常
        if(this.parsed) {
            throw new BuilderException("Each XMLConfigBuilder can only be used once.");
        } else {
            // 标记已解析
            this.parsed = true;
            // 解析 XML configuration 节点
            this.parseConfiguration(this.parser.evalNode("/configuration"));
            return this.configuration;
        }
    }

```

# parseConfiguration

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

## settingsAsProperties

这是 MyBatis 中极为重要的调整设置，它们会改变 MyBatis 的运行时行为，一个配置完整的 settings 元素的示例如下
 ```
<settings>
  <setting name="cacheEnabled" value="true"/>
  <setting name="lazyLoadingEnabled" value="true"/>
  <setting name="multipleResultSetsEnabled" value="true"/>
  <setting name="useColumnLabel" value="true"/>
  <setting name="useGeneratedKeys" value="false"/>
  <setting name="autoMappingBehavior" value="PARTIAL"/>
  <setting name="autoMappingUnknownColumnBehavior" value="WARNING"/>
  <setting name="defaultExecutorType" value="SIMPLE"/>
  <setting name="defaultStatementTimeout" value="25"/>
  <setting name="defaultFetchSize" value="100"/>
  <setting name="safeRowBoundsEnabled" value="false"/>
  <setting name="mapUnderscoreToCamelCase" value="false"/>
  <setting name="localCacheScope" value="SESSION"/>
  <setting name="jdbcTypeForNull" value="OTHER"/>
  <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode,toString"/>
</settings>

 ```
解析setting的源码如下：
 ```
 // XMLConfigBuilder.java
private Properties settingsAsProperties(XNode context) {
    // 将子标签，解析成 Properties 对象
    if (context == null) {
        return new Properties();
    }
    Properties props = context.getChildrenAsProperties();
    // Check that all settings are known to the configuration class
    // 校验每个属性，在 Configuration 中，有相应的 setting 方法，否则抛出 BuilderException 异常
    MetaClass metaConfig = MetaClass.forClass(Configuration.class, localReflectorFactory);
    for (Object key : props.keySet()) {
        if (!metaConfig.hasSetter(String.valueOf(key))) {
            throw new BuilderException("The setting " + key + " is not known.  Make sure you spelled it correctly (case sensitive).");
        }
    }
    return props;
}
 ```

 ## propertiesElement
 