---
layout: post
title: SpringBoot异步任务@Async的使用
categories: SpringBoot
tags: [多线程, 异步]
---
# 异步任务

异步任务在工作中使用的很多，主要是通过异步的形式提高效率

## 自定义线程池与异步任务(有返回值)

场景：被调用方提供了一个只支持单个订单创建的接口，现在调用方却需要满足能同时创建多个订单的需求.

1. 首先先向容器中注入线程池(自定义的线程池), 这里使用的是阿里巴巴的TTL，需要引入依赖

```xml
    <dependency>
      <groupId>com.alibaba</groupId>
      <artifactId>transmittable-thread-local</artifactId>
      <version>2.12.1</version>
    </dependency>
```

```java
@Configuration
public class AsyncAutoConfiguration {

    @Bean
    public ExecutorService executorService() {
        return TtlExecutors.getTtlExecutorService(
                new ThreadPoolExecutor(
                        120,
                        120,
                        60,
                        TimeUnit.SECONDS,
                        new ArrayBlockingQueue<>(500),
                        new ThreadPoolExecutor.CallerRunsPolicy())
        );
    }
}
```

2. 模拟插入数据库成功返回订单号

```java
@Component
public class OrderMapper {

    public String createOrder(String orderId) {
        return UUID.randomUUID().toString();
    }
}

```

3. Service层 通过循环创建异步任务，并且将异步创建订单的返回值组装起来返回给前端.

```java
@Service
public class OrderService {

    @Autowired
    private ExecutorService executorService;

    @Autowired
    private OrderMapper orderMapper;

    /**
     * 通过异步任务创建订单
     */
    public List<String> createOrder(List<String> orderIds) {

        List<Future<String>> orderTaskFutures = new ArrayList<>();
        List<String> orderIdResponse = new ArrayList<>();

        try {
            orderIds.forEach(orderId -> {
                Future<String> orderTaskFuture = executorService.submit(new OrderCreateTask(orderMapper, orderId));
                orderTaskFutures.add(orderTaskFuture);
            });

            for (Future<String> orderTaskFuture : orderTaskFutures) {
                orderIdResponse.add(orderTaskFuture.get());
            }
        } catch (Exception e) {
            e.printStackTrace();
            Thread.currentThread().interrupt();
        }

        return orderIdResponse;
    }

}
```

4. 编写用于异步的任务, 异步任务中调用第三方生成订单，获取返回值

```java
public class OrderCreateTask implements Callable<String> {

    private OrderMapper orderMapper;
    private String orderId;

    public OrderCreateTask(OrderMapper orderMapper, String orderId) {
        this.orderMapper = orderMapper;
        this.orderId = orderId;
    }

    @Override
    public String call() throws Exception {
        // 这里调用底层代码 生成订单号
        return orderMapper.createOrder(orderId);
    }
}
```

## 通过@Async创建异步任务

1. 自定义线程池，可指定执行器

```
  @Async(Contants.TaskThreadPool.ASYNPOOL_NAME)
```

```
@Configuration
public class AsyncConfiguration implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        return executorService();
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return AsyncConfigurer.super.getAsyncUncaughtExceptionHandler();
    }

    @Bean("executorService")
    public ExecutorService executorService() {
        return TtlExecutors.getTtlExecutorService(
                new ThreadPoolExecutor(
                        120,
                        120,
                        60,
                        TimeUnit.SECONDS,
                        new ArrayBlockingQueue<>(500),
                        new ThreadPoolExecutor.CallerRunsPolicy())
        );
    }
}
```



2. 首先需要在启动类上加上@EnableAsync 注解

3. 在需要异步执行的方法上添加@Async 注解, 这里有返回值, 通过AsyncResult 返回Future.

```
@Component
public class OrderSubmitTask {

    @Async
    public Future<String> submitOrder(String orderId) {
        System.out.println("=======submit" + Thread.currentThread().getId());
        return new AsyncResult<String>(UUID.randomUUID().toString());
    }
}

```

注意：

被调用的异步方法不能与调用者在同一个类中，否则会失效

通过@Async修饰的任务只能放回void 或者Future



```
@Service
public class OrderService {

    @Autowired
    private ExecutorService executorService;

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private OrderSubmitTask orderSubmitTask;

    /**
     * 通过异步任务创建订单
     */
    public List<String> createOrder(List<String> orderIds) {
        System.out.println("=======create order ser"+Thread.currentThread().getId());
        List<String> orderIdResponse = new ArrayList<>();
        for (String orderId : orderIds) {
            Future<String> stringFuture = orderSubmitTask.submitOrder(orderId);

            if (Objects.nonNull(stringFuture)) {
                try {
                    orderIdResponse.add(stringFuture.get());
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
        return orderIdResponse;

    }
}
```

# 扩展

## 注解失效

1. 在同一个类中，一个方法调用另一个有注解的方法(@Async, @Transactional )注解会失效.

原因:  Spring在扫描bean的时候会扫描方法上是否包含@Transactional注解或者 @Async之类的注解，如果包含，Spring 会为这个bean生成一个代理类，代理类是继承原来那个bean的，此时，当有这个注解的方法被调用时，实际是由代理类来调用的, 代理类在调用时增加异步作用，然而，这个有注解的方法是被同一个类中的其他方法调用的，那么这个有注解的方法的调用并没有经过代理类，则没有增加异步效果, 导致注解失效

2. 

## EventBus

EventBus顾名思义，事件总线，是一个轻量级的发布/订阅模式的应用模式。相比于MQ更加简洁，轻量，它可以在一个小系统模块内部使用

## AsyncEventBus

```
@Service
public class AsyncEventBusHelper {
    private AsyncEventBus asyncEventBus;

    @PostConstruct
    private void init() {
        asyncEventBus = new AsyncEventBus(new ThreadPoolExecutor(10, 10,
                0L, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<>()));
    }

    public void register(Object handler) {
        asyncEventBus.register(handler);
    }

    public void post(Object event) {
        asyncEventBus.post(event);
    }

}
```

```
public abstract class AsyncEventListener<T> {

    @Resource
    private AsyncEventBusHelper asyncEventBusHelper;

    @PostConstruct
    public void init() {
        asyncEventBusHelper.register(this);
    }

    public abstract void handleEvent(T event);

}
```

使用步骤:

1. 发送消息时注入AsyncEventBusHelper，使用post方法发送消息
2. 监听消息时继承AsyncEventListener，实现handleEvent方法，在该类中执行业务逻辑
