layout: new
title: restTemplate泛型问题
date: 2020-09-22 17:45:42
tags: 优化
---

RestTemplate不论是post还是get、patch都只支持传入一个class作为返回值的类型，并且不支持泛型。<!--more-->
![image](/asset/article/20200922/1.png)

## 错误案例
定义一个统一的response，里面的字段data泛型，一般Response都会有类似的统一基类
```
public class PaymentDataResponse<T> {

    private String code;
    private String message;
    private T data;

 ...
 }

```

然后强行使用post方法
```
...
PaymentDataResponse<QueryOrderStatusDTO> result=new PaymentDataResponse<>();
result=restTemplate.postForObject(url,formEntity,result.getClass());
...

```
会发现泛型的类被序列化成了hashMap。后续如果使用data的getBean方法会报错，提示LinkedHashMap Cannot cast ...
![image](/asset/article/20200922/2.png)

## 使用fastJson
如果依然想使用post方法，可以在post的时候不转换格式，统一返回String,后续再转换格式。
```

String resultStr=restTemplate.postForObject(url,formEntity,String.class);
 result = JSON.parseObject(resultStr, new TypeReference<PaymentDataResponse<QueryOrderStatusDTO>>() { });

```
![image](/asset/article/20200922/3.png)

当然还可以提供一个公共方法出来，因为TypeReference支持泛型动态传参
```
 /*
  *公共方法 统一转换json为entity
  */
 public static <T> PaymentDataResponse<T> parseToMap(String str, Class<T> classType) {
        return JSON.parseObject(str, new TypeReference<PaymentDataResponse<T>>(classType) {
        });
    }

...
String resultStr=restTemplate.postForObject(url,formEntity,String.class);
result = parseToMap(resultStr,QueryOrderStatusDTO.class);
...
```
不过这样不好，每次请求都需要new一个新的TypeReference。我还是喜欢统一类型的使用同一个typeReference，typeReference定义一个final类型的避免被篡改。


## 使用restTemplate的exchange
如果不想在之后转换格式，而是在请求返回的时候就序列化好，那么可以使用restTemplate的exchange方法

```

 result = restTemplate.exchange(url, HttpMethod.POST, formEntity, new ParameterizedTypeReference<PaymentDataResponse<QueryOrderStatusDTO>>() {
        }).getBody()
```
![image](/asset/article/20200922/4.png)
每种返回类型都需要自定义一个PaymentDataResponse，这样有点繁琐   