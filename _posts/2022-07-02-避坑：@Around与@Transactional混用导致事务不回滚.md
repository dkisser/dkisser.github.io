# 前言
上个月，同事出于好奇在群里问AOP的环绕通知与事务注解混合用会不会导致出现异常不回滚的情况。这个问题我一下子回答不上来，因为平时没这样用过，在好奇心的驱使下，我调试了半天终于得到结果，今天我就展开讲讲。**（源码解读在最后面，感兴趣的可以看看。）**

# 结论
首先告诉大家的是，**同时使用AOP环绕通知和事务注解之后，最终生成的拦截器链的相对顺序是事务的拦截器在前面，AOP环绕通知的拦截器在后面。** 在事务的实现中将拦截器的执行过程包裹在了try-catch块中，发生异常后根据配置来决定是否回滚事务。（详见`org.springframework.transaction.interceptor.TransactionInterceptor#invoke`），因此事务后面的拦截器都会影响事务的执行结果。**如果在AOP环绕通知里面将拦截器链执行结果中的异常给吞掉，那么事务就会正常提交而不会回滚。**

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
启动项目后直接用GET请求访问本地`http://localhost:8080/home/echo` ，你会发现断点的执行顺序是 `org.springframework.transaction.interceptor.TransactionAspectSupport#invokeWithinTransaction line:388` -- `com.example.demo.aspect.CustomAspect#around line:24` -- `com.example.demo.aspect.CustomService#echo line:18` -- `com.example.demo.aspect.CustomAspect#around line:26` -- `org.springframework.transaction.interceptor.TransactionAspectSupport#invokeWithinTransaction line:407`

**现象：同时有环绕通知和事务时，在业务代码中抛出的异常会先被环绕通知处理，所以后面事务会正常提交而不会回滚。**

# 现实情况
现实中，我们很多人喜欢用环绕通知处理一些通用逻辑，那为什么没有出现bug呢？这里我列举下，环绕通知的常见使用场景如下。
* 打日志
* 流控

这些场景的共同特点是它们都是在入口层（也就是Web层）做的环绕通知，而我们通常不会把所有逻辑都写在入口层（如果你真的这么做了，我相信Code Review不会通过），而是有一个逻辑处理层（也就是service层）对入口层的数据进行专门处理。因此，我们的事务注解也大都标注在逻辑处理层的方法上。这样看来，确实很少有上面提到的场景。**（这里我们可以看到，遵守开发规范在一定程度上对开发效率是有提升的，它可以规避一些bug）**
# 源码解读
**说明：** 在上面我们介绍了拦截器链的顺序是事务在前，环绕通知在后。**这里解读源码的目的是为了搞清楚为什么事务的拦截器在前，环绕通知的拦截器在后。**

我们知道事务和环绕通知的最终实现都是通过 AOP，而 spring 默认的 AOP构造类就是 `org.springframework.aop.framework.CglibAopProxy`，通过`getProxy()` 完成构造，而`getCallbacks()`就是构造的关键，可以看到关键代码在`DynamicAdvisedInterceptor`中，在这个类中完成了拦截器链的构造。（代码就不展示AOP代码了，这里涉及到AOP的太多知识，有兴趣的可以联系我，我可以单独讲讲）。
```java

/**
 * General purpose AOP callback. Used when the target is dynamic or when the
 * proxy is not frozen.
 */
private static class DynamicAdvisedInterceptor implements MethodInterceptor, Serializable {

  private final AdvisedSupport advised;

  public DynamicAdvisedInterceptor(AdvisedSupport advised) {
    this.advised = advised;
  }

  @Override
  @Nullable
  public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
    Object oldProxy = null;
    boolean setProxyContext = false;
    Object target = null;
    TargetSource targetSource = this.advised.getTargetSource();
    try {
      if (this.advised.exposeProxy) {
        // Make invocation available if necessary.
        oldProxy = AopContext.setCurrentProxy(proxy);
        setProxyContext = true;
      }
      // Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
      target = targetSource.getTarget();
      Class<?> targetClass = (target != null ? target.getClass() : null);
      List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
      Object retVal;
      // Check whether we only have one InvokerInterceptor: that is,
      // no real advice, but just reflective invocation of the target.
      if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
        // We can skip creating a MethodInvocation: just invoke the target directly.
        // Note that the final invoker must be an InvokerInterceptor, so we know
        // it does nothing but a reflective operation on the target, and no hot
        // swapping or fancy proxying.
        Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
        retVal = methodProxy.invoke(target, argsToUse);
      }
      else {
        // We need to create a method invocation...
        retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
      }
      retVal = processReturnType(proxy, target, method, retVal);
      return retVal;
    }
    finally {
      if (target != null && !targetSource.isStatic()) {
        targetSource.releaseTarget(target);
      }
      if (setProxyContext) {
        // Restore old proxy.
        AopContext.setCurrentProxy(oldProxy);
      }
    }
  }

  @Override
  public boolean equals(@Nullable Object other) {
    return (this == other ||
      (other instanceof DynamicAdvisedInterceptor &&
        this.advised.equals(((DynamicAdvisedInterceptor) other).advised)));
  }

  /**
   * CGLIB uses this to drive proxy creation.
   */
  @Override
  public int hashCode() {
    return this.advised.hashCode();
  }
}

```
上面可以看到拦截器链是通过`this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);`来构造的，最终其实走到`org.springframework.aop.framework.DefaultAdvisorChainFactory#getInterceptorsAndDynamicInterceptionAdvice`，简化后的代码如下。
```java
public class DefaultAdvisorChainFactory implements AdvisorChainFactory, Serializable {

	@Override
	public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
			Advised config, Method method, @Nullable Class<?> targetClass) {

		// This is somewhat tricky... We have to process introductions first,
		// but we need to preserve order in the ultimate list.
		AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
		Advisor[] advisors = config.getAdvisors();
		List<Object> interceptorList = new ArrayList<>(advisors.length);
		Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
		Boolean hasIntroductions = null;

		for (Advisor advisor : advisors) {
			if (advisor instanceof PointcutAdvisor) {
				// Add it conditionally.
				PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
				if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
					MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
					boolean match;
					if (mm instanceof IntroductionAwareMethodMatcher) {
						if (hasIntroductions == null) {
							hasIntroductions = hasMatchingIntroductions(advisors, actualClass);
						}
						match = ((IntroductionAwareMethodMatcher) mm).matches(method, actualClass, hasIntroductions);
					}
					else {
						match = mm.matches(method, actualClass);
					}
					if (match) {
						MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
						if (mm.isRuntime()) {
							// Creating a new object instance in the getInterceptors() method
							// isn't a problem as we normally cache created chains.
							for (MethodInterceptor interceptor : interceptors) {
								interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
							}
						}
						else {
							interceptorList.addAll(Arrays.asList(interceptors));
						}
					}
				}
			}
			else {
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}

		return interceptorList;
	}

}
```
可以看到，最终还是通过Ioc中的`org.springframework.aop.Advisor`来得到最终的拦截器链。代码里面是遍历 Advisor，判断是否符合条件，把符合条件的拦截器放入最终结果。**因此 Advisor 的相对顺序和拦截器链的相对顺序是一致的。**

而在SpringBoot启动的时候，会通过spring.factories中配置的相对顺序来自动装配模块。Aop先于事务装载，在装载Aspectj相关模块时会将`org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator`注册到IOC容器中。当装配事务时，也向IOC容器中注入了`org.springframework.transaction.interceptor.BeanFactoryTransactionAttributeSourceAdvisor`。在通过Aop生成的Advisor时，会通过`org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator#findCandidateAdvisors`来找所有的 Advisor，而此时还在IOC刷新阶段，只有事务注册了Advisor，因此会先加载事务相关的Advisor。（详细代码在`org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors`有兴趣的可以自行查阅）。而到DI时再去生成拦截器链时，就会发现事务的拦截器永远在最前面

# 推荐读物
《Spring 技术内幕》 -- 计文柯
> 这本书我反复读了3遍以上。虽然书是12年出版的，基于Spring Framework 4.X 进行讲解，版本有些旧。但是，当你读完这本书再去看 Spring Framework 5.x 你会发现书上讲的spring核心思想在最新版本中并没有发生太多变化，只是有了些增强。在我们对Spring核心还不太了解的时候如果直接上手最新版本可能会有些复杂，因为有很多优化实现，这样容易让我们陷入细节太深不太能看到系统的全貌。
