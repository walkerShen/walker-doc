## 缓存异常实践案例
redis基本上是高并发场景上会用到的一个高性能的key-value数据库，属于nosql类型，一般用作于缓存，一般是结合数据库一块使用的，但是在使用的过程中可能会出现异常的问题，就是面试常常唠嗑的缓存异常问题
分别是缓存击穿，缓存穿透和雪崩，简单解释如下：
**缓存穿透：**
就是当用户查询数据时，缓存和数据库该数据都是不存在的，此时如果用户不断的请求，就会不断的查询缓存和数据库，对数据库造成很大压力
**缓存击穿：**
当热点key过期或者丢失时，大量的请求访问该数据，缓存不存在，就会将大量的请求直接访问数据库，造成数据库有大量连接，存在崩溃风险
**缓存雪崩：**
是指大量请求在缓存中没有查到数据，直接访问数据库，导致数据库压力增大，最终导致数据库崩溃，从而波及整个系统不可用，好像雪崩一样。
下面就讲讲案例，并提供解决方案
### 常规写法
平常的写法，未考虑异常时
现在是有一个查询商户的接口
这个是正常的，结合了redis的逻辑，
![](https://img-blog.csdnimg.cn/img_convert/2d7d0fa73b127da9c82fbe80756c4aeb.png#averageHue=#f9f8f7&clientId=u05ed020d-8b60-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=567&id=u54a8040b&originHeight=709&originWidth=762&originalType=binary&ratio=1&rotation=0&showTitle=false&size=71205&status=done&style=none&taskId=u0e1f7abe-f40d-4a82-9b8f-ebc72922d8d&title=&width=609.6#errorMessage=unknown%20error&id=j5zaE&originHeight=709&originWidth=762&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
```java
public TbShopEntity rawQuery(Long id) {
        TbShopEntity shop = null;
        //1.是否命中缓存
        String shopJson = redisClient.get(CACHE_SHOP_KEY + id);
        //1.1 如果命中，且数据非空，则直接返回
        if (!StrUtil.isEmpty(shopJson)) {
            return JSONObject.parseObject(shopJson, TbShopEntity.class);
        }
        //1.2 查询数据库
        shop = getById(id);
        //1.2.1 如果数据库不存在，则直接返回
        if (shop == null) return null;
        //1.2.2 如果存在则设置到redis中
        redisClient.set(CACHE_SHOP_KEY + id, JSON.toJSONString(shop));
        //2.返回数据
        return shop;
    }
```
现在使用一个不存在的商品id，然后使用jmeter进行压测，例如使用id=0的商品，然后使用200个线程共发起200个请求进行压测
![](https://img-blog.csdnimg.cn/img_convert/3205f83e5be31d1ffc7b6b1138567762.png#averageHue=#efefee&clientId=u05ed020d-8b60-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=50&id=u47e5244a&originHeight=63&originWidth=965&originalType=binary&ratio=1&rotation=0&showTitle=false&size=6680&status=done&style=none&taskId=u8d5a6376-75a6-4f1c-9f6b-52a3a64bab2&title=&width=772#errorMessage=unknown%20error&id=tC0hH&originHeight=63&originWidth=965&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
![](https://img-blog.csdnimg.cn/img_convert/614f5e22599828b6d30d5f171f442ab6.png#averageHue=#f1f1f1&clientId=u05ed020d-8b60-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=111&id=u6cf7e692&originHeight=139&originWidth=739&originalType=binary&ratio=1&rotation=0&showTitle=false&size=3723&status=done&style=none&taskId=u85d8e48e-c7a6-4fb9-8bba-e304dbb8c2b&title=&width=591.2#errorMessage=unknown%20error&id=IxFb6&originHeight=139&originWidth=739&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
结果：
发现有大量的请求直接请求数据库，如果时大量的请求，有可能会把数据库搞崩,也就是我们的缓存穿透问题
![](https://img-blog.csdnimg.cn/img_convert/e86e49074188ec4814a2df3a87e65158.png#averageHue=#302f2e&clientId=u05ed020d-8b60-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=202&id=u9de9fc89&originHeight=253&originWidth=1794&originalType=binary&ratio=1&rotation=0&showTitle=false&size=120231&status=done&style=none&taskId=uf0cd2424-5bbf-4b93-b276-33b2c2391bc&title=&width=1435.2#errorMessage=unknown%20error&id=Nxq4J&originHeight=253&originWidth=1794&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

### 缓存穿透问题
分析：
如果是这种情况，当用户并发测试访问一个不存在的key时，会有大量的不存在的key访问数据，导致数据库压力剧增，也就是缓存穿透问题

#### 改进方式
缓存穿透一般有两种解决方式，分别是： 

- 设置带有过期时间的空值
- 布隆过滤器

**设置带有过期时间的空值**
逻辑图如下：主要是改进这里
![](https://img-blog.csdnimg.cn/img_convert/ffb9ca456b9f044dd0ed72f46db9a70b.png#averageHue=#f9f8f7&clientId=u3a364862-abff-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=591&id=u8af12c4e&originHeight=739&originWidth=866&originalType=binary&ratio=1&rotation=0&showTitle=false&size=75459&status=done&style=none&taskId=ue4c7faea-0591-4a0c-9022-82302a1e511&title=&width=692.8#errorMessage=unknown%20error&id=qsuBq&originHeight=739&originWidth=866&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
```java
/**
     * 缓存穿透解决方案, 1、设置一个空值，且带有较短的过期时间 2、布隆过滤器
     * <p>
     * 但是目前还是没有解决穿透的问题的，因为该key不存在，所以会有多个请求去直接请求数据库(并发问题)，从而需要进行优化，解决缓存穿透问题
     */
    private TbShopEntity queryWithPassThrough(Long id) {
        //1.是否命中缓存
        String shopJson = redisClient.get(CACHE_SHOP_KEY + id);

        //1.1 如果命中缓存，则直接返回
        //这里不能使用！Strutil.isEmpty去判断，因为在下边会使用”“去存空值
        if (shopJson != null) {
            //如果返回的值是""，则直接返回null
            if (StrUtil.isEmpty(shopJson)) return null;
            //否则则返回结果
            return JSONObject.parseObject(shopJson, TbShopEntity.class);
        }

        //1.2 如果没有命中,查询数据库
        TbShopEntity shop = getById(id);

        //1.2.1 如果数据库查询数据不存在，则直设置值为空，以及过期时间,现在是设置为10s，如果10s已经过，再重新查询
        if (shop == null) {
            redisTemplate.opsForValue().set(CACHE_SHOP_KEY + id, StrUtil.EMPTY, SECKILL_SECONDS, TimeUnit.SECONDS);
            return null;
        }

        //1.2.2 如果数据库存在，则设置到redis中
        redisClient.set(CACHE_SHOP_KEY + id, JSON.toJSONString(shop));
        //并转成shop返回

        //2.返回数据
        return JSONObject.parseObject(shopJson, TbShopEntity.class);
    }
```
**测试一：直接调用接口**
之后进行测试，首先是使用链接直接访问一个不存在的商品
![](https://img-blog.csdnimg.cn/img_convert/27e22f9b7caaee8bb0abdb63bd83f90d.png#averageHue=#e8f3f3&clientId=u05ed020d-8b60-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=38&id=u214029d9&originHeight=47&originWidth=468&originalType=binary&ratio=1&rotation=0&showTitle=false&size=4217&status=done&style=none&taskId=u0075e2ca-4569-48db-b588-c110a036ded&title=&width=374.4#errorMessage=unknown%20error&id=ogoxF&originHeight=47&originWidth=468&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
可以发现，在第一次访问的时候，会去查找数据库，然后在之后的10s内，都不会进行数据库的查询了，然后当数据过期之后，才会进行查询，从这个结果来看，是没有问题的，也就是平常我们可以这样子使用的！

**测试二：使用并发测试进行**
但是如果使用200个并发去测试，结果又是如何呢？
![](https://img-blog.csdnimg.cn/img_convert/e93e86232d539c3b5be53c6192f3cbfc.png#averageHue=#ededec&clientId=u94a77f1f-6df0-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=123&id=u57a5c30d&originHeight=154&originWidth=1121&originalType=binary&ratio=1&rotation=0&showTitle=false&size=13522&status=done&style=none&taskId=u2c735d73-17bd-4c5b-8d1e-cfebef954f2&title=&width=896.8#errorMessage=unknown%20error&id=TRebI&originHeight=154&originWidth=1121&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
可以发现，还是有大部分请求直接去查询数据库，
![](https://img-blog.csdnimg.cn/img_convert/0f8ec4d6491244dc818a55e3b26f33fa.png#averageHue=#2f2e2d&clientId=u94a77f1f-6df0-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=192&id=ub018a97a&originHeight=240&originWidth=1301&originalType=binary&ratio=1&rotation=0&showTitle=false&size=81854&status=done&style=none&taskId=udff8f0b9-bde9-4465-9b53-6e1778f6da0&title=&width=1040.8#errorMessage=unknown%20error&id=ACk3y&originHeight=240&originWidth=1301&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
原因是由于线程的并发问题，大部分请求都执行到这一步，这个时候redis的数据还没有补充上去，所以导致了大量请求还是重新去查数据库，其实也类似于**缓存穿透问题**，当某个key失效时，大量请求会去查询数据库，那么这个问题应该如何解决呢？
![](https://img-blog.csdnimg.cn/img_convert/917002e9273aff37a8baba9d1bfdfa8e.png#averageHue=#312c2b&clientId=u94a77f1f-6df0-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=298&id=u09a15344&originHeight=372&originWidth=809&originalType=binary&ratio=1&rotation=0&showTitle=false&size=44333&status=done&style=none&taskId=ube2ba62e-d1e2-416b-85e1-66aa7b2e5a2&title=&width=647.2#errorMessage=unknown%20error&id=ctbhY&originHeight=372&originWidth=809&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)


### 缓存击穿问题（其中也解决了穿透问题）
当有一个访问量较高的key，在失效时，会导致大量的请求发向数据库，导致数据库崩溃
解决方式主要有：

- 加互斥锁
- 逻辑过期

#### 加互斥锁
![](https://img-blog.csdnimg.cn/img_convert/eb0901201ebc90b7593855bae4ac6cb7.png#averageHue=#f9f9f9&clientId=u420f32ae-cde3-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=511&id=u80415e8a&originHeight=639&originWidth=777&originalType=binary&ratio=1&rotation=0&showTitle=false&size=103600&status=done&style=none&taskId=u1b95166d-6b4e-43dd-859b-4eea5ade03c&title=&width=621.6#errorMessage=unknown%20error&id=aGvvK&originHeight=639&originWidth=777&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
```java
public TbShopEntity queryWithMutexLock(Long id) {
        TbShopEntity shop = null;
        //是否命中缓存
        String shopJson = redisClient.get(CACHE_SHOP_KEY + id);

        //如果命中缓存，则先判断缓存是否为”“，如果是返回null，否则返回结果
        if (shopJson != null) {
            //如果缓存为空的话，则直接返回结果
            if (StrUtil.isEmpty(shopJson)) return null;
            return JSONObject.parseObject(shopJson, TbShopEntity.class);
        }


        //获取锁，锁的粒度需要精确到id，不能太大
        RLock lock = redissonClient.getLock(LOCK_SHOP + id);
        try {
            //加锁
            boolean isLock = lock.tryLock(10,TimeUnit.SECONDS);

            //如果没有获取到锁，则休眠50ms，然后重试
            if (!isLock) {
                Thread.sleep(50);
                queryWithMutexLock(id);
            }

            //这里需要做doubleCheck，需要重新查询缓存的数据是否存在，否则还会出现重复查询数据库的情况
            shopJson = redisClient.get(CACHE_SHOP_KEY + id);

            if (shopJson != null) {
                if (StrUtil.isEmpty(shopJson)) return null;
                return JSONObject.parseObject(shopJson, TbShopEntity.class);
            }

            //如果缓存不存在，则查询数据库
            shop = getById(id);
            //如果数据库查询数据不存在，则直设置值为空，以及过期时间，直接返回null
            if (shop == null) {
                redisTemplate.opsForValue().set(CACHE_SHOP_KEY + id, StrUtil.EMPTY, SECKILL_SECONDS, TimeUnit.SECONDS);
                return null;
            }
            //如果数据库数据存在则设置到redis中
            redisClient.set(CACHE_SHOP_KEY + id, JSON.toJSONString(shop));

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }

        //2.返回数据
        return shop;
    }
```
#### 
压测：
使用200个线程测试，结果是只查询了一次，是ok的
![](https://img-blog.csdnimg.cn/img_convert/c2fd31464cbf016167c94febc146ed02.png#averageHue=#302f2e&clientId=u420f32ae-cde3-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=97&id=ube30ba09&originHeight=121&originWidth=1438&originalType=binary&ratio=1&rotation=0&showTitle=false&size=43591&status=done&style=none&taskId=u05964c02-87c3-42f1-9e8f-9b231330ee7&title=&width=1150.4#errorMessage=unknown%20error&id=Pbvr7&originHeight=121&originWidth=1438&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

#### 逻辑过期
逻辑过期的原理，指的是不使用redis自带的expire进行存储，而是在存储的数据中，添加一个过期字段，然后在获取数据的时候，进行该字段的判断，如果已经过期了，则返回旧的数据，**启动一个线程去更新新的数据**
**数据结构**如下：
![](https://img-blog.csdnimg.cn/img_convert/0970616db82eb79c99abee818cc7b78c.png#averageHue=#fdf9f8&clientId=u420f32ae-cde3-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=265&id=u63922d6f&originHeight=331&originWidth=574&originalType=binary&ratio=1&rotation=0&showTitle=false&size=31656&status=done&style=none&taskId=ua717d54a-85e8-4ed9-bf9a-1557833deae&title=&width=459.2#errorMessage=unknown%20error&id=BkW5u&originHeight=331&originWidth=574&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
data用来存储数据，expired存储过期时间，所以我们只要比较，如果expired小于当前时间的话，就代表该数据是过期的了
**逻辑图**如下：
这个逻辑图稍微比较复杂，基本将空值和互斥锁都加进去了
![](https://img-blog.csdnimg.cn/img_convert/2f12f39665590c577a1b1559e622c9cc.png#averageHue=#f8f8f7&clientId=ua929a256-c1ae-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=618&id=u6bb1cf61&originHeight=773&originWidth=831&originalType=binary&ratio=1&rotation=0&showTitle=false&size=104469&status=done&style=none&taskId=udff84e4e-5587-463d-80b8-37b857c3b1e&title=&width=664.8#errorMessage=unknown%20error&id=uEQcX&originHeight=773&originWidth=831&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
**queryWithLogicExpire**
```java
 public TbShopEntity queryWithLogicExpire(Long id) {
        TbShopEntity shop = null;

        //查询是否命中缓存
        String shopJson = redisClient.get(CACHE_SHOP_KEY + id);
        //如果命中,则判断结果是空置，还是过期的值，或者没过期的值
        if (shopJson != null) {
            //这里的代码往下滑查看
             return redisDTO2Entity(id, shopJson);
        }

        //获取锁，粒度具体到商户
        RLock lock = redissonClient.getLock("lock:shop:" + id);
        try {
            //加锁,10s过期
            boolean isLock = lock.tryLock(10,TimeUnit.SECONDS);

            // 如果没有获取到锁，则休眠50ms，然后重试
            if (!isLock) {
                Thread.sleep(50);
                queryWithLogicExpire(id);
            }

            //这里需要做doubleCheck，否则还会出现重复查询数据库的情况
            //这里先不做判断是否逻辑过期的逻辑
            shopJson = redisClient.get(CACHE_SHOP_KEY + id);
            if (shopJson != null) {
                redisDTO2Entity(id,shopJson);
            }

            shop = getById(id);
            //1.2 如果数据库查询数据不存在，则直设置值为空，以及过期时间，直接返回null
            if (shop == null) {
                redisTemplate.opsForValue().set(CACHE_SHOP_KEY + id, "", SECKILL_SECONDS, TimeUnit.MINUTES);
                return null;
            }
            //1.3 如果存在则设置到redis中
            redisClient.set(CACHE_SHOP_KEY + id, JSON.toJSONString(shop));

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }

        //2.返回数据
        return shop;
    }
```
**redisDTO2Entity**
```java
private TbShopEntity redisDTO2Entity(Long id,String shopJson){
        //如果是空值，则直接返回null
        if (StrUtil.isEmpty(shopJson)) return null;
        //或者则转换成redis实体类
        RedisDTO redisDTO = JSONObject.parseObject(shopJson, RedisDTO.class);
        //获取逻辑过期时间
        LocalDateTime expired = redisDTO.getExpired();
        // 判断时间是否过期，如果过期，则启动线程更新数据，其他直接返回
        if (expired.isBefore(LocalDateTime.now())) {
            //使用异步线程,更新数据
            saveShop(id);
        }
        return JSONObject.parseObject(JSON.toJSONString(redisDTO.getData()), TbShopEntity.class);
    }
```

使用异步线程进行数据的更新
**saveShop**
```java
    @Async
    public void saveShop(Long id){
        TbShopEntity entity = getById(id);
        if(entity==null) return;
        redisClient.setLogicExpired(RedisConstant.CACHE_SHOP_KEY+id,entity,RedisConstant.SECKILL_SECONDS, TimeUnit.SECONDS);
        log.info("线程{},更新商户信息",Thread.currentThread().getName());
    }
```

> 测试

- 需要先添加一条数据

执行下面的test方法，进行添加数据，添加成功后
数据如下：
![](https://img-blog.csdnimg.cn/img_convert/de1cc0f166a08cbe07cd9663fdfbaa52.png#averageHue=#fcfbfa&clientId=u420f32ae-cde3-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=266&id=u80ec8e44&originHeight=333&originWidth=451&originalType=binary&ratio=1&rotation=0&showTitle=false&size=26506&status=done&style=none&taskId=ub39d49dd-5faa-4ce4-abcd-19ffd2fcf54&title=&width=360.8#errorMessage=unknown%20error&id=bmJzQ&originHeight=333&originWidth=451&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
```java
package com.walker.dianping;

import com.walker.dianping.common.constants.RedisConstant;
import com.walker.dianping.common.utils.RedisClient;
import com.walker.dianping.model.TbShopEntity;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.concurrent.TimeUnit;

@SpringBootTest
public class RedisTest {

    @Autowired
    private RedisClient redisClient;

    @Test
    void addShop() {
        TbShopEntity tbShopEntity = new TbShopEntity();
        tbShopEntity.setId(1L);
        tbShopEntity.setName("逻辑过期测试商户");
        //该方法可以查看大纲，放在了完整代码中
        redisClient.setLogicExpired(RedisConstant.CACHE_SHOP_KEY + 1L, tbShopEntity, 20, TimeUnit.MINUTES);
    }
}

```

- 调用接口发起测试

![](https://img-blog.csdnimg.cn/img_convert/0e1089ec4df7c4400baf3a7357dec048.png#averageHue=#d6f3d3&clientId=u420f32ae-cde3-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=46&id=u5a2c3603&originHeight=58&originWidth=536&originalType=binary&ratio=1&rotation=0&showTitle=false&size=4862&status=done&style=none&taskId=u392635ef-a80f-4548-a859-73b6ab706f2&title=&width=428.8#errorMessage=unknown%20error&id=fTpqf&originHeight=58&originWidth=536&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
因为一开始是有的，所以可以直接拿到数据

- 当过期时间expired小于当前时间时，这个时候重新去调用接口（可以直接更改redis的数据）

这个时候就会去更新商户的数据了，这个时候就能拿到新的数据了
![](https://img-blog.csdnimg.cn/img_convert/812d5cee4a2520ad7880b5e7312eefda.png#averageHue=#323130&clientId=u420f32ae-cde3-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=34&id=u5ec31725&originHeight=43&originWidth=455&originalType=binary&ratio=1&rotation=0&showTitle=false&size=4272&status=done&style=none&taskId=u4f6500be-751e-42cf-8479-020fd7ddeec&title=&width=364#errorMessage=unknown%20error&id=gFgJX&originHeight=43&originWidth=455&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
![](https://img-blog.csdnimg.cn/img_convert/f8adf761f460e41cb1e2d9928fb35f26.png#averageHue=#fcf8f7&clientId=u420f32ae-cde3-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=287&id=ucf9f8a69&originHeight=359&originWidth=517&originalType=binary&ratio=1&rotation=0&showTitle=false&size=31104&status=done&style=none&taskId=ufb6a951e-6f2d-4fff-ba59-1505ac9c854&title=&width=413.6#errorMessage=unknown%20error&id=nLDjn&originHeight=359&originWidth=517&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

### 完整代码
#### controller
```java
package com.walker.dianping.controller;


import com.walker.dianping.model.R;
import com.walker.dianping.service.TbShopService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;

import org.springframework.web.bind.annotation.RestController;

/**
 * <p>
 *  前端控制器
 * </p>

 *
 * @author walker
 * @since 2023-01-18
 */
@RestController
@RequestMapping("/tb-shop-entity")
public class TbShopController {

    @Autowired
    private TbShopService tbShopService;

    /**
    * 获取商铺
    */
    @GetMapping("/shop/{id}")
    public R getShop(@PathVariable(value = "id") Long id){
        return tbShopService.getShop(id);
    }
}

```

#### service
```java
package com.walker.dianping.service;


import com.baomidou.mybatisplus.extension.service.IService;
import com.walker.dianping.model.R;
import com.walker.dianping.model.TbShopEntity;

/**
 * <p>
 *  服务类
 * </p>

 *
 * @author walker
 * @since 2023-01-18
 */
public interface TbShopService extends IService<TbShopEntity> {

    R getShop(Long id);
}

```
#### TbShopServiceImpl
```java
package com.walker.dianping.service.impl;

import cn.hutool.core.bean.BeanUtil;
import cn.hutool.core.util.StrUtil;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.walker.dianping.common.constants.RedisConstant;
import com.walker.dianping.common.utils.RedisClient;
import com.walker.dianping.mapper.TbShopMapper;
import com.walker.dianping.model.R;
import com.walker.dianping.model.TbShopEntity;
import com.walker.dianping.model.dto.RedisDTO;
import com.walker.dianping.service.TbShopService;
import lombok.extern.slf4j.Slf4j;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.concurrent.TimeUnit;

import static com.walker.dianping.common.constants.RedisConstant.*;

@Slf4j
@Service
public class TbShopServiceImpl extends ServiceImpl<TbShopMapper, TbShopEntity> implements TbShopService {


    @Autowired
    private StringRedisTemplate redisTemplate;

    @Autowired
    private RedisClient redisClient;

    @Autowired
    private RedissonClient redissonClient;


    @Override
    public R getShop(Long id) {
        //初始查询
//        TbShopEntity shop = rawQuery(id);
        //缓存穿透
//        TbShopEntity shop = queryWithPassThrough(id);

        //解决缓存击穿问题
//        TbShopEntity shop = queryWithMutexLock(id);

        //解决缓存击穿+穿透问题，使用逻辑过期
        TbShopEntity shop = queryWithLogicExpire(id);

        if (shop == null) {
            return R.fail("店铺不存在");
        }
        return R.ok(shop);
    }


    /**
     * 最初始的版本
     */
    public TbShopEntity rawQuery(Long id) {
        TbShopEntity shop = null;
        //1.是否命中缓存
        String shopJson = redisClient.get(CACHE_SHOP_KEY + id);
        //1.1 如果命中，且数据非空，则直接返回
        if (!StrUtil.isEmpty(shopJson)) {
            return JSONObject.parseObject(shopJson, TbShopEntity.class);
        }

        //1.2 查询数据库
        shop = getById(id);
        //1.2.1 如果数据库不存在，则直接返回
        if (shop == null) return null;
        //1.2.2 如果存在则设置到redis中
        redisClient.set(CACHE_SHOP_KEY + id, JSON.toJSONString(shop));
        //2.返回数据
        return shop;
    }


    /**
     * 缓存穿透解决方案, 1、设置一个空值，且带有较短的过期时间 2、布隆过滤器
     * <p>
     * 但是目前还是没有解决穿透的问题的，因为该key不存在，所以会有多个请求去直接请求数据库(并发问题)，从而需要进行优化，解决缓存穿透问题
     */
    private TbShopEntity queryWithPassThrough(Long id) {
        //1.是否命中缓存
        String shopJson = redisClient.get(CACHE_SHOP_KEY + id);

        //1.1 如果命中缓存，则直接返回
        //这里不能使用！Strutil.isEmpty去判断，因为在下边会使用”“去存空值
        if (shopJson != null) {
            //如果返回的值是""，则直接返回null
            if (StrUtil.isEmpty(shopJson)) return null;
            //否则则返回结果
            return JSONObject.parseObject(shopJson, TbShopEntity.class);
        }

        //1.2 如果没有命中,查询数据库
        TbShopEntity shop = getById(id);

        //1.2.1 如果数据库查询数据不存在，则直设置值为空，以及过期时间,现在是设置为10s，如果10s已经过，再重新查询
        if (shop == null) {
            redisTemplate.opsForValue().set(CACHE_SHOP_KEY + id, StrUtil.EMPTY, SECKILL_SECONDS, TimeUnit.SECONDS);
            return null;
        }

        //1.2.2 如果数据库存在，则设置到redis中
        redisClient.set(CACHE_SHOP_KEY + id, JSON.toJSONString(shop));
        //并转成shop返回

        //2.返回数据
        return JSONObject.parseObject(shopJson, TbShopEntity.class);
    }


    /**
     * 缓存击穿，key失效，导致高并发的请求
     * 加锁：可以使用redisson
     */
    public TbShopEntity queryWithMutexLock(Long id) {
        TbShopEntity shop = null;
        //是否命中缓存
        String shopJson = redisClient.get(CACHE_SHOP_KEY + id);

        //如果命中缓存，则先判断缓存是否为”“，如果是返回null，否则返回结果
        if (shopJson != null) {
            //如果缓存为空的话，则直接返回结果
            if (StrUtil.isEmpty(shopJson)) return null;
            return JSONObject.parseObject(shopJson, TbShopEntity.class);
        }


        //获取锁，锁的粒度需要精确到id，不能太大
        RLock lock = redissonClient.getLock(LOCK_SHOP + id);
        try {
            //加锁
            boolean isLock = lock.tryLock(10,TimeUnit.SECONDS);

            //如果没有获取到锁，则休眠50ms，然后重试
            if (!isLock) {
                Thread.sleep(50);
                queryWithMutexLock(id);
            }

            //这里需要做doubleCheck，需要重新查询缓存的数据是否存在，否则还会出现重复查询数据库的情况
            shopJson = redisClient.get(CACHE_SHOP_KEY + id);

            if (shopJson != null) {
                if (StrUtil.isEmpty(shopJson)) return null;
                return JSONObject.parseObject(shopJson, TbShopEntity.class);
            }

            //如果缓存不存在，则查询数据库
            shop = getById(id);
            //如果数据库查询数据不存在，则直设置值为空，以及过期时间，直接返回null
            if (shop == null) {
                redisTemplate.opsForValue().set(CACHE_SHOP_KEY + id, StrUtil.EMPTY, SECKILL_SECONDS, TimeUnit.SECONDS);
                return null;
            }
            //如果数据库数据存在则设置到redis中
            redisClient.set(CACHE_SHOP_KEY + id, JSON.toJSONString(shop));

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }

        //2.返回数据
        return shop;
    }


    private TbShopEntity redisDTO2Entity(Long id,String shopJson){
        //如果是空值，则直接返回null
        if (StrUtil.isEmpty(shopJson)) return null;
        //或者则转换成redis实体类
        RedisDTO redisDTO = JSONObject.parseObject(shopJson, RedisDTO.class);
        //获取逻辑过期时间
        LocalDateTime expired = redisDTO.getExpired();
        // 判断时间是否过期，如果过期，则启动线程更新数据，其他直接返回
        if (expired.isBefore(LocalDateTime.now())) {
            //使用异步线程,更新数据
            saveShop(id);
        }
        return JSONObject.parseObject(JSON.toJSONString(redisDTO.getData()), TbShopEntity.class);
    }

    /**
     * 缓存击穿解决方式二：逻辑过期
     */
    public TbShopEntity queryWithLogicExpire(Long id) {
        TbShopEntity shop = null;

        //查询是否命中缓存
        String shopJson = redisClient.get(CACHE_SHOP_KEY + id);
        //如果命中,则判断结果是空置，还是过期的值，或者没过期的值
        if (shopJson != null) {
             return redisDTO2Entity(id, shopJson);
        }

        //获取锁，粒度具体到商户
        RLock lock = redissonClient.getLock("lock:shop:" + id);
        try {
            //加锁,10s过期
            boolean isLock = lock.tryLock(10,TimeUnit.SECONDS);

            // 如果没有获取到锁，则休眠50ms，然后重试
            if (!isLock) {
                Thread.sleep(50);
                queryWithLogicExpire(id);
            }

            //这里需要做doubleCheck，否则还会出现重复查询数据库的情况
            //这里先不做判断是否逻辑过期的逻辑
            shopJson = redisClient.get(CACHE_SHOP_KEY + id);
            if (shopJson != null) {
                redisDTO2Entity(id,shopJson);
            }

            shop = getById(id);
            //1.2 如果数据库查询数据不存在，则直设置值为空，以及过期时间，直接返回null
            if (shop == null) {
                redisTemplate.opsForValue().set(CACHE_SHOP_KEY + id, "", SECKILL_SECONDS, TimeUnit.MINUTES);
                return null;
            }
            //1.3 如果存在则设置到redis中
            redisClient.set(CACHE_SHOP_KEY + id, JSON.toJSONString(shop));

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }

        //2.返回数据
        return shop;
    }

    @Async
    public void saveShop(Long id) {
        //查询数据
        TbShopEntity entity = getById(id);
        if (entity == null) return;
        redisClient.setLogicExpired(RedisConstant.CACHE_SHOP_KEY + id, entity, RedisConstant.SECKILL_SECONDS, TimeUnit.SECONDS);
        log.info("线程{},更新商户信息", Thread.currentThread().getName());
    }
}

```

#### RedisClient代码
```java
package com.walker.dianping.common.utils;

import cn.hutool.core.bean.BeanUtil;
import cn.hutool.core.util.StrUtil;
import com.alibaba.fastjson.JSON;
import com.walker.dianping.model.dto.RedisDTO;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.convert.RedisData;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.concurrent.TimeUnit;

@Slf4j
@Component
public class RedisClient {

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
    * 定义set方法
    */
    public void set(String key,Object value){
        redisTemplate.opsForValue().set(key, JSON.toJSONString(value));
    }

    /**
     * 定义set方法
     */
    public void setLogicExpired(String key,Object value,long timeout, TimeUnit unit){
        RedisDTO dto = new RedisDTO();
        dto.setData(value);
        dto.setExpired(LocalDateTime.now().plusSeconds(unit.toSeconds(timeout)));
        redisTemplate.opsForValue().set(key, JSON.toJSONString(dto));
    }



    /**
    * get方法
    */
    public String get(String key){
        String s = redisTemplate.opsForValue().get(key);
        return s;
    }

}

```
#### RedisConstant
```java
package com.walker.dianping.common.constants;

public interface RedisConstant {

    String SECKILL_LUA_SCRIPT="seckill.lua";
    String CACHE_SHOP_KEY= "cache:shop:";
    Integer SECKILL_SECONDS=10;
    String LOCK_SHOP="lock:shop:";
}

```
