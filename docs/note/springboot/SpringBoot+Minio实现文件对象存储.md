# 简介

Minio 是一个基于 Apache License v2.0 开源协议的对象存储服务。它兼容亚马逊 S3 云存储服务接口，非常适合于存储大容量非结构化的数据，例如图片、视频、日志文件、备份数据和容器/虚拟机镜像等，而一个对象文件可以是任意大小，从几 kb 到最大 5T 不等。
Minio 是一个非常轻量的服务,可以很简单的和其他应用的结合，类似 NodeJS, Redis 或者 MySQL。

#

# 安装

## docker-compose 安装

### 步骤

#### 1、创建 minio 文件夹

`mkdir minio`

#### 2、创建 data config 文件夹

进入 minio 文件夹中`cd minio/`
创建 data 和 config 文件夹
`mkdir data`
`mkdir config`

#### 3、在 minio 文件夹中创建 docker-compose.yaml 文件

```yaml
version: "3"

services:
  minio:
    image: minio/minio:RELEASE.2022-03-22T02-05-10Z
    # 将data config文件夹挂在到当前目录的data config
    volumes:
      - "./data:/data"
      - "./config:/root/.minio"
    # 账号密码配置
    environment:
      - MINIO_ACCESS_KEY=admin
      - MINIO_SECRET_KEY=123456
    # 端口配置
    ports:
      - "9000:9000"
      - "9001:9001"
    command: server /data --console-address ":9001"
    restart: always
```

复制的时候，可以复制下面这个文件，因为上面文件使用注释的，所以可能会导致格式混乱

<!-- more -->

```yaml
version: "3"

services:
  minio:
    image: minio/minio:RELEASE.2022-03-22T02-05-10Z
    volumes:
      - "./data:/data"
      - "./config:/root/.minio"
    environment:
      - MINIO_ACCESS_KEY=admin
      - MINIO_SECRET_KEY=12345678
    ports:
      - "9000:9000"
      - "9001:9001"
    command: server /data --console-address ":9001"
    restart: always
```

#### 4、启动

`docker-compose up -d`

![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/d620a7b98530f167e69968f7517a3e67.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=259&id=uc4f6f4fd&originHeight=259&originWidth=720&originalType=binary&ratio=1&rotation=0&showTitle=false&size=21711&status=done&style=none&taskId=uc6db2230-885b-4170-88d6-c5b84a0bb14&title=&width=720#errorMessage=unknown%20error&id=NYNu2&originHeight=259&originWidth=720&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none ":crossorgin")
之后等待下载
下载完成后，会显示该页面

![](https://img-blog.csdnimg.cn/img_convert/bdb3135a6ead5d167681d652334418de.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=29&id=u92d934a9&originHeight=29&originWidth=306&originalType=binary&ratio=1&rotation=0&showTitle=false&size=2211&status=done&style=none&taskId=u98d8b682-5046-4ad2-8a21-86bf963b4a4&title=&width=306#errorMessage=unknown%20error&id=GkoPP&originHeight=29&originWidth=306&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

#### 5、检查是否启动成功

使用`docker ps`
![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/4256012c2a834240e216fe544bdcee42.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=83&id=u41c2c16c&originHeight=83&originWidth=1141&originalType=binary&ratio=1&rotation=0&showTitle=false&size=9033&status=done&style=none&taskId=u6226a37e-412d-4a1b-9394-60b8670bb63&title=&width=1141#errorMessage=unknown%20error&id=AYd8A&originHeight=83&originWidth=1141&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
如果是 up 则代表 ok 了，如果不是的话，则在 minio 文件夹内，执行`docker-compose logs -f`查看日志

#### 6、访问连接

访问 ip:9001 即可
![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/dee6cc3f94100341c98c18b90bd16972.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=800&id=u0404c9a6&originHeight=800&originWidth=1919&originalType=binary&ratio=1&rotation=0&showTitle=false&size=380374&status=done&style=none&taskId=ub1837e1a-c7bc-4fe6-a66c-711aff486ec&title=&width=1919#errorMessage=unknown%20error&id=v1kI9&originHeight=800&originWidth=1919&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
访问成功后的页面
之后使用 docker-compose.yaml 的配置文件的密码登录
![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/d01c408e7e0fd37701456fe1908532dc.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=74&id=u592256d6&originHeight=74&originWidth=459&originalType=binary&ratio=1&rotation=0&showTitle=false&size=5730&status=done&style=none&taskId=ua8061e65-9a5d-4781-932f-dafbe172495&title=&width=459#errorMessage=unknown%20error&id=UTX9D&originHeight=74&originWidth=459&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
登录后页面
![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/8d0d3598e92c8259e819c7b0c6a5f129.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=650&id=u9c047770&originHeight=650&originWidth=1919&originalType=binary&ratio=1&rotation=0&showTitle=false&size=137501&status=done&style=none&taskId=u0da8d3fb-9d72-4a1c-8190-6f9a7d5f8c5&title=&width=1919#errorMessage=unknown%20error&id=uCqgi&originHeight=650&originWidth=1919&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

### 遇到问题

1、密码至少要 8 位
![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/d787c7019414a0fb5e790d06f1b78a48.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=94&id=u8d6f7361&originHeight=94&originWidth=691&originalType=binary&ratio=1&rotation=0&showTitle=false&size=8573&status=done&style=none&taskId=u1280faef-2761-432d-bebb-1e34a94c9b0&title=&width=691#errorMessage=unknown%20error&id=ubRg6&originHeight=94&originWidth=691&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

# minio 使用

## 创建 bucket，上传图片，并访问

### 1、点击创建

![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/ee9361f595504bf9eed7a99e1124fe13.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=454&id=u0dd28bc0&originHeight=454&originWidth=1919&originalType=binary&ratio=1&rotation=0&showTitle=false&size=101848&status=done&style=none&taskId=u0384dcf1-9a1b-4de1-96c4-829aeec1542&title=&width=1919#errorMessage=unknown%20error&id=VmCDy&originHeight=454&originWidth=1919&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

### 2、创建 bucket

![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/08ecd530813cf930042a89d1e754f948.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=563&id=uece23bae&originHeight=563&originWidth=1652&originalType=binary&ratio=1&rotation=0&showTitle=false&size=41059&status=done&style=none&taskId=uf05ff539-71f5-4415-aea4-1f26e7b7981&title=&width=1652#errorMessage=unknown%20error&id=ugfuZ&originHeight=563&originWidth=1652&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
目前我们这里的配置都选择默认，也就是不配置
Versioning：多版本控制
**Object Locking：对象锁定**
**Quota：限制数量**
创建后，桶的文件是空的，我们可以尝试进行上传
![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/54d7fea04df7bbd0b1e3092391f59236.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=351&id=u6679075b&originHeight=351&originWidth=1651&originalType=binary&ratio=1&rotation=0&showTitle=false&size=18623&status=done&style=none&taskId=uea995b9b-c5fa-4736-8844-60a0e13023c&title=&width=1651#errorMessage=unknown%20error&id=yhKgh&originHeight=351&originWidth=1651&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

### 3、测试上传文件

![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/4bcd494f32cc7a023900d16a3e40c1ff.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=341&id=u80fcb924&originHeight=341&originWidth=1666&originalType=binary&ratio=1&rotation=0&showTitle=false&size=25490&status=done&style=none&taskId=u8d744717-1abd-4701-ba8b-a9be526e561&title=&width=1666#errorMessage=unknown%20error&id=Od6iw&originHeight=341&originWidth=1666&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/f79266196fc969090fae7b223e6cc1e3.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=203&id=u81718cdf&originHeight=203&originWidth=407&originalType=binary&ratio=1&rotation=0&showTitle=false&size=10363&status=done&style=none&taskId=u8c8dd06d-37b3-4239-b6db-d41446ce4a6&title=&width=407#errorMessage=unknown%20error&id=jnRWE&originHeight=203&originWidth=407&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
上传之后，就会生成一条数据了
![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/6a85f0b8e87432498325ad461d0faa82.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=207&id=uc8603e16&originHeight=207&originWidth=1255&originalType=binary&ratio=1&rotation=0&showTitle=false&size=16541&status=done&style=none&taskId=ua0d99a85-df0c-4df0-9e59-29bc1a516be&title=&width=1255#errorMessage=unknown%20error&id=R1tOn&originHeight=207&originWidth=1255&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
之后我们点击进入图片
![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/d6852bbebd29512311f0bbc16834b1b3.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=829&id=u11ec752c&originHeight=829&originWidth=1651&originalType=binary&ratio=1&rotation=0&showTitle=false&size=50841&status=done&style=none&taskId=ub1ee2691-21d2-498e-adf6-e84de27a980&title=&width=1651#errorMessage=unknown%20error&id=MxixK&originHeight=829&originWidth=1651&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

### 4、生成链接并访问

#### 临时链接

![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/deb3c37a399de8959acf362dd26c137e.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=498&id=uef0064da&originHeight=498&originWidth=1368&originalType=binary&ratio=1&rotation=0&showTitle=false&size=55553&status=done&style=none&taskId=u6f41334f-1f4d-4fa0-8435-1003fa1d9a4&title=&width=1368#errorMessage=unknown%20error&id=q9P1k&originHeight=498&originWidth=1368&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
之后点击临时链接访问即可
但是需要注意，复制出来的链接可能会无法访问，这个原因暂时也不是特别清楚，所以我们采用永久链接进行处理

#### 永久链接

1、开放权限
![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/9c838f1a00cd057276da51525751f58b.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=417&id=u6505180c&originHeight=417&originWidth=1919&originalType=binary&ratio=1&rotation=0&showTitle=false&size=102939&status=done&style=none&taskId=u20ae018a-e030-4016-8463-34396429484&title=&width=1919#errorMessage=unknown%20error&id=LwnTe&originHeight=417&originWidth=1919&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/e1606d5828a20d2a473427515c0e769f.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=534&id=u80c9de4b&originHeight=534&originWidth=1653&originalType=binary&ratio=1&rotation=0&showTitle=false&size=39937&status=done&style=none&taskId=u2692828c-563b-4182-8262-e551a6de616&title=&width=1653#errorMessage=unknown%20error&id=obHVQ&originHeight=534&originWidth=1653&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/980458a6359fa9a80be397a965cfeb6d.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=560&id=uc1901887&originHeight=560&originWidth=1657&originalType=binary&ratio=1&rotation=0&showTitle=false&size=25834&status=done&style=none&taskId=uca855728-8569-4ff0-9b13-4637498c7e9&title=&width=1657#errorMessage=unknown%20error&id=kz0xk&originHeight=560&originWidth=1657&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/9728eb2a52cf6978e4a8980c7afcf5b8.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=332&id=uac666d81&originHeight=332&originWidth=943&originalType=binary&ratio=1&rotation=0&showTitle=false&size=17887&status=done&style=none&taskId=u3141f7db-364f-455d-8819-cc372539665&title=&width=943#errorMessage=unknown%20error&id=TwOzt&originHeight=332&originWidth=943&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

# springboot 整合 minio

## 步骤

### 1、导入依赖

```xml
 <dependency>
            <groupId>io.minio</groupId>

            <artifactId>minio</artifactId>

            <version>8.0.0</version>

        </dependency>

```

### 2、添加配置

可以在 yaml 中配置参数，当然，如果有多环境的话，需要进行多环境配置
![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/113b6c717485aec79a8c3df76174a496.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=625&id=u956bd56b&originHeight=625&originWidth=1301&originalType=binary&ratio=1&rotation=0&showTitle=false&size=52627&status=done&style=none&taskId=uf11b1e8e-db24-45ef-8a70-f689becf7c6&title=&width=1301#errorMessage=unknown%20error&id=W6Ncg&originHeight=625&originWidth=1301&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

```yaml
minio:
  endpoint: http://192.168.56.10:9000
  accessKey: admin
  secretKey: 12345678
  bucketName: walkertest
```

或者如果你觉得麻烦，想直接写死在代码中也是可以的，但是不是很建议那样做

### 3、编写配置类和工具类

#### 配置类

```java
// 使用配置类注解
@Configuration
public class MinioConfig {

//    获取yaml文件中的数据
    @Value("${minio.endpoint}")
    private String endpoint;
    @Value("${minio.accessKey}")
    private String accessKey;
    @Value("${minio.secretKey}")
    private String secretKey;
    @Value("${minio.bucketName}")
    private String bucketName;


//    注入容器中
    @Bean
    public MinIOUtils createMinioClient(){
        return new MinIOUtils(endpoint, bucketName, accessKey, secretKey);
    }

}

```

#### 工具类

```java
package com.walker.springbootdemo.util;

import io.minio.*;
import io.minio.http.Method;
import io.minio.messages.Bucket;
import io.minio.messages.DeleteObject;
import io.minio.messages.Item;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.web.multipart.MultipartFile;
import java.io.ByteArrayInputStream;
import java.io.InputStream;
import java.io.UnsupportedEncodingException;
import java.net.URLDecoder;
import java.util.ArrayList;
import java.util.LinkedList;
import java.util.List;
import java.util.Optional;

/**
 * MinIO工具类
 */
@Slf4j
@Component
public class MinIOUtils {

    private static MinioClient minioClient;

    private static String endpoint;
    private static String bucketName;
    private static String accessKey;
    private static String secretKey;


    private static final String SEPARATOR = "/";

    public MinIOUtils() {
    }

    public MinIOUtils(String endpoint, String bucketName, String accessKey, String secretKey) {
        MinIOUtils.endpoint = endpoint;
        MinIOUtils.bucketName = bucketName;
        MinIOUtils.accessKey = accessKey;
        MinIOUtils.secretKey = secretKey;
        createMinioClient();
    }

    /**
     * 创建基于Java端的MinioClient
     */
    public void createMinioClient() {
        try {
            if (null == minioClient) {
                log.info("开始创建 MinioClient...");
                minioClient = MinioClient
                                .builder()
                                .endpoint(endpoint)
                                .credentials(accessKey, secretKey)
                                .build();
                createBucket(bucketName);
                log.info("创建完毕 MinioClient...");
            }
        } catch (Exception e) {
            log.error("MinIO服务器异常：{}", e);
        }
    }

    /**
     * 获取上传文件前缀路径
     * @return
     */
    public static String getBasisUrl() {
        return endpoint + SEPARATOR + bucketName + SEPARATOR;
    }

    /******************************  Operate Bucket Start  ******************************/

    /**
     * 启动SpringBoot容器的时候初始化Bucket
     * 如果没有Bucket则创建
     * @throws Exception
     */
    private static void createBucket(String bucketName) throws Exception {
        if (!bucketExists(bucketName)) {
            minioClient.makeBucket(MakeBucketArgs.builder().bucket(bucketName).build());
        }
    }

    /**
     *  判断Bucket是否存在，true：存在，false：不存在
     * @return
     * @throws Exception
     */
    public static boolean bucketExists(String bucketName) throws Exception {
        return minioClient.bucketExists(BucketExistsArgs.builder().bucket(bucketName).build());
    }


    /**
     * 获得Bucket的策略
     * @param bucketName
     * @return
     * @throws Exception
     */
    public static String getBucketPolicy(String bucketName) throws Exception {
        String bucketPolicy = minioClient
                                .getBucketPolicy(
                                        GetBucketPolicyArgs
                                                .builder()
                                                .bucket(bucketName)
                                                .build()
                                );
        return bucketPolicy;
    }


    /**
     * 获得所有Bucket列表
     * @return
     * @throws Exception
     */
    public static List<Bucket> getAllBuckets() throws Exception {
        return minioClient.listBuckets();
    }

    /**
     * 根据bucketName获取其相关信息
     * @param bucketName
     * @return
     * @throws Exception
     */
    public static Optional<Bucket> getBucket(String bucketName) throws Exception {
        return getAllBuckets().stream().filter(b -> b.name().equals(bucketName)).findFirst();
    }

    /**
     * 根据bucketName删除Bucket，true：删除成功； false：删除失败，文件或已不存在
     * @param bucketName
     * @throws Exception
     */
    public static void removeBucket(String bucketName) throws Exception {
        minioClient.removeBucket(RemoveBucketArgs.builder().bucket(bucketName).build());
    }

    /******************************  Operate Bucket End  ******************************/


    /******************************  Operate Files Start  ******************************/

    /**
     * 判断文件是否存在
     * @param bucketName 存储桶
     * @param objectName 文件名
     * @return
     */
    public static boolean isObjectExist(String bucketName, String objectName) {
        boolean exist = true;
        try {
            minioClient.statObject(StatObjectArgs.builder().bucket(bucketName).object(objectName).build());
        } catch (Exception e) {
            exist = false;
        }
        return exist;
    }

    /**
     * 判断文件夹是否存在
     * @param bucketName 存储桶
     * @param objectName 文件夹名称
     * @return
     */
    public static boolean isFolderExist(String bucketName, String objectName) {
        boolean exist = false;
        try {
            Iterable<Result<Item>> results = minioClient.listObjects(
                    ListObjectsArgs.builder().bucket(bucketName).prefix(objectName).recursive(false).build());
            for (Result<Item> result : results) {
                Item item = result.get();
                if (item.isDir() && objectName.equals(item.objectName())) {
                    exist = true;
                }
            }
        } catch (Exception e) {
            exist = false;
        }
        return exist;
    }

    /**
     * 根据文件前缀查询文件
     * @param bucketName 存储桶
     * @param prefix 前缀
     * @param recursive 是否使用递归查询
     * @return MinioItem 列表
     * @throws Exception
     */
    public static List<Item> getAllObjectsByPrefix(String bucketName,
                                                   String prefix,
                                                   boolean recursive) throws Exception {
        List<Item> list = new ArrayList<>();
        Iterable<Result<Item>> objectsIterator = minioClient.listObjects(
                ListObjectsArgs.builder().bucket(bucketName).prefix(prefix).recursive(recursive).build());
        if (objectsIterator != null) {
            for (Result<Item> o : objectsIterator) {
                Item item = o.get();
                list.add(item);
            }
        }
        return list;
    }

    /**
     * 获取文件流
     * @param bucketName 存储桶
     * @param objectName 文件名
     * @return 二进制流
     */
    public static InputStream getObject(String bucketName, String objectName) throws Exception {
        return minioClient.getObject(GetObjectArgs.builder().bucket(bucketName).object(objectName).build());
    }

    /**
     * 断点下载
     * @param bucketName 存储桶
     * @param objectName 文件名称
     * @param offset 起始字节的位置
     * @param length 要读取的长度
     * @return 二进制流
     */
    public InputStream getObject(String bucketName, String objectName, long offset, long length)throws Exception {
        return minioClient.getObject(
                GetObjectArgs.builder()
                        .bucket(bucketName)
                        .object(objectName)
                        .offset(offset)
                        .length(length)
                        .build());
    }

    /**
     * 获取路径下文件列表
     * @param bucketName 存储桶
     * @param prefix 文件名称
     * @param recursive 是否递归查找，false：模拟文件夹结构查找
     * @return 二进制流
     */
    public static Iterable<Result<Item>> listObjects(String bucketName, String prefix,
                                                     boolean recursive) {
        return minioClient.listObjects(
                ListObjectsArgs.builder()
                        .bucket(bucketName)
                        .prefix(prefix)
                        .recursive(recursive)
                        .build());
    }

    /**
     * 使用MultipartFile进行文件上传
     * @param bucketName 存储桶
     * @param file 文件名
     * @param objectName 对象名
     * @param contentType 类型
     * @return
     * @throws Exception
     */
    public static ObjectWriteResponse uploadFile(String bucketName, MultipartFile file,
                                                String objectName, String contentType) throws Exception {
        InputStream inputStream = file.getInputStream();
        return minioClient.putObject(
                PutObjectArgs.builder()
                        .bucket(bucketName)
                        .object(objectName)
                        .contentType(contentType)
                        .stream(inputStream, inputStream.available(), -1)
                        .build());
    }

    /**
     * 上传本地文件
     * @param bucketName 存储桶
     * @param objectName 对象名称
     * @param fileName 本地文件路径
     */
    public static ObjectWriteResponse uploadFile(String bucketName, String objectName,
                                                String fileName) throws Exception {
        return minioClient.uploadObject(
                UploadObjectArgs.builder()
                        .bucket(bucketName)
                        .object(objectName)
                        .filename(fileName)
                        .build());
    }

    /**
     * 通过流上传文件
     *
     * @param bucketName 存储桶
     * @param objectName 文件对象
     * @param inputStream 文件流
     */
    public static ObjectWriteResponse uploadFile(String bucketName, String objectName, InputStream inputStream) throws Exception {
        return minioClient.putObject(
                PutObjectArgs.builder()
                        .bucket(bucketName)
                        .object(objectName)
                        .stream(inputStream, inputStream.available(), -1)
                        .build());
    }

    /**
     * 创建文件夹或目录
     * @param bucketName 存储桶
     * @param objectName 目录路径
     */
    public static ObjectWriteResponse createDir(String bucketName, String objectName) throws Exception {
        return minioClient.putObject(
                PutObjectArgs.builder()
                        .bucket(bucketName)
                        .object(objectName)
                        .stream(new ByteArrayInputStream(new byte[]{}), 0, -1)
                        .build());
    }

    /**
     * 获取文件信息, 如果抛出异常则说明文件不存在
     *
     * @param bucketName 存储桶
     * @param objectName 文件名称
     */
    public static String getFileStatusInfo(String bucketName, String objectName) throws Exception {
        return minioClient.statObject(
                StatObjectArgs.builder()
                        .bucket(bucketName)
                        .object(objectName)
                        .build()).toString();
    }

    /**
     * 拷贝文件
     *
     * @param bucketName 存储桶
     * @param objectName 文件名
     * @param srcBucketName 目标存储桶
     * @param srcObjectName 目标文件名
     */
    public static ObjectWriteResponse copyFile(String bucketName, String objectName,
                                                 String srcBucketName, String srcObjectName) throws Exception {
        return minioClient.copyObject(
                CopyObjectArgs.builder()
                        .source(CopySource.builder().bucket(bucketName).object(objectName).build())
                        .bucket(srcBucketName)
                        .object(srcObjectName)
                        .build());
    }

    /**
     * 删除文件
     * @param bucketName 存储桶
     * @param objectName 文件名称
     */
    public static void removeFile(String bucketName, String objectName) throws Exception {
        minioClient.removeObject(
                RemoveObjectArgs.builder()
                        .bucket(bucketName)
                        .object(objectName)
                        .build());
    }

    /**
     * 批量删除文件
     * @param bucketName 存储桶
     * @param keys 需要删除的文件列表
     * @return
     */
    public static void removeFiles(String bucketName, List<String> keys) {
        List<DeleteObject> objects = new LinkedList<>();
        keys.forEach(s -> {
            objects.add(new DeleteObject(s));
            try {
                removeFile(bucketName, s);
            } catch (Exception e) {
                log.error("批量删除失败！error:{}",e);
            }
        });
    }

    /**
     * 获取文件外链
     * @param bucketName 存储桶
     * @param objectName 文件名
     * @param expires 过期时间 <=7 秒 （外链有效时间（单位：秒））
     * @return url
     * @throws Exception
     */
    public static String getPresignedObjectUrl(String bucketName, String objectName, Integer expires) throws Exception {
        GetPresignedObjectUrlArgs args = GetPresignedObjectUrlArgs.builder().expiry(expires).bucket(bucketName).object(objectName).build();
        return minioClient.getPresignedObjectUrl(args);
    }

    /**
     * 获得文件外链
     * @param bucketName
     * @param objectName
     * @return url
     * @throws Exception
     */
    public static String getPresignedObjectUrl(String bucketName, String objectName) throws Exception {
        GetPresignedObjectUrlArgs args = GetPresignedObjectUrlArgs.builder()
                                                                    .bucket(bucketName)
                                                                    .object(objectName)
                                                                    .method(Method.GET).build();
        return minioClient.getPresignedObjectUrl(args);
    }

    /**
     * 将URLDecoder编码转成UTF8
     * @param str
     * @return
     * @throws UnsupportedEncodingException
     */
    public static String getUtf8ByURLDecoder(String str) throws UnsupportedEncodingException {
        String url = str.replaceAll("%(?![0-9a-fA-F]{2})", "%25");
        return URLDecoder.decode(url, "UTF-8");
    }

    /******************************  Operate Files End  ******************************/

}


```

### 4、上传文件测试代码编写

```java
package com.walker.springbootdemo.controller;

import com.walker.springbootdemo.common.config.MinioConfig;
import com.walker.springbootdemo.util.MinIOUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.multipart.MultipartFile;

@RestController
@RequestMapping("/minio")
public class MinioController {

//    注入配置类
    @Autowired
    private MinioConfig minioConfig;


//    编写上传接口
    @PostMapping("/upload")
    public String upload(MultipartFile file) throws Exception {


        MinIOUtils.uploadFile(minioConfig.getBucketName(),file.getOriginalFilename(),file.getInputStream());
        String format="%s/%s/%s";

//        返回的访问连接
        String url = String.format(format, minioConfig.getEndpoint(), minioConfig.getBucketName(), file.getOriginalFilename());
        return url;
    }
}

```

### 5、测试

使用 Postman 调用接口进行测试
![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/86dded5562894081866c22f642e36602.png#clientId=ua3e9283e-00cb-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=454&id=ua1de8f61&originHeight=454&originWidth=1116&originalType=binary&ratio=1&rotation=0&showTitle=false&size=48866&status=done&style=none&taskId=uf36770fe-5ca8-4b41-9290-a7f1830b1e3&title=&width=1116#errorMessage=unknown%20error&id=Vk1N2&originHeight=454&originWidth=1116&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
之后访问返回的地址，便可以获取图片

## 问题

#### 1、io.minio.errors.InvalidResponseException: Non-XML response from server. Response code: 403, Content-Type: text/xml; charset=utf-8, body:

`AccessDenied` S3 API Request made to Console port. S3 Requests should be sent to API port. 0
原因：
![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/61db099ba84b65d4db39ff01618b19c6.png#clientId=u653688ab-3a73-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=169&id=ud1cfb037&originHeight=169&originWidth=449&originalType=binary&ratio=1&rotation=0&showTitle=false&size=14219&status=done&style=none&taskId=u2521b37c-ceda-49d6-823d-6fb78c2cff6&title=&width=449#errorMessage=unknown%20error&id=YryDY&originHeight=169&originWidth=449&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
端口用错了，应该使用的是 9000 的，而不是 9001,9001 是界面的端口，9000 才是 api 的端口

#### 2、The difference between the request time and the server's time is too large.

原因：linux 服务器时区的问题。
解决方式：
1、检查系统时间和硬件时间
![](https://images.weserv.nl/?url=https://img-blog.csdnimg.cn/img_convert/70df2f81e47f759400e43a0ce3df2657.png#clientId=ua3e9283e-00cb-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=68&id=u3526a0a6&originHeight=68&originWidth=410&originalType=binary&ratio=1&rotation=0&showTitle=false&size=5102&status=done&style=none&taskId=ude03701d-d116-47d5-bcda-45514df1324&title=&width=410#errorMessage=unknown%20error&id=cgSD2&originHeight=68&originWidth=410&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
2、如果不同的话，则需要同步
a.安装` yum -y install ntp ntpdate`
b.设置系统时间与网络时间同步
`ntpdate cn.pool.ntp.org`
c.将系统时间写入硬件时间`hwclock --systohc`

#
