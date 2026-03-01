---
title: SpringBoot 启动时实现缓存预热
categories:
  - 后端
tags: 
  - Spring
  - SpringBoot
  - Java
  - 缓存
abbrlink: 3b7df5c5
date: 2026-02-28 22:51:31
---
# SpringBoot 启动时实现缓存预热

Spring Boot 启动时预热缓存，核心思路就是在应用启动完成后、对外提供服务之前，把数据库里的热点数据或配置数据主动写入Redis。

主流方案有三种

- **CommandLineRunner / ApplicationRunner**
- **@PostConstruct 注解**
- **监听 ApplicationReadyEvent 事件**

## @PostConstruct 注解

在某个 Bean 初始化完成后执行预热逻辑。

执行时机较早（Bean 初始化阶段），此时可能其他依赖还未完全准备好（例如数据库连接池、Redis 连接等），不适合依赖外部资源的预热。

```java
@Component
public class CachePreloader {

    @Autowired
    private SomeService someService;

    @PostConstruct
    public void preload() {
        // 加载数据到缓存
        List<Data> list = someService.loadHotData();
        list.forEach(data -> someService.cacheData(data));
    }
}
```



## **CommandLineRunner / ApplicationRunner**

 执行时机晚于 `@PostConstruct`，此时所有 Bean 已初始化完成，服务未启动完成，可以安全使用外部资源。

```java
@Component
public class CachePreloader implements CommandLineRunner {

    @Autowired
    private SomeService someService;

    @Override
    public void run(String... args) throws Exception {
        // 预热逻辑
        someService.preloadCache();
    }
}
```



### 二者区别

| 特性                 | CommandLineRunner                        | ApplicationRunner                                  |
| :------------------- | :--------------------------------------- | :------------------------------------------------- |
| **参数类型**         | `String... args`                         | `ApplicationArguments args`                        |
| **参数访问方式**     | 直接接收原始的应用启动参数数组           | 通过封装对象获取解析后的参数                       |
| **是否支持选项解析** | 需要手动解析（如遍历判断 `--key=value`） | 内置支持解析选项参数（如 `--foo=bar`）和非选项参数 |
| **代码简洁性**       | 适合只需简单参数列表的场景               | 适合需要区分选项参数和普通参数的场景               |



## 参数解析示例

假设通过命令行启动 Spring Boot 应用：

```bash
java -jar myapp.jar --server.port=8080 arg1 arg2 --debug
```



### 使用 CommandLineRunner

```java
@Component
public class MyCommandLineRunner implements CommandLineRunner {
    @Override
    public void run(String... args) {
        // args = ["--server.port=8080", "arg1", "arg2", "--debug"]
        for (String arg : args) {
            System.out.println(arg);
        }
    }
}
```



你需要自己解析哪些是选项参数（`--xxx`），哪些是普通参数。

### 使用 ApplicationRunner

```java
@Component
public class MyApplicationRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        // 获取所有选项参数名
        System.out.println("Option names: " + args.getOptionNames());
        // 获取某个选项的值（例如 --server.port）
        System.out.println("server.port = " + args.getOptionValues("server.port"));
        // 获取非选项参数（arg1, arg2）
        System.out.println("Non-option args: " + args.getNonOptionArgs());
        // 获取原始数组（如果仍需）
        System.out.println("Raw args: " + Arrays.toString(args.getSourceArgs()));
    }
}
```



输出：

```
Option names: [debug, server.port]
server.port = [8080]
Non-option args: [arg1, arg2]
Raw args: [--server.port=8080, arg1, arg2, --debug]
```



可以看到，`ApplicationArguments` 已经帮你把参数分好类，直接调用方法即可，无需手动解析。

## 监听 ApplicationReadyEvent 事件

这是最稳妥的方案。ApplicationReadyEvent 在应用完全就绪、可以接收请求时才触发，比 CommandLineRunner 还要晚一点。如果你的缓存预热逻辑特别重要，不能有任何闪失，用这个最保险。

预热期间已有用户请求进入，可能短暂命中空白缓存，但通常可通过降级或重试缓解。

```java
@Component
public class CachePreloader {

    @Autowired
    private SomeService someService;

    @EventListener(ApplicationReadyEvent.class)
    public void preload() {
        // 预热逻辑
        someService.preloadCache();
    }
}
```



## 三种方案的执行时机对比

| 方案                                 | 执行时机                   | 适用场景                         |
| ------------------------------------ | -------------------------- | -------------------------------- |
| @PostConstruct                       | Bean 初始化完成后立即执行  | 单个服务自己负责的局部缓存       |
| CommandLineRunner/ ApplicationRunner | 所有 Bean 初始化完成后     | 全局缓存预热，需要多个 Bean 协作 |
| ApplicationReadyEvent                | 应用完全就绪，可接收请求时 | 最稳妥，适合核心缓存             |

如果你有多个 Runner 或者 Listener，可以通过 @Order 注解来控制执行顺序，数字越小越先执行。



## 大数据量预热的优化策略

如果要预热的数据量很大，比如几十万条商品信息，一次性全捞出来再逐条写 Redis，启动时间会被拖得很长，内存也可能撑不住。

1）**分批加载**：用分页查询，每次查 1000 条，处理完再查下一批。避免一次性把数据库和内存都撑爆。

```java
int page = 0;
int size = 1000;
Page<Product> products;
do {
    products = productRepository.findAll(PageRequest.of(page++, size));
    for (Product p : products) {
        redisTemplate.opsForValue().set("product:" + p.getId(), JSON.toJSONString(p));
    }
} while (products.hasNext());
```

2）**Pipeline 批量写入**：Redis 的 pipeline 可以把多个命令打包一次性发过去，减少网络往返。1000 条数据用 pipeline 写，比单条写快 10 倍不止。

```java
redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
    for (Product p : products) {
        connection.stringCommands().set(
            ("product:" + p.getId()).getBytes(),
            JSON.toJSONString(p).getBytes()
        );
    }
    return null;
});
```

3）**异步预热**：如果缓存预热不是强依赖，可以用 @Async 把预热逻辑丢到线程池里跑，不阻塞主启动流程。但要注意，这样应用启动后的前几秒缓存可能还没预热完，需要业务上能容忍短暂的缓存 miss。

![img](https://pic.code-nav.cn/mianshiya/question_picture/1843904816956411905/yYVuTYZu_pZJ25lgoYN_mianshiya.webp)

## @Cacheable 的懒加载方式

除了主动预热，还有一种懒加载思路：用 Spring Cache 的 @Cacheable 注解，第一次请求时自动把结果写入缓存。

```java
@Cacheable(value = "products", key = "#id")
public Product getProductById(Long id) {
    return productRepository.findById(id).orElse(null);
}
```

但这种方式有个问题：第一个请求还是会打到数据库。如果你的场景是高并发秒杀，第一波流量涌进来时缓存都是空的，效果就很差。所以 @Cacheable 更适合访问频率不高、对首次延迟不敏感的场景。

## 序列化踩坑提醒

往 Redis 写对象时，序列化方式一定要配对。默认的 JdkSerializationRedisSerializer 会往 key 和 value 里塞一堆乱码前缀，用 redis-cli 看起来很痛苦，而且跨语言读不了。

推荐配置 StringRedisSerializer 处理 key，Jackson2JsonRedisSerializer 或 GenericJackson2JsonRedisSerializer 处理 value：

```java
@Bean
public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
    RedisTemplate<String, Object> template = new RedisTemplate<>();
    template.setConnectionFactory(factory);
    template.setKeySerializer(new StringRedisSerializer());
    template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
    return template;
}
```