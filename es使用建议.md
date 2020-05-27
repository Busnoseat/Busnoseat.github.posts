layout: 
title: es使用建议
date: 2020-05-25 11:40:17
tags: 环境
---

es在使用过程中的的建议
<!--more-->
# linux配置
https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html
## 修改文件句柄数 

### 命令
在启动elasticsearch前通过root用户设置ulimit -n 65536，若要永久生效，设置/etc/security/limits.conf的nofile为65536。

### 说明
Elasticsearch使用大量的文件描述符或文件句柄。文件描述符超限在运行时灾难性的，很可能导致数据丢失。请确保调大运行Elasticsearchd的用户允许打开文件描述符数量到65536或更大。你可以通过各节点的Nodes StatsAPI来检查max_file_descriptions: GET _nodes/stats/process?filter_path=**.max_file_descriptors 

## 关闭swap
### 命令
有三种方式禁用swapping：
* 开启bootstrap.memory_lock
请在config/elasticsearch.yml中添加如下行：
bootstrap.memory_lock: true
Elasticsearch启动之后，你可以通过下面请求输出结果中的mlockall对应的值来验证是否设置成功：
GET _nodes?filter_path=**.mlockall
* 禁用所有的swap文件
通常Elasticsearch是运行在一个独立的机器上，且内存使用是通过JVM来控制的，这可能不需要开启swap。
在Linux操作系统，你可以通过sudo swapoff -a来临时禁用swap。
要永久的停用swap，可以通过编辑/etc/fstab文件，注释文件中所有包含swap单词的行。
在Windows操作系统，你可以通过系统设置->高级->性能->高级->虚拟内存来禁用分页文件。
* 配置swappiness
确保设置vm.swappiness为1。这减少了内核交换的倾向，同时仍然允许在紧急条件下的系统交换。
使用sysctl vm.swappiness可查看当前的配置值,如果不是1,则设置:
echo "vm.swappiness = 1" >> /etc/sysctl.conf 或者 sudo sysctl vm.swappiness=1

### 说明
大多数操作系统尝试使用尽可能多的内存文件系统缓存和热切换出未使用的应用程序内存，这可能导致部分JVM堆被交换到磁盘上。
对于性能和节点的稳定性交换是非常糟糕的，应该不惜一切代价避免。
它可能导致垃圾收集持续几分钟而不是几毫秒，这可能导致节点响应缓慢，甚至脱离集群。

## 虚拟内存	
### 命令
通过如下命令来增加限制数：sysctl -w vm.max_map_count=262144
若要永久的生效，可以更新/etc/sysctl.conf中的vm.max_map_count。
重启系统后可以通过运行sysctl vm.max_map_count来进行验证。
### 说明
Elasticsearch默认采用hybrid mmapfs / niofs目录来保存索引。
默认的操作系统mmap数限制太小，这可能会导致内存溢出的异常。

## 修改线程限制	
### 命令
通过root用户在启动前使用ulimit -u 2048来设置
或者是在/etc/security/limits.conf 如下设置
```
elasticsearch - nofile 65536
或
* soft nproc 1024000
* hard nproc 1024000
```
重新进入 session 生效

### 说明
Elasticsearch不同类的操作使用不同的线程池。
在必要的时候创建新的线程非常重要，确保elasticsearch用户可以创建的线程数至少为2048。

# es集群设置
https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html

```
$ -Xms16g -Xmx16g
```
通常情况下配置为机器内存的一半左右，另外一半留给 Lucene 的堆外内存。建议不要超过32G 

----
```
$ discovery.zen.ping.unicast.hosts: [ 'XXX:9300','YYYYY:9300']
$ discovery.zen.minimum_master_nodes: 3
```
防止脑裂 discovery.zen.minimum_master_nodes > = ( master 候选节点个数 / 2) + 1

----
```
$ cluster.routing.allocation.disk.threshold_enabled: true
$ cluster.routing.allocation.disk.watermark.low: 5gb
$ cluster.routing.allocation.disk.watermark.high: 2gb
```
低水位定义ES将不再分配新分片到该节点的磁盘使用百分比（默认是85%）
高水位定义分配将开始从该节点迁移走分片的磁盘使用百分比（默认是90%）
这两个属性都可以被定义为磁盘使用的百分比（比如“80%”表示80%的磁盘空间已使用，或者说还有20%未使用），
或者最小可用空间大小（比如“20GB”表示该节点还有20GB的可用空间）。

----
```
$ index.*
$ indices.*
```
索引配置
https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-indices.html
https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.htm

----
```
$ index.unassigned.node_left.delayed_timeout: 5m	
```
由于节点已经离开而成为未分配的副本分片的分配，可以使用此参数动态设置进行延迟，默认为1m

----
```
$ index.routing.allocation.include.{attribute}
$ index.routing.allocation.require.{attribute}
$ index.routing.allocation.exclude.{attribute}
```
分配索引到一个节点，节点的{attribute}至少满足一个逗号分隔的值。
分配索引到一个节点，节点的{attribute}满足所有逗号分隔的值。
分配索引到一个节点，节点的{attribute}不满足任何逗号分隔的值。
可用于冷热数据分离

----
```
$ thread_pool.*
```
各位线程池的参数
https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-threadpool.html

----
```
$ search.default_search_timeout: 30s
```
全局查询超时

----
```
$ action.destructive_requires_name: true
```
禁止使用_all作为索引名来代表所有索引，避免误操作

----
```
$ action.auto_create_index: false
```	
不自定生成索引

----



# 索引建议
## 索引分片数
建议设置为集群节点的整数倍；初始数据导入时副本数设置为0，生产环境副本数建议设置为1。单节点索引分片数建议不要超过3个，每个索引分片推荐10-40GB大小。索引分片数设置后不可以修改，副本数设置后可以修改。

## 一个索引只允许一个type，
不要在一个索引中创建多个type。

## 使用批量请求（_bulk）
将产生比单文档索引请求好得多的性能，写入数据时建议调用批量提交接口，推荐每批量提交5~15MB数据

## 调整refresh interval
在 Elasticsearch 中，写入和打开一个新段的轻量的过程叫做 refresh ，默认情况下每个分片会每秒自动刷新一次。
注意没有refresh的数据，在查询时是查不到的。通过调大refresh时间，可优化插入性能；调小refresh时间，则可插入后尽快查到数据。

## 所有索引中的type都必须事先设置mapping
* keyword：不要分词的文本字段，设置为keyword；需要分词设置为 text
* doc_value：对于keyword类型的字段，如果我们只需要对该字段进行搜索，不需要进行聚合、排序或是使用脚本操作，可以将doc_value设为false，可大幅降节省磁盘空间，一定程度上提升索引速度。
* index：如果我们只需要对某个字段进行聚合、排序或是使用脚本操作，不需要搜索该字段（通常是keyword或者numeric类型的字段），可以将index设为false，可释放该字段倒排索引占用的常驻内存。 
索引的倒排索引占用的内存可通过GET /<INDEX_NAME>/_stats?human=true中的segments.terms_memory查看。
* norms：对搜索评分很有用，若我们不需要计算字段的评分，将该参数设为false，特别是该字段仅用于过滤或聚合。
若索引中存在某个字段启用了norms，无论文档中是否存在该字段，所有文档都会占用N bytes(N为文档数)磁盘空间，因此禁用norms可以释放大量的磁盘空间。
* enable：在一些情况下，我们只需要保存某个字段，不需要对该字段进行搜索、排序、聚合等操作，可以将该字段的"enabled"设为false，同时节省大量内存和磁盘的空间。
* ignore_malformed：碰到一些内容不规范或者格式不对的数据，例如某个IP字段的里出现"UNKNOWN"，某个数字字段出现"-"。如果在这些字段上已经设置了明确的类型，比如"ip"或者"float"，字段中出现了非该类型的值，ES会抛出异常并丢弃整条数据。 在该字段上设置"ignore_malformed"： ture来忽略。
* _source：属于索引的元数据，其中存储了文档原始的JSON内容，会被存储但不会被索引，用于执行fetch请求时返回原始数据。 
当我们不需要获得任何原始数据，只需要对数据进行排序，聚合等计算，或者写入时文档id是手动指定的，通过搜索取到文档id来进一步处理，可以将"_source"设为false来节约大量的磁盘空间。 注意，禁用"_source"后会导致无法使用update，update_by_query，reindex等需要获取原始文档的API，也无法使用高亮功能。

# 查询建议
## 使用框架封装好的ES工具类
## 设置查询读取记录条数和字段
默认的查询请求通常返回排序后的前10条记录，最多一次读取10000条记录，通过from和size参数控制读取记录范围，避免一次读取过多的记录。
通过_source参数可以控制返回字段信息，尽量避免读取大字段。
## 深分页可能引起性能问题
可以的话，尽量限制查询的pageNo不要太大。若一次性要取大量数据的操作，使用scroll Api，ElasticSearchOpTemplate中有支持。
深分页问题：这个问题和elasticsearch的内部原理有关。比如，每页显示10条数据，现在要取第5001页的数据，在分页的时候，elasticsearch需要首先在每一个节点上取出50020的数据（单节点上排序），然后在协调节点上对每一个节点的所有数据进行排序（全局排序），取出排序后在50010到50020的数据，然后返回。这样随着数据量的增大，每次分页时排序的开销会越来越大。
## 复杂的查询尽量少做
如：nested/parent-child等，可以在java中事先处理的尽量在java中处理好再存到ES。

----
# 其他
## 关闭迁移分片
```	
//关闭迁移分片
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.enable": "none"
  }
}
//打开迁移分片
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.enable": "all"
  }
}
//查看设置
GET /_cluster/settings
```	

## 索引变为只读
一般在硬盘不够或节点宕机后出现，错误信息：blocked by: [FORBIDDEN/12/index read-only / allow delete (api)];
```	
//
PUT _settings
{
    "index": {
        "blocks": {
            "read_only_allow_delete": "false"
        }
    }
}
```	

