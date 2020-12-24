---
title: Java注解和自定义注解
date: 2020-12-24 14:51:21
tags: Java
---

注解在日常开发工作中使用的频率很高，不管是Java中的注解，还是Spring中的注解，在能熟练使用的时候还需要了解为什么。

## Java中的注解
Java Annotation是JDK5.0引入的一种注释机制。
Annotation是代码里的特殊标记，这些标记可以在 **编译**、**类加载**、**运行时** 被读取，并执行相应的处理。
通过使用Annotation，程序员可以在不改变原有逻辑的情况下，在源文件中嵌入一些补充信息。
Annotation可以像修饰符一样被使用，可以用于
> package
> class
> interface
> constructor
> method
> member variable (成员变量)
> parameter
> local variable (局部变量)

jdk 1.8之后，只要出现类型(包括类、接口、注解、枚举)的地方都可以使用注解了。
我们可以使用JDK以及其它框架提供的Annotation，也可以自定义Annotation。

## 元注解
元注解是java官方提供的,用于修饰其他注解的几个属性。
因为开放了自定义注解,所以所有的注解必须有章可循,他们的一些属性必须要被定义.比如:
* 这个注解用在什么地方?类上还是方法上还是字段上
* 这个注解的生命周期是什么
* 是保留在源码里供人阅读就好,还是会生成在class文件中,对程序产生实际的作用

这些都需要被提前定义好,因此就有了以下这几种类型
> @Target
> @Retation
> @Inherited
> @Documented

### @Target
用于描述注解的使用范围（即：被描述的注解可以用在什么地方）
他的取值范围JDK定义了枚举类 **ElementType** ,他的值共有以下几种:

1. CONSTRUCTOR:      用于描述构造器
2. FIELD:            用于描述域即类成员变量
3. LOCAL_VARIABLE:   用于描述局部变量
4. METHOD:           用于描述方法
5. PACKAGE:          用于描述包
6. PARAMETER:        用于描述参数
7. TYPE:             用于描述类、接口(包括注解类型) 或enum声明

在JDK1.8,新加了两种类型

1. TYPE_PARAMETER:表示这个 Annotation 可以用在 Type 的声明式前
2. TYPE_USE 表示这个 Annotation 可以用在所有使用 Type 的地方

### @Retation
表示需要在什么级别保存该注释信息，用于描述注解的生命周期（即：被描述的注解在什么范围内有效）
他的取值范围来自于枚举类 **RetentionPolicy** ,取值共以下几种:

> SOURCE:   在源文件中有效（即源文件保留）
> CLASS:    在class文件中有效（即class保留）
> RUNTIME:  在运行时有效（即运行时保留）

### @Inherited
@Inherited 元注解是一个标记注解，@Inherited阐述了某个被标注的类型是被继承的。
如果一个使用了@Inherited修饰的annotation类型被用于一个class，则这个annotation将被用于该class的子类。

### @Documented
@Documented用于描述其它类型的annotation应该被作为被标注的程序成员的公共API，因此可以被例如javadoc此类的工具文档化。
Documented是一个标记注解，没有成员。

## 自定义注解

实现一个请求的Controller需要图片文件的注解
```java
import java.lang.annotation.*;

@Inherited                          // 可继承
@Retention(RetentionPolicy.RUNTIME) // 运行时
@Target({ElementType.METHOD})       // 描述方法
public @interface ImageNeed {

    // 可以给注解加入参数

}
```

### 注解解析器
我们写好了一个注解后，应该怎么去使用他呢，我们目前想实现的功能是 在需要图片的controller处使用，所以可以用AOP

```java
package com.sailliao.config;


import com.sailliao.dto.Result;
import com.sailliao.annotation.ImageNeed;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.web.multipart.MultipartHttpServletRequest;
import org.springframework.web.multipart.MultipartResolver;
import org.springframework.web.multipart.support.StandardServletMultipartResolver;

import javax.servlet.http.HttpServletRequest;
import java.util.Locale;

@Aspect         // 切面
@Configuration  // 注解
public class FileAspect {

    @Pointcut("execution(* com.sailiao.controller.*(..))") // 在哪儿切, 我们在controller切
    public void executeService() {
    }

    @Around("@annotation(imageNeed)") // 切的是注解
    public Object before(ProceedingJoinPoint joinPoint, ImageNeed imageNeed) throws Throwable {

        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();

        String contentType = request.getContentType();

        if (null == contentType || !contentType.toLowerCase(Locale.ENGLISH).startsWith("multipart/")) {
            return new Result(201, "please upload file");
        }

        MultipartResolver resolver = new StandardServletMultipartResolver();
        MultipartHttpServletRequest fileRequest = resolver.resolveMultipart(request);

        MultipartFile file = fileRequest.getFile("file");

        if (file == null) {
            return new Result(202, "please upload file");
        }

        if (file.getSize() == 0) {
            return new Result(203, "file size is 0");
        }

        return joinPoint.proceed();
    }

}
```

### 使用
定义了注解和切面后，使用就很简单了
```java
@ImageNeed
@RequestMapping("get")
public Result get(MultipartFile file) {
    
}
```
这样，在请求这个接口的时候，如果没携带文件，就会被事先拦截下来，不会进入到这个方法了

## 总结
java 中注解使用不仅仅是 APO,也可以是拦截器之类的。例如接收的参数是实体类，可以在实体类的属性上添加自定义的注解来限制参数条件。
这样就不用自己来在controller里面来各种if判断了，极大的简化了开发量，也让代码更好看。
使用类似
```java
public class man{

    @Validate(need=true, type="String", length=25)
    private String name;

}
```

文章借鉴了：
[这是使用自定义注解实现接口参数校验一个链接](https://juejin.cn/post/6844903889821499400)