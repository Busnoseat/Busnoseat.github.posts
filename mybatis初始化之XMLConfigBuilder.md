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
            this.loadCustomVfs(e);
            this.typeAliasesElement(root.evalNode("typeAliases"));
            this.pluginElement(root.evalNode("plugins"));
            this.objectFactoryElement(root.evalNode("objectFactory"));
            this.objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
            this.reflectionFactoryElement(root.evalNode("reflectionFactory"));
            this.settingsElement(e);
            this.environmentsElement(root.evalNode("environments"));
            this.databaseIdProviderElement(root.evalNode("databaseIdProvider"));
            this.typeHandlerElement(root.evalNode("typeHandlers"));
            this.mapperElement(root.evalNode("mappers"));
        } catch (Exception var3) {
            throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + var3, var3);
        }
    }
 ```