---
abbrlink: ''
categories:
- - 学习记录
date: '2025-10-02T17:18:37.453043+08:00'
tags:
- Kafka
- Java
title: 解决Kafka 序列化 Java 8 LocalDate 失败的踩坑记录
updated: '2025-10-02T17:18:40.889+08:00'
readmore: true
author: FZ688
excerpt: '记录kafka生产者发送对象消息测试时遇到的一个问题，对象中的属性有Java8的`java.time`类型，导致Json序列化无法成功！ 下面是示例: ```java @Data @AllArgsConstructor @NoArgsConstructor @Builder public class User implements Serializable { private int id; private String name; private String phone; private LocalDate birthday; } ``` ```java @Component public class EventProducer { @Resource private KafkaTemplate<String, Object> kafkaTemplate; public void sendEvent() { User user = User.builder() .id(2) .name("里斯") ...'
---
记录kafka生产者发送对象消息测试时遇到的一个问题，对象中的属性有Java8的`java.time`类型，导致Json序列化无法成功！
下面是示例:
## 示例代码
```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class User implements Serializable {

    private int id;

    private String name;

    private String phone;

    private LocalDate birthday;
}
```

```java
@Component
public class EventProducer {

    @Resource
    private KafkaTemplate<String, Object> kafkaTemplate;
  
    public void sendEvent() {
      
        User user = User.builder()
                .id(2)
                .name("里斯")
                .phone("13800138021")
                .birthday(LocalDate.of(1991, 1, 1))
                .build();
      
        kafkaTemplate2.sendDefault(null, System.currentTimeMillis(), "k1", user);
    }
}
```

```java
@Test
void test() {
    eventProducer.sendEvent();
}
```

## 问题分析
测试完发现报错信息如下：

可以看到最重要的一句话

`Caused by: com.fasterxml.jackson.databind.exc.InvalidDefinitionException: Java 8 date/time type java.time.LocalDate not supported by default: add Module "com.fasterxml.jackson.datatype:jackson-datatype-jsr310" to enable handling (or disable MapperFeature.REQUIRE_HANDLERS_FOR_JAVA8_TIMES) (through reference chain: com.fz.module.User["birthday"])`

这句话的意思式:Jackson 在序列化 User 对象时，遇到字段 birthday（类型 java.time.LocalDate），但当前使用的 ObjectMapper 没有注册处理 Java 8 日期时间 API 的模块，因此不支持序列化这个类型，必须手动加入 jackson-datatype-jsr310 模块（或关闭 MapperFeature.REQUIRE_HANDLERS_FOR_JAVA8_TIMES 检查，但不推荐）。

![image-20251002170422417](https://8b63a20.webp.li/2025-10-02-1759395862553.png)

![image-20251002170039813](https://8b63a20.webp.li/2025-10-02-1759395640158.png)

## 解决方案
问题的核心在于`java.time.LocalDateTime`类型在`Jackson`默认序列化器中不被支持。为了解决这个问题，需要引入`Jackson`的`jackson-datatype-jsr310` 模块，并将其注册到 ObjectMapper 中。

在对应的pom.xml中添加以下依赖：

```xml
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
    <version>2.19.2</version><!-- 请根据实际使用的 Jackson 版本调整 -->
</dependency>
```

完成配置后，重新运行测试代码，Kafka 消息成功发送！通过消费者查看消息内容，可以看到 LocalDate 字段被正确序列化为字符串（如`"birthday":"1991-01-01"`），而非默认的时间戳或错误格式。
