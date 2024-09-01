# 步骤
### 1、异常类
```java
package com.walker.dianping.common.exceptions;

import lombok.Data;

@Data
//继承RuntimeException
public class BizException extends RuntimeException {

    private String msg;

    public BizException(String msg) {
        super(msg);  //此处记得实例化
        this.msg = msg;
    }
}

```
### 2、统一异常处理配置类
```java
package com.walker.dianping.common.config;

import com.walker.dianping.common.exceptions.BizException;
import com.walker.dianping.model.R;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
@Slf4j
public class MyExceptionHandler {

    @ExceptionHandler(value = BizException.class)
    public R BizExceptionHandler(Exception e) {
        log.error("错误原因是：" + e.getMessage());
        return R.fail("系统异常，原因如下："+e.getMessage());
    }

    @ExceptionHandler(value = Exception.class)
    public R exceptionHandler(Exception e) {
        log.error("错误原因是：" + e.getMessage());
        return R.fail(e.getMessage());
    }


}

```
### 3、断言类
可以自己进行补充
```java
package com.walker.dianping.common.utils;

import com.walker.dianping.common.exceptions.BizException;
import org.apache.commons.lang3.StringUtils;

public abstract class Assert {

    //如果字符串为空的话，就抛出异常
    public static void isBlank(String str, String message) {
        if (StringUtils.isBlank(str)) {
            throw new BizException(message);
        }
    }

    public static void isNull(Object object, String message) {
        if (object == null) {
            throw new BizException(message);
        }
    }
}

```
### 4、使用
```java
//查询类，如果类不存在则抛出异常
TbSeckillVoucherEntity entity = getById(voucherId);
 Assert.isNull(entity,"该"+voucherId+"优惠券不存在");
```
可以让代码变得相对简洁
