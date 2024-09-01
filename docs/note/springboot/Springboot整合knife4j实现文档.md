## springboot 整合

文档:[https://doc.xiaominfo.com/](https://doc.xiaominfo.com/)

### 1、导入依赖

```xml
<!--引入Knife4j的官方start包,该指南选择Spring Boot版本<3.0,开发者需要注意-->
<dependency>
    <groupId>com.github.xiaoymin</groupId>

    <artifactId>knife4j-openapi2-spring-boot-starter</artifactId>

    <version>4.0.0</version>

</dependency>

```

### 2、编写配置类

```java
package com.walker.dianping.common.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2WebMvc;

@Configuration
@EnableSwagger2WebMvc
public class Knife4jConfiguration {

    @Bean(value = "dockerBean")
    public Docket dockerBean() {
        //指定使用Swagger2规范
        Docket docket=new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(new ApiInfoBuilder()
                //描述字段支持Markdown语法
                .description("walker点评文档")  // 简介
                .termsOfServiceUrl("https://doc.xiaominfo.com/") // 服务url
                .contact("walker@114.com") // 练习方式
                .version("1.0") // 版本
                .build())
                .groupName("用户服务") //分组名称
                .select()
                //这里指定Controller扫描包路径
                .apis(RequestHandlerSelectors.basePackage("com.walker.dianping.controller"))
                .paths(PathSelectors.any())
                .build();
        return docket;
    }
}

```

<!-- more -->

### 3、编写 controller 类

```java
@Api(tags = "首页模块") //写在控制类的标签
@RestController
public class IndexController {

    //requestParam参数 如果是body参数的话，需要在类里面进行编写 @ApiModel
    @ApiImplicitParam(name = "name",value = "姓名",required = true)
    //接口名称
    @ApiOperation(value = "向客人问好")
    @GetMapping("/sayHi")
    public ResponseEntity<String> sayHi(@RequestParam(value = "name")String name){
        return ResponseEntity.ok("Hi:"+name);
    }
}
```

### 4、启动项目后访问 [http://ip:port/doc.thml](http://ip:port/doc.thml)

例如： [http://localhost:8080/doc.html](http://localhost:8080/doc.html)
注意：如果有做名单拦截的话，需要将其设为白名单
