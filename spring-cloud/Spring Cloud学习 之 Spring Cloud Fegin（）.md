如何优雅的使用Fegin去构造通用的服务调用的API

###### 第一步：

创建一个公共的API服务：命名为api（根据自己实际情况进行命名）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.5.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.stuyd.cloud</groupId>
    <artifactId>api</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>api</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Greenwich.SR1</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

###### 第二步：创建调用的API

```java
/**
 * name属性代表要调用的服务，不区分大小写
 * fallback代表服务熔断调用的方法，类似还是有fabllbackFactory,后续介绍
 * decode404为true代表404状态码不进入fallback逻辑
 * path配置全局路径，类型类上使用@RequestMapping
 */
@FeignClient(name = "eureka-client", fallback = FallBack.class, decode404 = true,path = "/client")
public interface FeignApi {
    @PostMapping("/hello/{who}")
    String hello(@PathVariable(value = "who") String who) throws Exception;
}
```

###### 第三步：在调用的服务中，创建实际的实现逻辑

```java
@RequiredArgsConstructor
@Slf4j
@RestController
//这个地方记得要跟上面的path属性中的值保持一致
@RequestMapping("/client")
public class ClientController implements FeignApi {

    private final DiscoveryClient discoveryClient;

    @Override
    public String hello(String who) throws Exception {
        Annotation[] annotations = this.getClass().getAnnotations();
        System.out.println(1111);
        Thread.sleep(3000);
        List<String> services = discoveryClient.getServices();
        services.forEach(s -> {
            List<ServiceInstance> instances = discoveryClient.getInstances(s);
            instances.forEach(serviceInstance -> {
                System.out.println(serviceInstance.getHost());
                System.out.println(serviceInstance.getInstanceId());
            });
        });
        return "hello, " + who;
    }
}
```

###### 第四步：实现fallback逻辑

1. 这种方案拿不到致使服务降级的错误码，可能导致无法做一些针对性的处理

```java
@Component
public class FallBack implements FeignApi {

    @Override
    public String hello(String who) throws Exception {
        return "error";
    }
}
```

2. @FeignClient(name = "eureka-client", fallbackFactory = FallBack.class, decode404 = true,path = "/client")，如上修改一下，采用fallbackFactory 的方式

```java
@Component
public class FallBack implements FallbackFactory<FeignApi> {

    @Override
    public FeignApi create(Throwable throwable) {
        return who -> throwable.getMessage();
    }
}
```