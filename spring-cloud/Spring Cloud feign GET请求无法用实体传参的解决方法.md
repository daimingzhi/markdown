Spring Cloud feign GET请求无法用实体传参的解决方法

代码如下：

```java
@FeignClient(name = "eureka-client", fallbackFactory = FallBack.class, decode404 = true, path = "/client")
public interface FeignApi {
//    @PostMapping("/hello/{who}")
//    String hello(@PathVariable(value = "who") String who) throws Exception;

    @GetMapping("/hello")
    String hello(Params params) throws Exception;
}
```

调用报错：

feign.FeignException: status 405 reading FeignApi#hello(Params)

解决办法：

1. 改用post请求，添加@RequestBodey注解

2. 新增@SpringQueryMaq注解，如下：

3. ```java
   @GetMapping("/hello")
   String hello(@SpringQueryMap Params params) throws Exception;
   ```

   