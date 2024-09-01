# 前提
## 引入依赖
```xml
 //web
<dependency>
            <groupId>org.springframework.boot</groupId>

            <artifactId>spring-boot-starter-web</artifactId>

            <scope>provided</scope>

        </dependency>

//lombok
        <dependency>
            <groupId>org.projectlombok</groupId>

            <artifactId>lombok</artifactId>

            <optional>true</optional>

        </dependency>

```
# 1、定义注解
## 加密注解类Encrypt
```java
package com.walker.encrypt.anotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
* 加密注解
*/
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Encrypt {
}

```
## 解密注解类
```java
package com.walker.encrypt.anotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
* 解密注解
*/
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD,ElementType.PARAMETER})
public @interface Decrypt {
}

```
# 2、编写工具类
```java
package com.walker.encrypt.util;

import com.sun.media.sound.FFT;

import javax.crypto.BadPaddingException;
import javax.crypto.Cipher;
import javax.crypto.IllegalBlockSizeException;
import javax.crypto.NoSuchPaddingException;
import javax.crypto.spec.SecretKeySpec;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;
import java.util.Base64;

public class AESUtil {
    private final static String AES="AES";
    private static final String AES_ALGORITHM = "AES/ECB/PKCS5Padding";


    /**
    * 获取cipher
    */
    private static Cipher getCipher(byte[] key,int model) throws NoSuchPaddingException, NoSuchAlgorithmException, InvalidKeyException {
        SecretKeySpec aes = new SecretKeySpec(key, AES);
        Cipher cipher = Cipher.getInstance(AES_ALGORITHM);
        cipher.init(model,aes);
        return cipher;
    }

    /**
    * 加密
    */
    public static String encrypt(byte[] data,byte[] key) throws NoSuchPaddingException, NoSuchAlgorithmException, InvalidKeyException, IllegalBlockSizeException, BadPaddingException {
        Cipher cipher = getCipher(key, Cipher.ENCRYPT_MODE);
        return Base64.getEncoder().encodeToString(cipher.doFinal(data));
    }

    /**
    * 解密
    */
    public static byte[] decrypt(byte[] data,byte[] key) throws NoSuchPaddingException, NoSuchAlgorithmException, InvalidKeyException, IllegalBlockSizeException, BadPaddingException {
        Cipher cipher = getCipher(key, Cipher.DECRYPT_MODE);
        return cipher.doFinal(Base64.getDecoder().decode(data));
    }


}
```
# 3、配置密钥和密钥类
## 1、application.properties
```properties
#长度为16位
body.encrypt.key=iamwalkerencrypt
```
## 2、属性类
```java
package com.walker.encrypt.properties;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "body.encrypt")
@Data
public class EncryptProperties {
    private String key;
}
```
# 4、相关类
## 1、响应类
```java
package com.walker.encrypt.model;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@AllArgsConstructor
@NoArgsConstructor
public class R {
    private Integer code;
    private String msg;
    private Object data;

    public static R ok(String msg){
        return new R(200,msg,null);
    }

    public static R ok(String msg,Object data){
        return new R(200,msg,data);
    }

    public static R error(String msg){
        return new R(500,msg,null);
    }

    public static R error(String msg,Object data){
        return new R(500,msg,data);
    }

}

```
## 2、用户类 【主要用于测试】
# 5、配置类
## 解密请求参数类
```java
package com.walker.encrypt.config;

import com.walker.encrypt.anotation.Decrypt;
import com.walker.encrypt.properties.EncryptProperties;
import com.walker.encrypt.util.AESUtil;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.core.MethodParameter;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpInputMessage;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.servlet.mvc.method.annotation.RequestBodyAdviceAdapter;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.lang.reflect.Type;

//允许获取EncryptProperties配置类
@EnableConfigurationProperties(EncryptProperties.class)
//@ControllerAdvice是一个全局数据处理组件
@ControllerAdvice

/**
* 实现的不是RequestBodyAdvice，而是RequestBodyAdviceAdapter
 * RequestBodyAdviceAdapter继承RequestBodyAdvice
*/
public class DecryptRequest extends RequestBodyAdviceAdapter {

    //引入加密属性类 主要是为了获取配置文件中的密钥
    @Autowired
    private EncryptProperties encryptProperties;


    /**
    * 配置支持条件
     * 这里是只有方法或者参数有Decrypt注解的时候才生效
    */
    @Override
    public boolean supports(MethodParameter methodParameter, Type type, Class<? extends HttpMessageConverter<?>> aClass) {
        return methodParameter.hasMethodAnnotation(Decrypt.class)||methodParameter.hasParameterAnnotation(Decrypt.class);
    }


    /**
    * 使用aop，在读取请求参数的之前，对请求参数进行解密
    */
    @Override
    public HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage, MethodParameter parameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType) throws IOException {

        //获取密钥的字节
        byte[] keyByte = encryptProperties.getKey().getBytes();
        //读取请求参数成为字节body
        byte[] body = new byte[inputMessage.getBody().available()];
        inputMessage.getBody().read(body);


        try {
            //解密
            byte[] decrypt = AESUtil.decrypt(body, keyByte);
            //将解密的字节放入字节数组输入流中
            ByteArrayInputStream bais = new ByteArrayInputStream(decrypt);

            //返回HttpInputMessage
            return new HttpInputMessage() {
                @Override
                public InputStream getBody() throws IOException {
                    return bais;
                }

                @Override
                public HttpHeaders getHeaders() {
                    return inputMessage.getHeaders();
                }
            };
        } catch (Exception e) {
            e.printStackTrace();
        }

        //如果不需要进行处理的话，则直接调用父类的beforeBodyRead，相当于直接返回inputMessage，不做处理
        return super.beforeBodyRead(inputMessage, parameter, targetType, converterType);
    }
}
package com.walker.encrypt.config;

import com.walker.encrypt.anotation.Decrypt;
import com.walker.encrypt.properties.EncryptProperties;
import com.walker.encrypt.util.AESUtil;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.core.MethodParameter;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpInputMessage;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.servlet.mvc.method.annotation.RequestBodyAdviceAdapter;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.lang.reflect.Type;

@EnableConfigurationProperties(EncryptProperties.class)
@ControllerAdvice

/**
* 实现的不是RequestBodyAdvice，而是RequestBodyAdviceAdapter
 * RequestBodyAdviceAdapter继承RequestBodyAdvice
*/
public class DecryptRequest extends RequestBodyAdviceAdapter {

    //引入加密属性类 主要是为了获取配置文件中的密钥
    @Autowired
    private EncryptProperties encryptProperties;


    /**
    * 配置支持条件
     * 这里是只有方法或者参数有Decrypt注解的时候才生效
    */
    @Override
    public boolean supports(MethodParameter methodParameter, Type type, Class<? extends HttpMessageConverter<?>> aClass) {
        return methodParameter.hasMethodAnnotation(Decrypt.class)||methodParameter.hasParameterAnnotation(Decrypt.class);
    }


    /**
    * 使用aop，在读取请求参数的之前，对请求参数进行解密
    */
    @Override
    public HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage, MethodParameter parameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType) throws IOException {

        //获取密钥的字节
        byte[] keyByte = encryptProperties.getKey().getBytes();
        //读取请求参数成为字节body
        byte[] body = new byte[inputMessage.getBody().available()];
        inputMessage.getBody().read(body);


        try {
            //解密
            byte[] decrypt = AESUtil.decrypt(body, keyByte);
            //将解密的字节放入字节数组输入流中
            ByteArrayInputStream bais = new ByteArrayInputStream(decrypt);
            
            //返回HttpInputMessage
            return new HttpInputMessage() {
                @Override
                public InputStream getBody() throws IOException {
                    return bais;
                }

                @Override
                public HttpHeaders getHeaders() {
                    return inputMessage.getHeaders();
                }
            };
        } catch (Exception e) {
            e.printStackTrace();
        }

        //如果不需要进行处理的话，则直接调用父类的beforeBodyRead，相当于直接返回inputMessage，不做处理
        return super.beforeBodyRead(inputMessage, parameter, targetType, converterType);
    }
}

```
## 加密返回结果类
```java
package com.walker.encrypt.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.walker.encrypt.anotation.Encrypt;
import com.walker.encrypt.model.R;
import com.walker.encrypt.properties.EncryptProperties;
import com.walker.encrypt.util.AESUtil;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.core.MethodParameter;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.servlet.mvc.method.annotation.ResponseBodyAdvice;

//允许获取EncryptProperties配置类
@EnableConfigurationProperties(EncryptProperties.class)
//开启ControllerAdvice
@ControllerAdvice

//实现ResponseBodyAdvice
public class EncrptResponse implements ResponseBodyAdvice<R> {

    @Autowired
    private EncryptProperties encryptProperties;

    //用来Object对象的转换
    private ObjectMapper om=new ObjectMapper();


    /**
    * 支持什么时候加密
    */
    @Override
    public boolean supports(MethodParameter methodParameter, Class<? extends HttpMessageConverter<?>> aClass) {
        return methodParameter.hasMethodAnnotation(Encrypt.class);
    }


    /**
    * 数据响应进行加密
     */
    @Override
    public R beforeBodyWrite(R r, MethodParameter methodParameter, MediaType mediaType, Class<? extends HttpMessageConverter<?>> aClass, ServerHttpRequest serverHttpRequest, ServerHttpResponse serverHttpResponse) {
        //获取key的字节
        byte[] keybytes = encryptProperties.getKey().getBytes();

        //如果msg和data存在的话，则进行加密，最后进行返回
        try {
            if(r.getMsg()!=null){
                r.setMsg(AESUtil.encrypt(r.getMsg().getBytes(),keybytes));
            }
            if(r.getData()!=null){
                r.setData(AESUtil.encrypt(om.writeValueAsBytes(r.getData()),keybytes));
            }

        } catch (Exception e) {
            e.printStackTrace();
        }

        return r;
    }
}

```
# 6、编写controller
```java
package com.walker.encrypt.cotroller;

import com.walker.encrypt.anotation.Decrypt;
import com.walker.encrypt.anotation.Encrypt;
import com.walker.encrypt.model.R;
import com.walker.encrypt.model.User;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/encrypt")
public class EncryptController {


    @Encrypt
    @GetMapping("/encrypt")
    public R encryptTest(){
        User user = new User();
        user.setName("walker");
        user.setAge(18);

        return R.ok("请求成功",user);
    }



    /**
    * 将前面返回的参数拿来测试
     * QXnfSrsDbFPR7tBfiPEsqMs58zFa94H6lnv4LnrHcfE=
    */
    @PostMapping("/decrypt")
    public R decrypt(@RequestBody @Decrypt User user){
        return R.ok("ok",user);
    }
}

```
# 7、测试
## 1、返回结果加密
![](https://img-blog.csdnimg.cn/img_convert/9bce2ea8300a8c59012a324d6d4a7226.png#clientId=u94a3d566-e293-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=478&id=u597af687&originHeight=478&originWidth=959&originalType=binary&ratio=1&rotation=0&showTitle=false&size=45752&status=done&style=none&taskId=u5eb7ac5c-5a26-470c-95eb-b9cc38aa32f&title=&width=959#errorMessage=unknown%20error&id=sprnE&originHeight=478&originWidth=959&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
## 2、请求参数解密
![](https://img-blog.csdnimg.cn/img_convert/aa6af998ee63896ae7472dc4ddaf9bda.png#clientId=u94a3d566-e293-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=566&id=u968d86cc&originHeight=566&originWidth=1008&originalType=binary&ratio=1&rotation=0&showTitle=false&size=61409&status=done&style=none&taskId=u30964a50-3b45-4060-81b9-1a02dce1312&title=&width=1008#errorMessage=unknown%20error&id=aaHQQ&originHeight=566&originWidth=1008&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

# 原理
本质用的还是AOP的思想

参考文档：[https://mp.weixin.qq.com/s/xq9bmpLJw6aqttTPyq_omA](https://mp.weixin.qq.com/s/xq9bmpLJw6aqttTPyq_omA)
