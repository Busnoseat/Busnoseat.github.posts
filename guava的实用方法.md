---
title: guava的实用方法
date: 2020-06-02 10:54:47
tags: 优化
---

记录平常发现的guava的实用方法，以便于下次可以直接copy
<!--more-->
# list
* newArrayList 初始化
* charactersOf 将字符串转换成字符串集合
* transform 转换新list
* reverse 翻转list
* partition 根据指定size分页

```
    public static void main(String[] args) {
        //将字符串转换成list
        ImmutableList<Character> list = Lists.charactersOf("abbbccac");

        //list初始化
        List<String> newList = Lists.newArrayList();

        //list转换
        System.out.println(newList = Lists.transform(list, str -> String.valueOf(str)));

        //list反转
        System.out.println(newList = Lists.reverse(newList));

        //按照size为2分割
        System.out.println(Lists.partition(newList, 2));
    }
```

# Set
* newHashSet 初始化
* difference 差集
* intersection 交集
* union 并集
* filter 过滤

```
    public static void main(String[] args) {

        Set<String> set1 = Sets.newHashSet("1", "2", "3");

        Set<String> set2 = Sets.newHashSet("2", "3", "4");

        //差集
        System.out.println(Sets.difference(set1, set2));

        //交集
        System.out.println(Sets.intersection(set1, set2));

        //并集
        System.out.println(Sets.union(set1, set2));

        //过滤
        System.out.println(Sets.filter(set2, item -> Integer.valueOf(item) > 3));
    }

```


# table支持二层索引
当我们需要多个索引的数据结构的时候，通常情况下，我们只能用这种丑陋的Map<FirstName, Map<LastName, Person>>来实现。为此Guava提供了一个新的集合类型－Table集合类型，来支持这种数据结构的使用场景
* row(R) 返回Map<C,V>
* rowKeySet 返回Set<R>
* columnKeySet 返回Set<C>
* values 返回Collection的value值

```
 public static void main(String[] args) {
        //Table<Row, Column, Value> 简写Table<R, C, V>
        Table<String, String, Integer> tables = HashBasedTable.create();
        tables.put("A", "语文", 80);
        tables.put("A", "数学", 90);
        tables.put("A", "英语", 100);
        tables.put("B", "语文", 85);
        tables.put("B", "数学", 95);
        tables.put("B", "英语", 100);

        //A的语文成绩是80
        System.out.println(tables.row("A").get("语文"));

        //B的体育成绩未录入 null
        System.out.println(tables.row("B").get("体育"));

        //C的未录入 查看体育成绩不报错 结果为null
        System.out.println(tables.row("C").get("体育"));

        //查看Row  返回Set集合 结果为 ["A","B"]
        System.out.println(JSON.toJSONString(tables.rowKeySet()));

        //查看Column 返回Set集合  结果为 ["英语","数学","语文"]
        System.out.println(JSON.toJSONString(tables.columnKeySet()));

        //查看value 返回Collection 结果为 [100,90,80,100,95,85]
        System.out.println(JSON.toJSONString(tables.values()));
    }
 ```

 # Maps支持快速过滤和转换新map

 * filterEntries 过滤entry
 * filterKeys 根据key过滤
 * filterValues 根据value过滤
 * transformEntries 转换成新map

 ```
     public static void main(String[] args) {
        Map<String,Integer> map=new HashMap<>(3);
        map.put("语文",80);
        map.put("数学",90);
        map.put("英语",100);

        //过滤value大于85的值  结果为 {"英语":100,"数学":90}
        // 方法1 stream提供的
        map.entrySet().stream().filter(item->item.getValue()>85).collect(Collectors.toMap(x->x.getKey(),y->y.getValue()));

        // 方法2 guava提供的
        newMap=Maps.filterEntries(map,item->item.getValue()>85);
        // 同理guava还提供了filterKey方法和filterValue方法
        newMap=Maps.filterKeys(map,item->"语文".equals(item));
        newMap=Maps.filterValues(map,item->item > 85);

        // Maps还提供了转换新map的方法 结果为 {"英语":"英语100","数学":"数学90","语文":"语文80"}
        newMap=Maps.transformEntries(map,(k,y)->(k+y));
    }

 ```