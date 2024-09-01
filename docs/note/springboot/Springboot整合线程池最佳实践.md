## 封装 ThreadPoolTaskExecutor 线程池

### 1、新增 application.yml 配置

这里主要是配置 ThreadPoolTastExecutor 比较重要的参数

```yaml
thread:
  poolexecutor:
    corePoolSize: 10 # 核心线程数量
    maxPoolSize: 30 # 最大线程数量
    queueCapacity: 100 # 队列长度
    keppAliveSeconds: 60 # 存活时间
    prefixName: "taskExecutor-" # 线程名称前缀
```

### 2、properties 类

这个是为了简化代码，不用@Value 一个一个去获取值

```java
package com.walker.async.common.properties;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@Data
@ConfigurationProperties(prefix = "thread.poolexecutor")
public class ThreadPoolProperties {
    private Integer corePoolSize;
    private Integer maxPoolSize;
    private Integer queueCapacity;
    private Integer keppAliveSeconds;
    private String prefixName;
}


```

### 3、配置类

配置线程池，并将其注入到 bean 中

```java
package com.walker.async.common.config;

import com.walker.async.common.properties.ThreadPoolProperties;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;
import java.util.concurrent.ThreadPoolExecutor;

//开启异步
@EnableAsync
//配置类
@Configuration
public class ThreadPoolConfig {
    @Autowired
    private ThreadPoolProperties threadPoolProperties;

    //bean名称
    @Bean("taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(threadPoolProperties.getCorePoolSize());
        executor.setMaxPoolSize(threadPoolProperties.getMaxPoolSize());
        executor.setQueueCapacity(threadPoolProperties.getQueueCapacity());
        executor.setKeepAliveSeconds(threadPoolProperties.getKeppAliveSeconds());
        executor.setThreadNamePrefix(threadPoolProperties.getPrefixName());
        //设置线程池关闭的时候 等待所有的任务完成后再继续销毁其他的bean
        executor.setWaitForTasksToCompleteOnShutdown(true);
        //策略
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return executor;
    }
}

```

<!-- more -->

### 4、封装 Service 和 service 实现类,方便管理

```java
package com.walker.async.service.async;

public interface TestAsync {

    void doAsync();
}

```

```java
package com.walker.async.service.async.impl;

import com.walker.async.service.async.TestAsync;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

@Service
@Slf4j
public class TestAsyncImpl implements TestAsync {


    @Override
    //使用@Async，并将前面的注册的bean，填写到Async的value中
    @Async("taskExecutor")
    public void doAsync() {
        log.info("== async start==");
        log.info("线程{}执行代码逻辑",Thread.currentThread().getName());
        log.info("== async end==");
    }
}

```

### 5、测试

```java
package com.walker.async;

import com.walker.async.service.async.TestAsync;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
public class AsyncTest {
    //引用类
    @Autowired
    private TestAsync testAsync;

    @Test
    void test(){
        for (int i = 0; i < 10; i++) {
            testAsync.doAsync();
        }

    }
}

```

返回结果：

```java
2023-01-29 16:47:12.491  INFO 13332 --- [ taskExecutor-2] c.w.a.service.async.impl.TestAsyncImpl   : == async start==
2023-01-29 16:47:12.491  INFO 13332 --- [ taskExecutor-1] c.w.a.service.async.impl.TestAsyncImpl   : == async start==
2023-01-29 16:47:12.493  INFO 13332 --- [ taskExecutor-4] c.w.a.service.async.impl.TestAsyncImpl   : == async start==
2023-01-29 16:47:12.493  INFO 13332 --- [ taskExecutor-3] c.w.a.service.async.impl.TestAsyncImpl   : == async start==
2023-01-29 16:47:12.493  INFO 13332 --- [ taskExecutor-5] c.w.a.service.async.impl.TestAsyncImpl   : == async start==
2023-01-29 16:47:12.501  INFO 13332 --- [ taskExecutor-9] c.w.a.service.async.impl.TestAsyncImpl   : == async start==
2023-01-29 16:47:12.501  INFO 13332 --- [taskExecutor-10] c.w.a.service.async.impl.TestAsyncImpl   : == async start==
2023-01-29 16:47:12.500  INFO 13332 --- [ taskExecutor-7] c.w.a.service.async.impl.TestAsyncImpl   : == async start==
2023-01-29 16:47:12.502  INFO 13332 --- [ taskExecutor-8] c.w.a.service.async.impl.TestAsyncImpl   : == async start==
2023-01-29 16:47:12.500  INFO 13332 --- [ taskExecutor-6] c.w.a.service.async.impl.TestAsyncImpl   : == async start==
2023-01-29 16:47:12.502  INFO 13332 --- [ taskExecutor-6] c.w.a.service.async.impl.TestAsyncImpl   : 线程taskExecutor-6执行代码逻辑
2023-01-29 16:47:12.501  INFO 13332 --- [ taskExecutor-5] c.w.a.service.async.impl.TestAsyncImpl   : 线程taskExecutor-5执行代码逻辑
2023-01-29 16:47:12.503  INFO 13332 --- [ taskExecutor-5] c.w.a.service.async.impl.TestAsyncImpl   : == async end==
2023-01-29 16:47:12.492  INFO 13332 --- [ taskExecutor-2] c.w.a.service.async.impl.TestAsyncImpl   : 线程taskExecutor-2执行代码逻辑
2023-01-29 16:47:12.503  INFO 13332 --- [ taskExecutor-2] c.w.a.service.async.impl.TestAsyncImpl   : == async end==
2023-01-29 16:47:12.503  INFO 13332 --- [ taskExecutor-6] c.w.a.service.async.impl.TestAsyncImpl   : == async end==
2023-01-29 16:47:12.501  INFO 13332 --- [ taskExecutor-9] c.w.a.service.async.impl.TestAsyncImpl   : 线程taskExecutor-9执行代码逻辑
2023-01-29 16:47:12.492  INFO 13332 --- [ taskExecutor-1] c.w.a.service.async.impl.TestAsyncImpl   : 线程taskExecutor-1执行代码逻辑
2023-01-29 16:47:12.506  INFO 13332 --- [ taskExecutor-1] c.w.a.service.async.impl.TestAsyncImpl   : == async end==
2023-01-29 16:47:12.506  INFO 13332 --- [ taskExecutor-9] c.w.a.service.async.impl.TestAsyncImpl   : == async end==
2023-01-29 16:47:12.493  INFO 13332 --- [ taskExecutor-3] c.w.a.service.async.impl.TestAsyncImpl   : 线程taskExecutor-3执行代码逻辑
2023-01-29 16:47:12.501  INFO 13332 --- [taskExecutor-10] c.w.a.service.async.impl.TestAsyncImpl   : 线程taskExecutor-10执行代码逻辑
2023-01-29 16:47:12.506  INFO 13332 --- [ taskExecutor-3] c.w.a.service.async.impl.TestAsyncImpl   : == async end==
2023-01-29 16:47:12.506  INFO 13332 --- [taskExecutor-10] c.w.a.service.async.impl.TestAsyncImpl   : == async end==
2023-01-29 16:47:12.502  INFO 13332 --- [ taskExecutor-8] c.w.a.service.async.impl.TestAsyncImpl   : 线程taskExecutor-8执行代码逻辑
2023-01-29 16:47:12.506  INFO 13332 --- [ taskExecutor-8] c.w.a.service.async.impl.TestAsyncImpl   : == async end==
2023-01-29 16:47:12.502  INFO 13332 --- [ taskExecutor-7] c.w.a.service.async.impl.TestAsyncImpl   : 线程taskExecutor-7执行代码逻辑
2023-01-29 16:47:12.508  INFO 13332 --- [ taskExecutor-7] c.w.a.service.async.impl.TestAsyncImpl   : == async end==
2023-01-29 16:47:12.493  INFO 13332 --- [ taskExecutor-4] c.w.a.service.async.impl.TestAsyncImpl   : 线程taskExecutor-4执行代码逻辑
2023-01-29 16:47:12.508  INFO 13332 --- [ taskExecutor-4] c.w.a.service.async.impl.TestAsyncImpl   : == async end==

```

可以发现，该方法使用了线程池.

### 6、打印线程池情况（自主选择）

- 编写 ThreadPoolTaskExecutor 继承类

```java
package com.walker.async.common.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.util.concurrent.ListenableFuture;

import java.util.concurrent.Callable;
import java.util.concurrent.Future;
import java.util.concurrent.ThreadPoolExecutor;

@Slf4j
//1、继承ThreadPoolTaskExecutor
public class VisibleThreadPoolTaskExecutor extends ThreadPoolTaskExecutor  {


    //2、编写打印线程池方法
    private void log(String method){
        ThreadPoolExecutor threadPoolExecutor = getThreadPoolExecutor();

        if(threadPoolExecutor==null){
            return;
        }

        log.info("线程池：{}, 执行方法：{},任务数量 [{}], 完成任务数量 [{}], 活跃线程数 [{}], 队列长度 [{}]",
                this.getThreadNamePrefix(),
                method,
                threadPoolExecutor.getTaskCount(),
                threadPoolExecutor.getCompletedTaskCount(),
                threadPoolExecutor.getActiveCount(),
                threadPoolExecutor.getQueue().size());
    }



    //3、重写方法，进行日志的记录
    @Override
    public void execute(Runnable task) {
        log("execute");
        super.execute(task);
    }

    @Override
    public void execute(Runnable task, long startTimeout) {
        log("execute");
        super.execute(task, startTimeout);
    }

    @Override
    public Future<?> submit(Runnable task) {
        log("submit");
        return super.submit(task);
    }

    @Override
    public <T> Future<T> submit(Callable<T> task) {
        log("submit");
        return super.submit(task);
    }

    @Override
    public ListenableFuture<?> submitListenable(Runnable task) {
        log("submitListenable");
        return super.submitListenable(task);
    }

    @Override
    public <T> ListenableFuture<T> submitListenable(Callable<T> task) {
        log("submitListenable");
        return super.submitListenable(task);
    }
}

```

- 重新编写线程池配置类

```java
package com.walker.async.common.config;

import com.walker.async.common.properties.ThreadPoolProperties;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;
import java.util.concurrent.ThreadPoolExecutor;

@EnableAsync
@Configuration
public class ThreadPoolConfig {
    @Autowired
    private ThreadPoolProperties threadPoolProperties;

    @Bean("taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(threadPoolProperties.getCorePoolSize());
        executor.setMaxPoolSize(threadPoolProperties.getMaxPoolSize());
        executor.setQueueCapacity(threadPoolProperties.getQueueCapacity());
        executor.setKeepAliveSeconds(threadPoolProperties.getKeppAliveSeconds());
        executor.setThreadNamePrefix(threadPoolProperties.getPrefixName());
        //设置线程池关闭的时候 等待所有的任务完成后再继续销毁其他的bean
        executor.setWaitForTasksToCompleteOnShutdown(true);
        //策略
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return executor;
    }



    @Bean("visibleTaskExecutor")
    public Executor visible() {

        //1、使用VisibleThreadPoolTaskExecutor作为类
        ThreadPoolTaskExecutor executor = new VisibleThreadPoolTaskExecutor();
        executor.setCorePoolSize(threadPoolProperties.getCorePoolSize());
        executor.setMaxPoolSize(threadPoolProperties.getMaxPoolSize());
        executor.setQueueCapacity(threadPoolProperties.getQueueCapacity());
        executor.setKeepAliveSeconds(threadPoolProperties.getKeppAliveSeconds());
        executor.setThreadNamePrefix(threadPoolProperties.getPrefixName());
        //设置线程池关闭的时候 等待所有的任务完成后再继续销毁其他的bean
        executor.setWaitForTasksToCompleteOnShutdown(true);
        //策略
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return executor;
    }
}

```

- service 和实现类也重新编写

```java
void visibleAsync();

```

```java
    /**
    * 可视化，可以打印线程池情况
    */
    @Override
    @Async("visibleTaskExecutor")
    public void visibleAsync() {
        log.info("== async start==");
        log.info("线程{}执行代码逻辑",Thread.currentThread().getName());
        log.info("== async end==");
    }
```

- 测试

```java
package com.walker.async;

import com.walker.async.service.async.TestAsync;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
public class AsyncTest {
    @Autowired
    private TestAsync testAsync;


    @Test
    void test(){
        for (int i = 0; i < 10; i++) {
//            testAsync.doAsync();
            testAsync.visibleAsync();
        }
    }
}

```

- 返回结果

![](https://img-blog.csdnimg.cn/img_convert/62e93416f35ff168f8525f8933b99615.png#averageHue=#383736&clientId=u3562c02b-c2ea-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=266&id=u6317e0f4&originHeight=333&originWidth=982&originalType=binary&ratio=1&rotation=0&showTitle=false&size=64672&status=done&style=none&taskId=u9d09be0e-7fbd-4503-9ccc-5f54c336f2c&title=&width=785.6#errorMessage=unknown%20error&id=Y5z3B&originHeight=333&originWidth=982&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
每次执行的时候，都会打印对应的线程池情况了
