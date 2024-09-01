分类：

- Java Web 中提供的 Filter
- SpringMvc 中提供的拦截器 Interceptor
- Spring 提供的 AOP 技术+自定义注解

下边结合 jwt、redis 实现简单的实现
这里 redis 整合的具体教程就不详细讲了，具体可以查看该文章
[【redis 系列】springboot 整合 redis](https://blog.csdn.net/Think_and_work/article/details/123165932?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522167532901816800180646138%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=167532901816800180646138&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-1-123165932-null-null.blog_rank_default&utm_term=redis&spm=1018.2226.3001.4450)
首先先导入依赖

```xml
        <dependency>
            <groupId>io.jsonwebtoken</groupId>

            <artifactId>jjwt</artifactId>

            <version>0.9.1</version>

        </dependency>

```

### filter 方式

这种方式目前是不大推荐的，因为它需要手动设置需要进行拦截的接口
![image](https://img-blog.csdnimg.cn/img_convert/d27a1cde379d212befc54cfb7c506d7f.png#averageHue=#312d2c&clientId=u2771c5eb-b1d5-4&from=paste&height=250&id=u916148d9&name=image.png&originHeight=312&originWidth=907&originalType=binary&ratio=1&rotation=0&showTitle=false&size=41114&status=done&style=none&taskId=uf530d3cf-2c56-41e1-a528-85197c9d83d&title=&width=725.6#errorMessage=unknown%20error&id=gOhBk&originHeight=312&originWidth=907&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

#### 1、jwt 参数类、配置

这个的目的是为了将参数封装在类里面，不需要每次都是使用@Value 去获取。

```java
package com.walker.dianping.common.properties;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Data
@Component
@ConfigurationProperties(prefix = "jwt")
public class JWTProperties {
    private String secret;
    private long expire;
    private String header;
    private String whiteList;
}

```

**application.yml**

```yaml
jwt:
  # 加密密钥
  secret: safr3414fffdw1
  # token有效时长 单位秒
  expire: 3600
  # header 名称
  header: Authorization
  # 白名单
  whiteList: /login
```

<!-- more -->

#### 2、编写过滤器 TokenFilter

首先需要实现 Filter，然后去重写他的方法
过滤方法的逻辑大致如下：
![](https://img-blog.csdnimg.cn/img_convert/129b89d1d3e3d3b750418826dbf525c6.png#averageHue=#f9f8f8&clientId=u4c28bc56-6486-4&from=paste&height=518&id=u2382529d&name=image.png&originHeight=647&originWidth=367&originalType=binary&ratio=1&rotation=0&showTitle=false&size=52283&status=done&style=none&taskId=uba841e94-bf4e-43fd-ae45-211c8195e50&title=&width=293.6#errorMessage=unknown%20error&id=r7YD1&originHeight=647&originWidth=367&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

```java
package com.walker.dianping.common.config.interceptor;

import cn.hutool.core.util.StrUtil;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.walker.dianping.common.constants.SystemConstant;
import com.walker.dianping.common.properties.JWTProperties;
import com.walker.dianping.common.utils.Assert;
import com.walker.dianping.entity.R;
import com.walker.dianping.entity.TbUserEntity;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.concurrent.TimeUnit;


@Slf4j
//注册到容器中
@Component
//1、实现Filter，然后重写init，doFilter，destroy方法
public class TokenFilter implements Filter {

    @Autowired
    private JWTProperties jwtProperties;

    //redisTemplate redis的工具类
    @Autowired
    private StringRedisTemplate redisTemplate;


    String WHILE_LIST="/login";
    String TOKEN_USER_KEY="token:user:";


    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        Filter.super.init(filterConfig);
    }



    //过滤方法
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        //先将其转成HttpServletRequest
        HttpServletRequest request=(HttpServletRequest) servletRequest;

        String requestURI = request.getRequestURI();
        //如果不在白名单中，则检测token是否正常
        if(!WHILE_LIST.contains(requestURI)){

            //获取请求头的参数
            String token = request.getHeader(jwtProperties.getHeader());
            if(StrUtil.isEmpty(token)){
                resp(servletResponse,"账号未登录");
                return;
            }

            //从redis中获取token是否存在，是否过期
            String json = redisTemplate.opsForValue().get(TOKEN_USER_KEY + token);
            if(StrUtil.isEmpty(json)){
                resp(servletResponse,"账号未登录");
                return;
            }

        }

        filterChain.doFilter(servletRequest, servletResponse);
    }



    @Override
    public void destroy() {
        Filter.super.destroy();
    }


    //相应方法封装
        private void resp(ServletResponse servletResponse,String msg) throws IOException {
        HttpServletResponse response=(HttpServletResponse) servletResponse;
        ServletOutputStream outputStream = response.getOutputStream();
        response.setStatus(SystemConstant.CODE_401);
        response.setContentType(SystemConstant.JSON);
        R<Object> r = R.fail(msg);
        outputStream.write(JSON.toJSONString(r).getBytes(StandardCharsets.UTF_8));
    }
}

```

#### 3、WebMvcConfig 实现 WebMvcConfigurer

```java
package com.walker.dianping.common.config;

import com.walker.dianping.common.config.interceptor.TokenFilter;
import com.walker.dianping.common.properties.JWTProperties;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import javax.servlet.Filter;

@Configuration
    //实现WebMvcConfigurer
public class WebMvcConfig implements WebMvcConfigurer {

    @Autowired
    private TokenFilter tokenFilter;


    /**
    * 登录过滤器
    * 如果这个没有配置的话，默认所有的请求都会走filter
    */
    @Bean
    public FilterRegistrationBean<Filter> loginFilterRegistration(){
        FilterRegistrationBean<Filter> registrationBean = new FilterRegistrationBean<>();
        //设置过滤器
        registrationBean.setFilter(tokenFilter);
        registrationBean.setName("loginFilter");
        //拦截路径,这个就不大好，每次新增接口都得添加新的拦截器
        registrationBean.addUrlPatterns("/test/get");
        //指定顺序，数字越小越靠前
        registrationBean.setOrder(-1);
        return registrationBean;
    }



}

```

#### 4、编写测试接口

```java
package com.walker.dianping.controller;

import com.walker.dianping.entity.R;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/test")
public class TestController {

    @GetMapping("/get")
    public R get(){
        return R.ok("hello");
    }
}

```

- 请求头未带 token 时

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-xFP0pqwq-1675924860207)([https://cdn.nlark.com/yuque/0/2023/png/21370279/1675331842944-17baa622-b569-4b4c-8a98-0775808033ae.png#averageHue=%23fcfbfb&clientId=u2771c5eb-b1d5-4&from=paste&height=535&id=uf2bdddf8&name=image.png&originHeight=669&originWidth=1457&originalType=binary&ratio=1&rotation=0&showTitle=false&size=59731&status=done&style=none&taskId=u888d20f1-82df-4dac-a351-1755efd1f0b&title=&width=1165.6)]](https://cdn.nlark.com/yuque/0/2023/png/21370279/1675331842944-17baa622-b569-4b4c-8a98-0775808033ae.png#averageHue=%23fcfbfb&clientId=u2771c5eb-b1d5-4&from=paste&height=535&id=uf2bdddf8&name=image.png&originHeight=669&originWidth=1457&originalType=binary&ratio=1&rotation=0&showTitle=false&size=59731&status=done&style=none&taskId=u888d20f1-82df-4dac-a351-1755efd1f0b&title=&width=1165.6)])

- 请求头带 token 时

![](https://img-blog.csdnimg.cn/img_convert/cbc43c2b0a60f6855c8842e8826d021a.png#averageHue=#fcfcfb&clientId=u2771c5eb-b1d5-4&from=paste&height=565&id=u18324933&name=image.png&originHeight=706&originWidth=1454&originalType=binary&ratio=1&rotation=0&showTitle=false&size=70018&status=done&style=none&taskId=u51a52655-d95e-4611-990c-ecaf4b44157&title=&width=1163.2#errorMessage=unknown%20error&id=qlZgp&originHeight=706&originWidth=1454&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

### HandlerInterceptor 拦截器

这个接口是 springboot 自带的，实现起来方便了不少

#### 1、编写 token 拦截器

```java
package com.walker.dianping.common.config.interceptor;

import cn.hutool.core.util.StrUtil;
import com.alibaba.fastjson.JSON;
import com.walker.dianping.common.constants.SystemConstant;
import com.walker.dianping.common.properties.JWTProperties;
import com.walker.dianping.entity.R;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import javax.servlet.ServletOutputStream;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.nio.charset.StandardCharsets;

/**
* HandlerInterceptor 拦截器
*/
@Component

//实现HandlerInterceptor接口
public class TokenInterceptor implements HandlerInterceptor {


    @Autowired
    private StringRedisTemplate redisTemplate;

    @Autowired
    private JWTProperties jwtProperties;
    String TOKEN_USER_KEY = "token:user:";


    //拦截方法
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //从请求头中获取方法token
        String token = request.getHeader(jwtProperties.getHeader());
        if(StrUtil.isEmpty(token)){
            resp(response,"用户未登录");
            return false;
        }

        String userJson = redisTemplate.opsForValue().get(TOKEN_USER_KEY + token);
        if(StrUtil.isEmpty(userJson)){
            resp(response,"用户未登录/登录已过期");
            return false;
        }
        return true;
    }

    private void resp(ServletResponse servletResponse, String msg) throws IOException {
        HttpServletResponse response = (HttpServletResponse) servletResponse;
        ServletOutputStream outputStream = response.getOutputStream();
        response.setStatus(SystemConstant.CODE_401);
        response.setContentType(SystemConstant.JSON);
        R<Object> r = R.fail(msg);
        outputStream.write(JSON.toJSONString(r).getBytes(StandardCharsets.UTF_8));
    }
}

```

#### 2、编写 webMvcConfig

```java
package com.walker.dianping.common.config.mvc;

import com.walker.dianping.common.config.interceptor.TokenInterceptor;
import com.walker.dianping.common.properties.JWTProperties;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebMvcConfig implements WebMvcConfigurer {


    //注入token拦截器
    @Autowired
    private TokenInterceptor tokenInterceptor;
    @Autowired
    private JWTProperties jwtProperties;

    /**
    * 重写addInterceptors(InterceptorRegistry registry)方法
    */
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(tokenInterceptor)
                //不需要进行拦截的，也就是白名单
                .excludePathPatterns("/user/login")
                .addPathPatterns("/**");
    }


}

```

3、测试
现在是将"/user/login"作为白名单，不进行拦截，所以下面做两个测试的方式

- user/login 接口

![](https://img-blog.csdnimg.cn/img_convert/e91df2fcefef93758e56457038b7af1e.png#averageHue=#fcfcfb&clientId=u6b82fa0c-fb1d-4&from=paste&height=616&id=u5abc6033&name=image.png&originHeight=770&originWidth=1463&originalType=binary&ratio=1&rotation=0&showTitle=false&size=84503&status=done&style=none&taskId=uec554a58-ea9e-413d-9b14-5e9f5a0a782&title=&width=1170.4#errorMessage=unknown%20error&id=yKD8Y&originHeight=770&originWidth=1463&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
返回结果是 200，是 ok 的，因为不需要被拦截

- test/get 接口，不带 token 时

![](https://img-blog.csdnimg.cn/img_convert/6031bc499814b49ae9a60501ae1d79dc.png#averageHue=#fcfcfc&clientId=u6b82fa0c-fb1d-4&from=paste&height=545&id=u39f9ef17&name=image.png&originHeight=681&originWidth=1402&originalType=binary&ratio=1&rotation=0&showTitle=false&size=67181&status=done&style=none&taskId=u9bfb4b2b-1710-4655-b944-4341a5c25d5&title=&width=1121.6#errorMessage=unknown%20error&id=upLEU&originHeight=681&originWidth=1402&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
可以发现，走了拦截器的方法，因为其未带 token，所以返回 500 和未登录

- test/get 接口，带 token 时

![](https://img-blog.csdnimg.cn/img_convert/781889e632003b630f6c3fc246a430e0.png#averageHue=#fdfcfc&clientId=u6b82fa0c-fb1d-4&from=paste&height=569&id=ub36049d8&name=image.png&originHeight=711&originWidth=1385&originalType=binary&ratio=1&rotation=0&showTitle=false&size=66747&status=done&style=none&taskId=u14e8149d-eb80-4912-93d4-0b6334cdb98&title=&width=1108#errorMessage=unknown%20error&id=tc5wg&originHeight=711&originWidth=1385&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
可以发现返回 200

### aop+注解方式

这个是编写一个注解 IgnoreToken，然后如果接口中有带该方法的，则认为是白名单，不需要做 token 的校验

#### 1、导入依赖

```xml
     <dependency>
            <groupId>org.springframework.boot</groupId>

            <artifactId>spring-boot-starter-aop</artifactId>

        </dependency>

```

#### 2、注解类

定义该注解，如果是带有该注解的，则是白名单，不需要进行 token 的校验

```java
package com.walker.dianping.common.annotation;

import java.lang.annotation.*;

/**
* author:walker
* time: 2023/2/3
* description: 是否需要token
*/

//注解可以存在于方法，以及类中
@Target({ElementType.METHOD,ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface IgnoreToken {


}

```

#### 3、编写切面类

切面拦截方法大概流程如下：
![](https://img-blog.csdnimg.cn/img_convert/ffa7a7c69e26dd8e8699be1d5cf49a14.png#averageHue=#f9f9f8&clientId=u4c28bc56-6486-4&from=paste&height=244&id=u1b2cf3d8&name=image.png&originHeight=305&originWidth=915&originalType=binary&ratio=1&rotation=0&showTitle=false&size=35714&status=done&style=none&taskId=ueadc7ecb-9b48-41ad-a46a-eaf1a510b99&title=&width=732#errorMessage=unknown%20error&id=Ogx3R&originHeight=305&originWidth=915&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

```java
package com.walker.dianping.common.aspect;

import cn.hutool.core.util.StrUtil;
import com.alibaba.fastjson.JSON;
import com.walker.dianping.common.annotation.IgnoreToken;
import com.walker.dianping.common.constants.SystemConstant;
import com.walker.dianping.common.properties.JWTProperties;
import com.walker.dianping.entity.R;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.servlet.ServletOutputStream;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.lang.reflect.Method;
import java.nio.charset.StandardCharsets;

/**
 * author:walker
 * time: 2023/2/3
 * description:  token切面处理
 */
@Component
//声明切面
@Aspect
public class TokenAspect {

    @Autowired
    private JWTProperties jwtProperties;

    @Autowired
    private StringRedisTemplate redisTemplate;

    String TOKEN_USER_KEY = "token:user:";

    //切点
    //controller.*.*(..)  代表了controller包下的所有类.所有public方法(所有参数)
    //这里需要修改为自己的controller放置的地方
    @Pointcut(value = "execution(public * com.walker.dianping.controller.*.*(..))")
    public void allController() {

    }


    /**
    * @Around注解用于修饰Around增强处理，Around增强处理非常强大，表现在：
     * 1. @Around可以自由选择增强动作与目标方法的执行顺序，也就是说可以在增强动作前后，甚至过程中执行目标方法。这个特性的实现在于，调用ProceedingJoinPoint参数的procedd()方法才会执行目标方法。
     * 2. @Around可以改变执行目标方法的参数值，也可以改变执行目标方法之后的返回值。
    */
    @Around("allController()")
    public Object before(ProceedingJoinPoint joinPoint) throws Throwable {
        //获取方法签名和调用的方法
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        Method method = methodSignature.getMethod();


        //获取注解
        IgnoreToken annotation = method.getAnnotation(IgnoreToken.class);
        //如果有该注解的，则不需要进行token的校验
        if (annotation != null) {
            return joinPoint.proceed(joinPoint.getArgs());
        }
        //否则需要校验token的情况
        ServletRequestAttributes requestAttributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();

        //如果不是从前端过来的，没有ServletRequestAttributes，则直接放行？
        //但是这一步需要思考一下，我们的接口一般都提供给前端使用，所以也可以不做该判断处理
        if (requestAttributes == null || requestAttributes.getResponse() == null) {
            return joinPoint.proceed(joinPoint.getArgs());
        }

        HttpServletRequest request = requestAttributes.getRequest();
        HttpServletResponse response = requestAttributes.getResponse();
        //从请求头获取token，如果没有的话，则提示未登录
        String token = request.getHeader(jwtProperties.getHeader());
        if (StrUtil.isEmpty(token)) {
            resp(response,"用户未登录");
            return null;
        }

        //从redis中获取token
        String userJson = redisTemplate.opsForValue().get(TOKEN_USER_KEY + token);
        if(StrUtil.isEmpty(userJson)){
            resp(response,"用户未登录/登录已过期");
            return false;
        }

        return joinPoint.proceed(joinPoint.getArgs());

    }

        private void resp(ServletResponse servletResponse, String msg) throws IOException {
        HttpServletResponse response = (HttpServletResponse) servletResponse;
        ServletOutputStream outputStream = response.getOutputStream();
        response.setStatus(SystemConstant.CODE_401);
        response.setContentType(SystemConstant.JSON);
        R<Object> r = R.fail(msg);
        outputStream.write(JSON.toJSONString(r).getBytes(StandardCharsets.UTF_8));
    }

}

```

#### 4、编写测试接口

这里只做一个演示，具体的实现逻辑大家自己写，这里就不将代码列出来了

```java
package com.walker.dianping.controller;


import com.walker.dianping.common.annotation.IgnoreToken;
import com.walker.dianping.component.UserComponent;
import com.walker.dianping.entity.R;
import com.walker.dianping.entity.form.UserLoginForm;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import javax.validation.Valid;

/**
 * <p>
 *  前端控制器
 * </p>

 *
 * @author walker
 * @since 2023-01-30
 */
@RestController
@RequestMapping("/user")
public class TbUserController {

    @Autowired
    private UserComponent userComponent;


    //编写注解，代表不需要进行token的校验
    @IgnoreToken
    @PostMapping("/login")
    public R login(@RequestBody @Valid UserLoginForm form){
        return R.ok(userComponent.login(form));
    }

    @GetMapping("/test")
    public R test(){
        return R.ok("hello");
    }

    @PostMapping("/add")
    public R add(@RequestBody @Valid UserLoginForm form){
        userComponent.add(form);
        return R.ok();
    }


}

```

#### 5、测试

- 调用 user/test 接口，未带 token

![](https://img-blog.csdnimg.cn/img_convert/5d52f3d77635c204f15df694ebc7ac4e.png#averageHue=#fdfcfc&clientId=ub753deab-83c2-4&from=paste&height=565&id=ua9f56d1a&name=image.png&originHeight=706&originWidth=1433&originalType=binary&ratio=1&rotation=0&showTitle=false&size=61527&status=done&style=none&taskId=uad284332-a7a2-49a6-ac25-2686e09c43b&title=&width=1146.4#errorMessage=unknown%20error&id=JLQEx&originHeight=706&originWidth=1433&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

- /user/test 带 token

![](https://img-blog.csdnimg.cn/img_convert/b22f1c701d1a65dc77f1a45ae8ed5223.png#averageHue=#fcfbfb&clientId=ub753deab-83c2-4&from=paste&height=583&id=ua99013c7&name=image.png&originHeight=729&originWidth=1228&originalType=binary&ratio=1&rotation=0&showTitle=false&size=57970&status=done&style=none&taskId=ua3d7ce16-5ee1-428a-8ca7-4673d750458&title=&width=982.4#errorMessage=unknown%20error&id=f5I6u&originHeight=729&originWidth=1228&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

- /user/login 接口

发现是不需要进行拦截的
![](https://img-blog.csdnimg.cn/img_convert/e6c9446de029294603d9176759eb1194.png#averageHue=#fcfcfc&clientId=ub753deab-83c2-4&from=paste&height=609&id=u6d2d927f&name=image.png&originHeight=761&originWidth=1423&originalType=binary&ratio=1&rotation=0&showTitle=false&size=82339&status=done&style=none&taskId=ub328f2ab-7f6a-4e0a-9b26-5c91ba113a1&title=&width=1138.4#errorMessage=unknown%20error&id=wLtZX&originHeight=761&originWidth=1423&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)

### 参数解析器

#### 1、编写解析器

- 先实现 HandlerMethodArgumentResolver 接口
-

```java
package com.walker.dianping.common.config.resolver;

import cn.hutool.core.util.StrUtil;
import com.alibaba.fastjson.JSONObject;
import com.walker.dianping.common.annotation.LoginUser;
import com.walker.dianping.common.properties.JWTProperties;
import com.walker.dianping.entity.TbUserEntity;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.MethodParameter;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.support.WebDataBinderFactory;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.method.support.ModelAndViewContainer;

import javax.servlet.http.HttpServletRequest;

/**
 * author:walker
 * time: 2023/2/3
 * description:  参数解析器
 */

@Component
public class LoginUserResolver implements HandlerMethodArgumentResolver {

    @Autowired
    private JWTProperties jwtProperties;

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 什么是否进行拦截处理，也就是支持的条件，
     * 这里是当有LoginUser注解时
     */
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        //当有LoginUser的进行进行解析
        return parameter.hasParameterAnnotation(LoginUser.class);
    }


//重写解析参数方法
    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

        //获取请求属性
        ServletRequestAttributes requestAttributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (requestAttributes == null) {
            return null;
        }


        HttpServletRequest request = requestAttributes.getRequest();
        //从请求头中获取token
        String token = request.getHeader(jwtProperties.getHeader());
        if (StrUtil.isEmpty(token)) {
            return null;
        }

        //在redis中获取token
        String userJson = redisTemplate.opsForValue().get("token:user:" + token);


        if (StrUtil.isEmpty(userJson)) {
            return null;
        }

        //如果token中存储的用户数据存在，则返回
        return JSONObject.parseObject(userJson, TbUserEntity.class);
    }
}

```

#### 2、webMvcConfig 配置

```java
package com.walker.dianping.common.config.mvc;

import com.walker.dianping.common.config.resolver.LoginUserResolver;
import com.walker.dianping.common.properties.JWTProperties;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import java.util.List;

@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    /**
     *  参数解析器
     */
    @Autowired
    private LoginUserResolver loginUserResolver;



    /**
     * 参数解析器
     */
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(loginUserResolver);
    }
}

```

3、测试

- 测试接口

```java
    @GetMapping("/test")
    //使用@LoginUser注解和TbUserEntity去接收参数
    public R test(@LoginUser TbUserEntity tbUserEntity){
        return R.ok(JSON.toJSONString(tbUserEntity));
    }
```

- 使用 postman 带 token 发起请求

![](https://img-blog.csdnimg.cn/img_convert/c96a9ca7e33b8975dc76d364dc1fa9a2.png#averageHue=#fdfcfc&clientId=ub753deab-83c2-4&from=paste&height=614&id=ucb02471b&name=image.png&originHeight=767&originWidth=1417&originalType=binary&ratio=1&rotation=0&showTitle=false&size=76468&status=done&style=none&taskId=u083ec072-7407-429e-a4f7-64c07d0c1c2&title=&width=1133.6#errorMessage=unknown%20error&id=kqqjd&originHeight=767&originWidth=1417&originalType=binary&ratio=1&rotation=0&showTitle=false&status=error&style=none)
发现结果可以获取到解析后的参数

参考文档:
[Springboot 实现登录拦截的三种方式](https://blog.csdn.net/HLH_2021/article/details/119491890?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0-119491890-blog-126308356.pc_relevant_3mothn_strategy_and_data_recovery&spm=1001.2101.3001.4242.1&utm_relevant_index=3)

> 我是程序员 walker，一个持续学习，分享干货的博主
> 关注公众号【**I am Walker**】，一块进步
