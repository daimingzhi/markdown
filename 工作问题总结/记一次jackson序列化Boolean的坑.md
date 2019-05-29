记一次jackson序列化Boolean的坑

```java
@Data
public class CouponTemplateDto {
    /**
     * 优惠券类型id
     */
    private Long couponTypeId;
    /**
     * 优惠券模板id
     */
    private Long couponTemplateId;
    /**
     * 用户id
     */
    private Long userId;

    /**
     * 优惠券描述
     */
    private String description;
    /**
     * 面值，满200减30，则此值为30
     */
    private BigDecimal value;
    /**
     * 从次日起，多少天可用
     */
    private Integer delayDays;
    /**
     * 从当日起，多少天可用
     */
    private Integer nowDays;
    /**
     * 满多少可以减，满200减30，则此值为200
     */
    private BigDecimal fullAmount;

    /**
     * 券号
     */
    private String couponNo;

    /**
     * 有效起始日期
     */
    private Date startTime;

    /**
     * 失效日期
     */
    private Date endTime;

    /**
     * 创建时间
     */
    private Date createTime;

    /**
     * 使用日期
     */
    private Date useTime;

    /**
     * 券使用状态:0-未使用 1-已使用 2-已过期
     */
    private Integer couponUseStatus;
    /**
     * 过期前多少天提醒，默认7天
     */
    private Integer overDueRemind;
    /**
     * 优惠券标题
     */
    private String title;

    /**
     * 优惠券是否能开始使用
     */
//    @JsonProperty("isStart")
    private Boolean start;
    /**
     * 优惠券是否过期
     */
//    @JsonProperty("isEnd")
    private Boolean end;

    private Boolean getStart() {
        return startTime.before(new Date());
    }

    private Boolean getEnd() {
        return endTime.before(new Date());
    }
}
```

我定义了一个这样的类，我们项目用的是Spring Boot，默认底层采用的是jackson序列化，但是在使用中出了一个问题`private Boolean start`跟`private Boolean end`这两个字段一直无法序列化

总结排查思路如下：

1. 是boolean还是Boolean，到底是基本数据类型还是包装类，如果是基本数据类型的话（包装类可以使用，但是不推荐），不要使用is开头。我们可以看看阿里巴巴规范中的这段话

   > 8. 【强制】POJO类中的任何布尔类型的变量，都不要加 is，否则部分框架解析会引起序列化错误。
   >
   > 反例：定义为基本数据类型 boolean isSuccess；的属性，它的方法也是 isSuccess()，RPC框架在反向解析的时候，“以为”对应的属性名称是 success，导致属性获取不到，进而抛出异常。

2. 这个错误也是我犯的错误，我复写了get方法，但是方法的访问权限被设置成了private级别

   解决方案：

   - 加注解，@JsonProperty("isEnd")
   - 将方法级别更正为public