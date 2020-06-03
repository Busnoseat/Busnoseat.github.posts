---
title: java8的stream相关
date: 2020-06-02 10:18:22
tags: 优化
---

整理下平时关于stream的复杂操作，以便于下次可以直接copy
<!--more-->

## filter
*	根据对象字段去重过滤
```
   class BatchCopyEntity {
      private String sourceKey;
      private String targetKey;
   }


   public static <T> Predicate<T> distinctByKey(Function<? super T, Object> keyExtractor) {
        Map<Object, Boolean> seen = new ConcurrentHashMap<>();
        return t -> seen.putIfAbsent(keyExtractor.apply(t), Boolean.TRUE) == null;
    }

    public static void main(String[] args) {
        List<BatchCopyEntity> affixList = new ArrayList<>();
        List filterList = affixList
                .stream()
                //需要根据BatchCopyEntity的字段sourceKey进行去重
                .filter(distinctByKey((p) -> (p.getSourceKey())))
                .collect(Collectors.toList());
    }  
```

## flatMap
* 将一个对象转换为多个其他对象

```
    public static void main(String[] args) {

        //方式一
        List<String> list = new ArrayList<>();
        list.add("aaa,bbb,ccc");
        list.add("ddd,eee,fff");
        list.add("ggg,hhh,iii");
        //flatMap：对流扁平化处理 ：各个数组并不是分别映射一个流，而是映射成流的内容，所有使用map(Array::stream)时生成的单个流被合并起来，即扁平化为一个流
        List<String> list1 = list.stream().map(s -> s.split(",")).flatMap(Arrays::stream).collect(Collectors.toList());

        //方式二：
        List<String> list2 = new ArrayList<>();
        list.stream().map(s -> s.split(",")).flatMap(Arrays::stream).forEach(item -> {
            list2.add(item);
        });
        System.out.println(list1.size());
        System.out.println(list2.size());

    }
 ```