在分布式的情况下，我们没法使用本地的锁进行加锁从而实现并发同步，这个使用我们就需要使用到 redis 实现的分布式锁了
这里的话介绍的是 redisson，也是官方推荐使用的

#

# 步骤

## 1、导入依赖

```xml
<!--        redisson-->
        <dependency>
            <groupId>org.redisson</groupId>

            <artifactId>redisson-spring-boot-starter</artifactId>

            <version>3.15.5</version>

        </dependency>

```

## 2、编写 application.yml

```yaml
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    password:
    # redisson配置文件路径
    redisson:
      file: classpath:redisson.yml
```

## 3、在 resource 下面创建 redisson.yml

```yaml
# 单节点配置
singleServerConfig:
  # 连接空闲超时，单位：毫秒
  idleConnectionTimeout: 10000
  # 连接超时，单位：毫秒
  connectTimeout: 10000
  # 命令等待超时，单位：毫秒
  timeout: 3000
  # 命令失败重试次数,如果尝试达到 retryAttempts（命令失败重试次数） 仍然不能将命令发送至某个指定的节点时，将抛出错误。
  # 如果尝试在此限制之内发送成功，则开始启用 timeout（命令等待超时） 计时。
  retryAttempts: 3
  # 命令重试发送时间间隔，单位：毫秒
  retryInterval: 1500
  # 密码
  password:
  # 单个连接最大订阅数量
  subscriptionsPerConnection: 5
  # 客户端名称
  clientName: myredis
  # 节点地址
  address: redis://127.0.0.1:6379
  # 发布和订阅连接的最小空闲连接数
  subscriptionConnectionMinimumIdleSize: 1
  # 发布和订阅连接池大小
  subscriptionConnectionPoolSize: 50
  # 最小空闲连接数
  connectionMinimumIdleSize: 32
  # 连接池大小
  connectionPoolSize: 64
  # 数据库编号
  database: 0
  # DNS监测时间间隔，单位：毫秒
  dnsMonitoringInterval: 5000
# 线程池数量,默认值: 当前处理核数量 * 2
#threads: 0
# Netty线程池数量,默认值: 当前处理核数量 * 2
#nettyThreads: 0
# 编码
codec: !<org.redisson.codec.JsonJacksonCodec> {}
# 传输模式
transportMode: "NIO"
```

<!-- more -->

## 4、测试

### 方法一：test

#### 1、编写测试类

```java
package com.walker;

import org.apache.catalina.core.ApplicationContext;
import org.junit.jupiter.api.Test;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
public class RedissonTest {

    //1、引入redisson客户端
    @Autowired
    private RedissonClient redissonClient;

    private final static String LOCK="I_AM_LOCK";

    @Test
    public void test1() throws InterruptedException {

        //获取锁
        RLock lock = redissonClient.getLock(LOCK);


        //执行多个线程去获取锁和释放锁
        for (int i = 0; i < 5; i++) {
            new Thread(()->{
                lock.lock();
                try {
                    Thread.sleep(1000);
                    System.out.println(Thread.currentThread().getName()+"：\t 获得锁");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    System.out.println(Thread.currentThread().getName()+"：\t 释放锁锁");
                    lock.unlock();
                }
            }).start();
        }
        Thread.sleep(100000);

    }

}

```

#### 2、测试

执行测试方法，之后发现每次只有一个线程可以获取锁和释放锁
![image.png](https://img-blog.csdnimg.cn/img_convert/ef30f1aad0347dd505dca416632bd6e5.png#clientId=ua2890fa5-5efd-4&from=paste&height=210&id=u18066995&margin=%5Bobject%20Object%5D&name=image.png&originHeight=210&originWidth=517&originalType=binary&ratio=1&size=15346&status=done&style=none&taskId=u02c74451-74df-403d-be18-d67aace0c1b&width=517#errorMessage=unknown%20error&id=VZPHE&originHeight=210&originWidth=517&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

### 方法二：controller

#### 1、编写 controller

编写两个方法，先执行 get1 方法 执行执行 get2，尽量保证两个方法同时执行，会发现 get1 会先执行，然后由于 get2 获取不到锁，所以会等待一段时间之后才会执行

```java
package com.walker.controller;

import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/redisson")
public class RedissonController {

    //1、
    @Autowired
    private RedissonClient redissonClient;
    private final static String LOCK="i_am_lock";

    /**
     *
     */
    @GetMapping("/get1")
    public String get1() throws InterruptedException {
        //获取锁
        RLock lock = redissonClient.getLock(LOCK);
        //加锁
        lock.lock();
        System.out.println("get1 获取数据");
        //让线程先休眠4s
        Thread.sleep(4000);
        //解锁
        lock.unlock();
        return "ok";
    }


    @GetMapping("/get2")
    public String get2(){
        long start = System.currentTimeMillis();
        RLock lock = redissonClient.getLock(LOCK);
        lock.lock();
        long time = System.currentTimeMillis()-start;
        //这里看看多久之后才能够获取锁，能够执行下面的语句
        System.out.println("过了"+String.valueOf(time/1000)+"s获取锁");
        System.out.println("get2 获取数据");
        lock.unlock();
        return "ok";
    }
}

```

#### 2、测试

这里 get1 获取锁之后输出数据，但是没有立即将锁释放
然后 get2 过了一段时间之后才能够拿到锁，这里的时间可能都不同，因为点击 get2 接口的时候有快有慢，就会导致时间不一致了，
但是都可能证明，必须要持有锁才能继续执行，否则就会进行等待
![image.png](https://img-blog.csdnimg.cn/img_convert/087aafa1e9453c1096d8fd85c395f836.png#clientId=ua2890fa5-5efd-4&from=paste&height=76&id=u1fd5fe6a&margin=%5Bobject%20Object%5D&name=image.png&originHeight=76&originWidth=407&originalType=binary&ratio=1&size=3847&status=done&style=none&taskId=u743fc6e4-9ddc-4d85-aa8a-f6712b6ebc4&width=407#errorMessage=unknown%20error&id=iiRLA&originHeight=76&originWidth=407&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

参考：[https://www.cnblogs.com/h--d/p/14858848.html](https://www.cnblogs.com/h--d/p/14858848.html)
[
](https://blog.csdn.net/qq_37892957/article/details/89322334)
