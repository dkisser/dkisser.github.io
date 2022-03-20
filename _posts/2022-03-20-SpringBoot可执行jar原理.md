# 前言
随着SpringBoot的流行，越来越多的企业开始使用SpingBoot。我认为之所以这个框架越来越流行，社区越来越活跃，跟这个框架为开发者带来的便利性有关。它不仅改变了原来我们的编程习惯，简化了业务编程模型，而且还大大的降低了搭建一个应用程序的复杂度。
但随着对这个框架的使用的深入，我发现这个框架远非其表面那么简单，日常所用的SpringBoot的功能特性仅仅是其冰山一角，我不禁有了疑问。SpringBoot到底为我们带来了哪些便利？它与Spring Framework的差别是什么？由于篇幅有限，本文着重讲解SpringBoot的可执行jar，其他特性也会顺带聊一聊。
要了解一个框架，首先要知道这个框架的功能特性，以及框架设计的目的，正所谓“名不正则言不顺”。我第一时间想到的是它的官方文档，官方总结的SpringBoot的特性包括如下：
>    - Create stand-alone Spring applications
>    - Embed Tomcat, Jetty or Undertow directly (no need to deploy WAR files)
>    - Provide opinionated 'starter' dependencies to simplify your build configuration
>    - Automatically configure Spring and 3rd party libraries whenever possible
>    - Provide production-ready features such as metrics, health checks, and externalized configuration
>    - Absolutely no code generation and no requirement for XML configuration
>
> [https://spring.io/projects/spring-boot#overview](https://spring.io/projects/spring-boot#overview)

根据官方文档再结合个人经验总结后的SpringBoot的特性如下：

- 创建独立运行的Spring应用	--- 可执行jar、SpringApplication
- 嵌入式web容器	---SpringApplication中的一部分
- 提供固化的starter，简化构建配置	---starter特性
- 自动装配Spring和第三方的依赖	---自动装配
- 提供Production-Ready特性（如Metrics、健康检查、外部化配置）
- 完全的无中间代码生成，也不再需要XML配置	---注解驱动

从上面可以看到SpringBoot可执行jar正是其提供的六大核心功能之一。不知大家是否还记得，在使用SpringMVC构建应用时想要运行自己的应用，必须要先将应用代码打包成一个WAR，再借助一个独立部署的Web容器（Tomcat、Jetty、UnderTow等等）才能运行。可是，当使用SpringBoot时，既不需要将应用打包成WAR，也不需要独立部署的Web容器，反而是直接将应用打包成一个JAR，然后直接通过java -jar命令就能启动。大家是否和我一样有疑问，为什么直接打成JAR就能运行？下面我将揭开它的神秘面纱。**（可执行jar只是指jar的运行，本文不讨论嵌入式Web容器相关内容）**

# Java可执行jar规范
当使用SpringBoot可执行jar来启动应用时，会发现最终启动的命令正是通过java原生的工具在启动。接下来，去看看java的可执行jar的规范是如何定义的。
> The java command starts a Java application. It does this by starting the Java Runtime Environment (JRE), loading the specified class, and calling that class's main() method. The method must be declared _public_ and _static_, it must not return any value, and it must accept a String array as a parameter. The method declaration has the following form:
> > public static void main(String[] args)
>
> By default, the first argument that is not an option of the java command is the fully qualified name of the class to be called. **If the -jar option is specified, its argument is the name of the JAR file containing class and resource files for the application. The startup class must be indicated by the Main-Class manifest header in its source code.**
> [https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html)

从上面可以知道，**使用java -jar启动一个应用需要两个条件。一是需要一个main方法，二是需要在manifest header中指定Main-Class**（当然若是有依赖非JDK自带的类，需要指定classPath，但此处没有展开讲，详细资料可以通过刚刚的链接进去查找），也就是MANIFEST.MF中要指定Main-Class。
main方法作为java程序的入口想必大家都已烂熟于心，在此我也不过多描述，对于MANIFEST.MF想必大家也不陌生，毕竟现在大多数项目都是基于maven构建，这里我也不多讲。不过，这里我还是给大家一个“传送门”，有兴趣的同学可以进去看看MENIFAST.MF中的相关内容。
> [https://docs.oracle.com/javase/8/docs/technotes/guides/jar/jar.html#The_META-INF_directory](https://docs.oracle.com/javase/8/docs/technotes/guides/jar/jar.html#The_META-INF_directory)

# SpringBoot可执行jar
先看看官方对SpringBoot可执行jar的讲解。
> The Spring Boot Maven Plugin provides Spring Boot support in [Apache Maven](https://maven.org/). **It allows you to package executable jar or war archives, run Spring Boot applications, generate build information and start your Spring Boot application prior to running integration tests.**
> [https://docs.spring.io/spring-boot/docs/2.5.3/maven-plugin/reference/htmlsingle/#?.?](https://docs.spring.io/spring-boot/docs/2.5.3/maven-plugin/reference/htmlsingle/#?.?)

从这里可以知道，SpringBoot的可执行jar依赖spring-boot-maven-plugin来实现，而这个插件就是实现SpringBoot可执行jar的关键。但是，这个plugin里面内容很多，又如何定位到关键的代码呢？既然无法直接得到答案，不妨先看看这个plugin打包之后的目录结构，看看能否找到什么蛛丝马迹。

## 可执行jar内部结构
（下面展示的是随便找的一个可执行jar的目录结构，有兴趣的同学可以自己创建一个试试）
> demo-0.0.1-SNAPSHOT
> |___BOOT-INF
> |___classes
> |___lib
> ...
> |___META-INF
> |___maven
> |___MANIFEST.MF
> |___org
> |___springframework
> |___boot
> ...

从上可以发现应用代码全在BOOT-INF/classes下，应用依赖的jar全都在BOOT-INF/lib，META-INF下存放maven相关信息以及MANIFEST.MF。下面展示下MENIFEST.MF的具体内容。
> Manifest-Version: 1.0
> Spring-Boot-Classpath-Index: BOOT-INF/classpath.idx
> Implementation-Title: demo
> Implementation-Version: 0.0.1-SNAPSHOT
> Spring-Boot-Layers-Index: BOOT-INF/layers.idx
> **Start-Class: com.example.demo.DemoApplication**
> Spring-Boot-Classes: BOOT-INF/classes/
> Spring-Boot-Lib: BOOT-INF/lib/
> Build-Jdk-Spec: 1.8
> Spring-Boot-Version: 2.5.2
> Created-By: Maven Jar Plugin 3.2.0
> **Main-Class: org.springframework.boot.loader.JarLauncher**

此时，似乎有点眉目了。根据java可执行jar规范，只要找到JarLauncher然后查看他的main方法就好。但此时直接去项目中搜索就会发现根本没有这个类，线索似乎在这里中断了。
**这里教大家一个小技巧，当你想确定一个类所属的jar包时，可以把类放到maven仓库中搜索**。
经过一番搜索后发现，这个类其实包含在另外一个jar中，当使用plugin打包时，插件会自动装载这个jar到BOOT-INF/lib下。所以，只需要在项目中手动加入如下依赖即可（注意要声明provided否则后面去使用maven打包时会报错）。
```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-loader</artifactId>
  <scope>provided</scope>
</dependency>

```


## JarLauncher讲解
根据上面提到过Java可执行jar规范，下面的关注重点就要在JarLauncher.main()了，接下来就去看看这个神秘的main方法。
```java
public static void main(String[] args) throws Exception {
   new JarLauncher().launch(args);
}

//--------------------
protected void launch(String[] args) throws Exception {
    if (!isExploded()) {
        //注册URLStreamHandler
        JarFile.registerUrlProtocolHandler();
    }
    //创建ClassLoader
    ClassLoader classLoader = createClassLoader(getClassPathArchivesIterator());
    String jarMode = System.getProperty("jarmode");
    String launchClass = (jarMode != null && !jarMode.isEmpty()) ? JAR_MODE_LAUNCHER : getMainClass();
    //通过ClassLoader反射调用Start-Class
    launch(args, launchClass, classLoader);
}

```
main方法只是简单的调用了JarLauncher.launch()方法，所以接下来我会围绕着launch方法来展开讲。可以看到上图的整个代码很简洁，我总结为三大步，分别是**注册URLStreamHandler、创建ClassLoader、通过ClassLoader反射调用Start-Class**。

### 注册URLStreamHandler
```java

private static final String HANDLERS_PACKAGE = "org.springframework.boot.loader";
/**
 * Register a {@literal 'java.protocol.handler.pkgs'} property so that a
 * {@link URLStreamHandler} will be located to deal with jar URLs.
 */
public static void registerUrlProtocolHandler() {
   	//...
    String handlers = System.getProperty(PROTOCOL_HANDLER, "");
    System.setProperty(PROTOCOL_HANDLER,
                       ((handlers == null || handlers.isEmpty()) ? HANDLERS_PACKAGE : handlers + "|" + HANDLERS_PACKAGE));
    //...
}
```
可以看到，此处的核心代码很简单，大致逻辑就是从系统变量中获取“java.protocol.handler.pkgs”，然后将HANDLERS_PACKAGE追加到这个系统属性末尾。这里我想很多人会有疑问，这样处理有什么用吗？接下来就去看看这个属性在哪里会被用到，先去看看URLStreamHandler类的说明文档。
> The abstract class URLStreamHandler is the common superclass for all stream protocol handlers. A stream protocol handler knows how to make a connection for a particular protocol type, such as http or https.
> In most cases, an instance of a URLStreamHandler subclass is not created directly by an application. **Rather, the first time a protocol name is encountered when constructing a URL, the appropriate stream protocol handler is automatically loaded.**
> Since:JDK1.0
> See Also:**URL.URL(String, String, int, String)**
> Author:James Gosling

从上可以看到，文档中提到的“当通过protocol构造URL时，会自动加载URLStreamHandler”，也就是说，这个URLStreamHandler会在URL的构造函数中自动加载，用于处理protocol连接相关的功能。
```java
 public URL(String protocol, String host, int port, String file,
               URLStreamHandler handler) throws MalformedURLException {
        //...

        protocol = protocol.toLowerCase();
        this.protocol = protocol;
        //...
        Parts parts = new Parts(file);
        path = parts.getPath();
        query = parts.getQuery();

        if (query != null) {
            this.file = path + "?" + query;
        } else {
            this.file = path;
        }
        ref = parts.getRef();

        // Note: we don't do validation of the URL here. Too risky to change
        // right now, but worth considering for future reference. -br
        if (handler == null &&
            (handler = getURLStreamHandler(protocol)) == null) {
            throw new MalformedURLException("unknown protocol: " + protocol);
        }
        this.handler = handler;
    }

```
从上面的代码中我们可以看到，URL创建时会初始化URLStreamHandler，具体方法就是通过**getURLStreamHandler()**，这个方法的流程大致是从系统变量中“java.protocol.handler.pkgs”，然后去实例化这个handler(代码太长，这里不展开讲，有兴趣的同学可以自己点进去看看)

### 创建ClassLoader
看到这里，大家可能一头雾水。上面的URL与本文的主题有什么关系呢？先不急，我先介绍SpringBoot的ClassLoader创建的过程。具体代码如下
```java
protected ClassLoader createClassLoader(Iterator<Archive> archives) throws Exception {
    List<URL> urls = new ArrayList<>(50);
    while (archives.hasNext()) {
        urls.add(archives.next().getUrl());
    }
    return createClassLoader(urls.toArray(new URL[0]));
}

protected ClassLoader createClassLoader(URL[] urls) throws Exception {
    return new LaunchedURLClassLoader(isExploded(), getArchive(), urls, getClass().getClassLoader());
}
```
从上面代码可以看到，创建的ClassLoader是LaunchedURLClassLoader，而这个类的继承关系如下所示
> ClassLoader
> |___SecureClassLoader
> |___URLClassLoader
> |___LaunchedURLClassLoader

可以看到LaunchedURLClassLoader其实是继承了URLClassLoader，而且重写了findResources方法，如下所示
```java
@Override
public Enumeration<URL> findResources(String name) throws IOException {
    if (this.exploded) {
        return super.findResources(name);
    }
    Handler.setUseFastConnectionExceptions(true);
    try {
        return new UseFastConnectionExceptionsEnumeration(super.findResources(name));
    }
    finally {
        Handler.setUseFastConnectionExceptions(false);
    }
}


//URLClassLoader.findResources
public Enumeration<URL> findResources(final String name)
        throws IOException
    {
        final Enumeration<URL> e = ucp.findResources(name, true);

        return new Enumeration<URL>() {
           //...
        };
    }

//URLClassPath.findResource
public Enumeration<URL> findResources(final String var1, final boolean var2) {
        return new Enumeration<URL>() {
            private int index = 0;
            private int[] cache = URLClassPath.this.getLookupCache(var1);
            private URL url = null;

            private boolean next() {
                if (this.url != null) {
                    return true;
                } else {
                    do {
                        URLClassPath.Loader var1x;
                        if ((var1x = URLClassPath.this.getNextLoader(this.cache, this.index++)) == null) {
                            return false;
                        }

                        this.url = var1x.findResource(var1, var2);
                    } while(this.url == null);

                    return true;
                }
            }

           //...
        };
    }

//URLClassPath.Loader.findResource
URL findResource(String var1, boolean var2) {
            URL var3;
            try {
                var3 = new URL(this.base, ParseUtil.encodePath(var1, false));
            } catch (MalformedURLException var7) {
                throw new IllegalArgumentException("name");
            }

            try {
                if (var2) {
                    URLClassPath.check(var3);
                }

                URLConnection var4 = var3.openConnection();
                if (var4 instanceof HttpURLConnection) {
                    HttpURLConnection var5 = (HttpURLConnection)var4;
                    var5.setRequestMethod("HEAD");
                    if (var5.getResponseCode() >= 400) {
                        return null;
                    }
                } else {
                    var4.setUseCaches(false);
                    InputStream var8 = var4.getInputStream();
                    var8.close();
                }

                return var3;
            } catch (Exception var6) {
                return null;
            }
        }
```
从上面代码中可以看到，LaunchedURLClassLoader.findResources()其实是调用其父类URLClassLoader.findResources()，而URLClassLoader又借助URLClassPatrh.findResources来返回Enumeration<URL>。
使用过Enumeration的人都知道，通过而这个接口的next()方法会返回具体实例对象，所以直接去看next()方法。我们会发现，next()方法还是将具体加载过程委托给了URLClassPath.Loader.findResource()。再进去Loader中看看，从里面可以发现URLClassPath.Loader借助URL.openConnection()得到具体的URL，也就是说**资源的定位是通过URL.openConnection()。**
```java
//URL.openConnection
public URLConnection openConnection() throws java.io.IOException {
    return handler.openConnection(this);
}
```
到这里，突然豁然开朗了。原来SpringBoot资源的定位是自定义实现的URLStreamHandler。我们再到项目中搜索，果然发现有个类实现了URLStreamHandler，它是**org.springframework.boot.loader.jar.Handler**。资源的定位在这个类中的openConnection方法中实现了，当然这个类的功能远不止此，我不再过多描述，有兴趣的同学可以进去看看。

### 通过ClassLoader反射调用Start-Class
SpringBoot应用的启动需要依赖启动类，因此，这个Start-Class就是SpringBoot的启动类（除了要在main方法中调用SpringApplication.run()还需要在类上标注@SpringBootApplication）,通过反射调用启动类来启动SpringBoot的启动过程，如：初始化IOC、Web容器、自动装配等等。

# 总结
可以说SpringBoot可执行jar的原理来自于java中的可执行jar规范。
但是，由于**原生的java中的可执行jar并不能加载jar中包含的jar**。因此，**SpringBoot通过自定义的ClassLoader以及URLStreamHandler等机制扩展定制自己的可执行jar来加载jar中BOOT-INF/lib包下的所有jar**，最后再通过反射调用SpringBoot启动类完成了jar的执行过程**。**


