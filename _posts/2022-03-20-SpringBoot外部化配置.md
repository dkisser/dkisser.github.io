# 前言
在前面的《SpringBoot可执行jar原理》里面我提到过SpringBoot的几个核心功能，而今天的主角外部化配置(Externalized Configuration)正是其中之一。
> Spring Boot lets you externalize your configuration so that you can work with the same application code in different environments. You can use a variety of external configuration sources, include Java properties files, YAML files, environment variables, and command-line arguments.

Spring官方给的解释是“外部化”你的配置文件，能让同一套的程序在不同的环境下工作。官方还给出properties、YAML、环境变量、还有command-line参数这些例子。
首先，这个外部化的配置肯定不是指我们自定义配置文件，而是指SpringBoot的系统配置文件。也就是我们常说的application.properties（或spring.yaml、spring.yml）。但是，“外部化”究竟是什么意思？SpringFramework和SpringBoot的配置到底有何差异？
# 差别对别
## 文本内容
### SpringFramework
```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:websocket="http://www.springframework.org/schema/websocket"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
                           http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd
                           http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd
                           http://www.springframework.org/schema/websocket http://www.springframework.org/schema/websocket/spring-websocket-4.0.xsd">
  <!-- <mvc:annotation-driven /> 会自动注册DefaultAnnotationHandlerMapping与AnnotationMethodHandlerAdapter 两个bean,是spring MVC为@Controllers分发请求所必须的。它提供了数据绑定支持，读取json的支持 -->
  <mvc:annotation-driven />

  <!-- 配置jsp视图解析器 -->
  <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver" id="jspViewResolver">
    <property name="viewClass" value="org.springframework.web.servlet.view.JstlView"/>
    <property name="prefix" value=""/>
    <property name="suffix" value=".jsp"/>
  </bean>
</beans>
```
这里我们看到了一些熟悉的配置，注解驱动、Componenet扫描、jsp相关配置等等。这些是我们在使用SpringFramework时经常看到的配置，可是在SpringBoot中我们似乎看不到了（忽略掉配置的内容，这里只是举个例子,想告诉你springFramework是通过这种方式来配置一个bean）。
### SpringBoot
在SpringFramework中我们配置一个bean时一定是通过XML的方式，而在SpringBoot中我们直接通过properties来配置.
```properties
management.endpoints.web.exposure.include=*
server.prot=8080
spring.project.name=demo
```
当然，我这里举的例子并不能明显的反映SpringFramework和SpringBoot在配置上的差别，只能说明两者的文本形式有些变化，SpringFramework采用xml而SpringBoot采用properties（或yaml）。
## 文件加载位置
### SpringFramework

- 必须在classPath下，且必须在pom中显示指定SpringFramework的配置文件地址
### SprinBoot

- 在classpath，或classpath下的config目录中的application.properties(yaml或yml)都能自动加载
- 非classpath,在当前目录或当前目录的config目录下的application.properties(yaml或yml)都能自动加载
- 非classpath,通过--spring.config.location在启动时显示指定
## 功能丰富度
SpringBoot在原有的SpringFramework的Environment框架中进行了扩展，支持了多配置的import、配置隔离、“外部化”配置加载。
### 多配置import
> Application properties may import further config data from other locations using the **spring.config.import** property. Imports are processed as they are discovered, and are treated as additional documents inserted immediately below the one that declares the import.

这里可以直接参考官方示例，[多配置import](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.files.importing).
### 配置隔离
> It is sometimes useful to only activate a given set of properties when certain conditions are met. For example, you might have properties that are only relevant when a specific profile is active.

这里可以直接参考官方示例，[配置隔离](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.files.activation-properties)。这里官方只是说可以只让部分配置生效，但是在实际生产中我们可以充分利用这个配置来达到配置隔离的目的。
### “外部化”配置
> Spring Boot will automatically find and load application.properties and application.yaml files from the following locations when your application starts:
>> 1. From the classpath
>>> a. The classpath root
>>> b. The classpath /config package
>> 2. From the current directory
>>> a. The current directory
>>> b. The /config subdirectory in the current directory
>>> c. Immediate child directories of the /config subdirectory

这也是我认为和SpringFramework差别较多的地方，具体的代码可以参考org.springframework.boot.context.config.ConfigFileApplicationListener，这个类中有整个application文件的加载过程。
# 踩坑日记

- 使用spring.profile.import导入配置后无法覆盖其中的部分内容

**错误描述**：application-prod.properties中配置了RPC调用的版本号为1.0.0。在application-prod-sg.properties和application-prod-de.properties中使用spring.profile.import导入了prod配置，启动时根据不同的环境选择不同的profile，例如新加坡的启动命令时--Dspring.profile.active=prod-sg,德国的启动命令是--Dspring.profile.active=prod-de。之所以这样做是因为新加坡的服务提供者的版本号是1.0.0-SG，德国的服务提供者的版本是1.0.0。我将prod-sg的rpc版本号修改成1.0.0-sg，德国的不变，修改之后发布，然后发现新加坡rpc的调用版本还是1.0.0。
**错误原因**：使用的springboot版本是1.5.22，这个版本的实现中是先解析application然后解析application-prod-sg，然后再解析prod，按照这个顺序把配置加载到Environment中。因此，prod中的配置最终覆盖了prod-sg的配置
**解决办法**：在prod中删除相关配置，在prod-sg和prod-de中按照rpc调用版本来单独配置。
说明：在springboot-2.x版本里面已经修复，具体代码可以参考org.springframework.boot.context.config.ConfigFileApplicationListener.Loader#load()
# 参考文献
[《SpringBoot官方文档》--spring社区](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config)
[《springmvc配置文件》--小白coder](https://www.cnblogs.com/alice-cj/p/10424049.html)
