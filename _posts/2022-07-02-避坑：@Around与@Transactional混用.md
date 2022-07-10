# 前言
上个月，同事出于好奇在群里问AOP的环绕通知与事务注解混合用会不会导致异常不回滚。这个问题我一下子回答不上来，因为平时没这样用过，在好奇心的驱使下，我调试了半天终于得到结果，今天我就展开讲讲。**（源码解读在最后面，感兴趣的可以看看。）**

# 结论
首先告诉大家的是，**同时使用AOP环绕通知和事务注解之后，最终生成的拦截器链的相对顺序是事务相关的拦截器在前面，AOP环绕通知的拦截器在后面。** 在事务的实现中将拦截器的执行过程包裹在了try-catch块中，发生异常后根据配置来决定是否回滚事务。（详见org.springframework.transaction.interceptor.TransactionInterceptor#invoke），因此事务后面的拦截器都会影响事务的执行结果。**如果在AOP环绕通知里面将拦截器链后面执行结果中的异常给吞掉，那么事务就永远不会回滚，事务在这种情况下就会失效。**

# 示例
## 业务代码
业务代码中直接抛出异常，代码如下所示。
```java
package com.example.demo.aspect;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * @Author Paul
 * @Date 2022/7/3 15:52
 */
@Service
public class CustomService {

  @Transactional(rollbackFor = Exception.class)
  public void echo(){
    boolean s = true;
    if (s){
      throw new RuntimeException("test");
    }
    System.out.println("Hello------");
  }

}

```
## 环绕通知
环绕通知中捕捉异常并打印日志，代码如下所示。
```java
package com.example.demo.aspect;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

/**
 * @Author Paul
 * @Date 2022/7/3 15:49
 */
@Component
@Aspect
public class CustomAspect {

    @Pointcut("execution(* com.example.demo.aspect..*(..))")
    public void pointcut(){
    }

    @Around("pointcut()")
    public void around(ProceedingJoinPoint joinPoint){
        try {
            joinPoint.proceed();
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
    }

}
```

## 测试类
这里是用get请求来测试（本来应该用 unit test 来测试的，但是懒得写代码了，手动测试和 UT 的效果一样），代码如下所示。
```java
package com.example.demo.controller;

import com.example.demo.aspect.CustomService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Date;

@RestController
@RequestMapping("/home")
public class HomeController {

    private CustomService customService;

    @Autowired
    public void setCustomService(CustomService customService) {
        this.customService = customService;
    }

    @GetMapping("/echo")
    public String echo(){
        customService.echo();
        return new Date().toString();
    }
}

```
## 打断点
直接debug更加清晰，给出几个关键的位置，方便大家定位（记得下载spring源码再debug，不然会跟我给出的行数对不上）。
* org.springframework.transaction.interceptor.TransactionAspectSupport#invokeWithinTransaction line:388和407 （分别对应事务往后执行和最后提交事务的代码）
* com.example.demo.aspect.CustomAspect#around line:24和26 （自定义环绕通知的代码中的 `joinPoint.proceed()`，以及异常处理的地方）
* com.example.demo.aspect.CustomService#echo line:18 （业务代码中抛出异常的地方）

## 效果描述
启动项目后直接访问`/home/echo`，你会发现断点的执行顺序是 `org.springframework.transaction.interceptor.TransactionAspectSupport#invokeWithinTransaction line:388` -> `com.example.demo.aspect.CustomAspect#around line:24` -> `com.example.demo.aspect.CustomService#echo line:18` -> `com.example.demo.aspect.CustomAspect#around line:26` -> `org.springframework.transaction.interceptor.TransactionAspectSupport#invokeWithinTransaction line:407`

**同时有环绕通知和事务时，在业务代码中抛出的异常会先被环绕通知处理，所以后面事务不会发生回滚。**

# 源码解读


spring.factories中配置的auto-configuration顺序决定了AopAutoConfiguration要先于TransactionAutoConfiguration被加载。
这两者分别使用了@EnableAspectJAutoProxy和@EnableTransactionManagement。而这两个注解又分别注册了ReflectiveAspectJAdvisorFactory和BeanFactoryTransactionAttributeSourceAdvisor。
不同点是BeanFactoryTransactionAttributeSourceAdvisor注册成了spring-bean而ReflectiveAspectJAdvisorFactory只是作为AnnotationAwareAspectJAutoProxyCreator的一个成员变量。而AnnotationAwareAspectJAutoProxyCreator
是Aop主要用来装配advisor的类。而它的findCandidateAdvisors实现则是先从spring-bean中拿已经注册了的advisor(BeanFactoryTransactionAttributeSourceAdvisor)，再通过ReflectiveAspectJAdvisorFactory去拿advisor(InstantiationModelAwarePointcutAdvisor)
**强调：** spring的aop不管是使用jdk代理还是使用cglib最后都会走到org.springframework.aop.framework.ReflectiveMethodInvocation，而proceed()就是各个advice(MethodInterceptor)执行的地方。DefaultAdvisorChainFactory#getInterceptorsAndDynamicInterceptionAdvice()是通过advisor来加载各个拦截器的地方
因此这两个advisor是在IOC过程中就确定顺序的，而到方法调用时就会直接使用IOC阶段已经初始化好的advisor的顺序来
