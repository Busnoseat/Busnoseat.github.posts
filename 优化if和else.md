---
title: 优化if和else
date: 2019-07-18 17:38:08
tags: 优化
---

我们编写程序时，经常会用到if和else。过多的使用不仅会影响阅读。而且也不方便于后续业务的穿插。大多数优化方案都是采取策略模式，有些直接分派到类的一个方法，有些直接独立出一个类来处理逻辑，以下是本人汇总的一些方案
<!--more-->

# lamdba表达式
利用Function函数来存储所有调用实例，Function<T, R>中T为入参，R为出参。如果方法没有出参也可以改用Consumer函数。

```
public class Vehicle {

    static Map<String, Function<Integer, String>> functionMap = new HashMap<>();

    static String getBus(Integer a) {
        return String.valueOf("getBus:" + a);
    }

    static String getCar(Integer a) {
        return String.valueOf("getCar:" + a);
    }

    static {
        functionMap.put("1", Vehicle::getBus);
        functionMap.put("2", Vehicle::getCar);
    }

    public static void main(String[] args) {
        System.out.println(functionMap.get("1").apply(333));
        System.out.println(functionMap.get("2").apply(333));
    }
}
```
以下例子使用consumer函数的例子，并且委托给各个实现类

```
public class Bus {
    static void getBus(Integer a) {
        System.out.println(String.valueOf("getBus:" + a));
    }
}

public class Car {
    static void getCar(Integer a) {
        System.out.println(String.valueOf("getCar:" + a));
    }
}

public class Vehicle {

    static Map<Class, Consumer<Integer>> consumerMap = new HashMap<>();

    static {
        consumerMap.put(Bus.class, Bus::getBus);
        consumerMap.put(Car.class, Car::getCar);
    }

    public static void main(String[] args) {
        consumerMap.get(Bus.class).accept(333);
        consumerMap.get(Car.class).accept(333);
    }

}

```


# Spring结合策略模式

```

--定义一个枚举，存放所有实现类的code
public enum VehicleEnum {
    BUS("bus","调用bus方法"),
    CAR("car","调用car方法"),
    ;

    private String code;

    private String value;

    VehicleEnum(String code, String value) {
        this.code = code;
        this.value = value;
    }

    public String getCode() {
        return code;
    }

    public void setCode(String code) {
        this.code = code;
    }

    public String getValue() {
        return value;
    }

    public void setValue(String value) {
        this.value = value;
    }
}

public interface VehicleStrategy {
    VehicleEnum supportCategory();

    void execute(Integer a);
}

@Component
public class Bus implements VehicleStrategy {
    static void getBus(Integer a) {
        System.out.println(String.valueOf("getBus:" + a));
    }

    @Override
    public VehicleEnum supportCategory() {
        return VehicleEnum.BUS;
    }

    @Override
    public void execute(Integer a) {
        getBus(a);
    }
}

@Component
public class Car implements VehicleStrategy {

    static void getCar(Integer a) {
        System.out.println(String.valueOf("getCar:" + a));
    }

    @Override
    public VehicleEnum supportCategory() {
        return VehicleEnum.BUS;
    }

    @Override
    public void execute(Integer a) {
        getCar(a);
    }
}

--利用上下文再项目启动的时候，吧所有实现类放到map中，最后再调用的时候根据传进来的key获取响应的value实现类
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(initializers = {ConfigFileApplicationContextInitializer.class})
@Import(value = {Bus.class,Car.class})
public class Vehicle implements ApplicationContextAware {

    private static Map<String, VehicleStrategy> strategyHashMap = new HashMap<>();

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println(111);
        Map<String, VehicleStrategy> a=applicationContext.getBeansOfType(VehicleStrategy.class);
        System.out.println(JsonUtil.toJson(a));
        applicationContext.getBeansOfType(VehicleStrategy.class).forEach((k, v) -> {
            strategyHashMap.put(v.supportCategory().getCode(), v);
        });
    }


    @Test
    public void testStrategy(){
        String code=VehicleEnum.BUS.getCode();
        Integer param=333;
        VehicleStrategy v=strategyHashMap.get(code);
        v.execute(param);
    }
}

```
贴上流程图
![Image text](/asset/article/20190722/1.png)

总结:核心思想为采用策略模式，将实现委托给各个子类，实现方式可以为上图、工厂模式等。