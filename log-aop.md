# AOP是什么

> AOP : 面向切面编程（Aspect Oriented Programming）
>
> Aspect是一种新的模块化机制，用来描述分散在对象、类或函数中的横切关注点（crosscutting concern）。从关注点中分离出横切关注点是面向切面的程序设计的核心概念。**分离关注点使解决特定领域问题的代码从业务逻辑中独立出来，业务逻辑的代码中不再含有针对特定领域问题代码的调用，业务逻辑同特定领域问题的关系通过切面来封装、维护**，这样原本分散在整个应用程序中的变动就可以很好地管理起来。

# 名词概念

Spring AOP 延用了 AspectJ 中的概念，使用了 AspectJ 提供的 jar 包中的注解。也就是Spring AOP里面的概念和术语，并不是Spring独有的，而是和AOP相关的。

| **术语**     | 概念                                                         |
| ------------ | ------------------------------------------------------------ |
| Aspect       | 切面是`Pointcut`和`Advice`的集合，一般单独作为一个类。`Pointcut`和`Advice`共同定义了关于切面的全部内容，它是**什么时候，在何时和何处**完成功能 |
| Joinpoint    | 这表示你的应用程序中可以插入AOP方面的一点。也可以说，这是应用程序中使用Spring AOP框架采取操作的实际位置。 |
| Advice       | 这是在方法执行之前或之后采取的实际操作。 这是在Spring AOP框架的程序执行期间调用的实际代码片段。 |
| Pointcut     | 这是一组一个或多个切入点，在切点应该执行`Advice`。 您可以使用表达式或模式指定切入点，后面示例会提到。 |
| Introduction | 引用允许我们向现有的类添加新的方法或者属性                   |
| Weaving      | 创建一个被增强对象的过程。这可以在编译时完成（例如使用AspectJ编译器），也可以在运行时完成。Spring和其他纯Java AOP框架一样，在运行时完成织入 |

还有一些注解，表示Advice的类型，或者说增强的时机

| **术语**        | **概念**                                             |
| --------------- | ---------------------------------------------------- |
| Before          | 在方法被调用之前执行增强                             |
| After           | 在方法被调用之后执行增强                             |
| After-returning | 在方法成功执行之后执行增强                           |
| After-throwing  | 在方法抛出指定异常后执行增强                         |
| Around          | 在方法调用的前后执行自定义的增强行为（最灵活的方式） |

# 使用方式

### 1. 开启`@AspectJ`注解配置方式

开启`@AspectJ`的注解配置方式，有两种方式

1. 在XML中配置：

   ```angelscript
   angelscript
   复制代码<aop:aspectj-autoproxy/>
   ```

2. 使用`@EnableAspectJAutoProxy`注解

   ```less
   less复制代码@Configuration
   @EnableAspectJAutoProxy
   public class Config {
   
   }
   ```

开启了上述配置之后，所有**在容器中**，**被`@AspectJ`注解的 bean** 都会被 Spring 当做是 AOP 配置类，称为一个 Aspect。

> NOTE：这里有个要注意的地方，@AspectJ 注解只能作用于Spring Bean 上面，所以你用 @Aspect 修饰的类要么是用 @Component注解修饰，要么是在 XML中配置过的。

```java
// 有效的AOP配置类
@Aspect
@Component
public class MyAspect {
 	//....   
}

// 如果没有在XML配置过，那这个就是无效的AOP配置类
@Aspect
public class MyAspect {
 	//....   
}
```

### 2. 配置 Pointcut (增强的切入点)

Pointcut 在大部分地方被翻译成切点，用于定义哪些方法需要被增强或者说需要被拦截。

在Spring 中，我们可以认为 Pointcut 是用来匹配Spring 容器中所有满足指定条件的bean的方法。

比如下面的写法，

```java
    // 指定的方法
    @Pointcut("execution(* testExecution(..))")
    public void anyTestMethod() {}
```

下面完整列举一下 Pointcut 的匹配方式：

1. **execution**：匹配方法签名

   这个最简单的方式就是上面的例子，`"execution(* testExecution(..))"`表示的是匹配名为`testExecution`的方法，`*`代表任意返回值，`(..)`表示零个或多个任意参数。

2. **within：**指定所在类或所在包下面的方法（Spring AOP 独有）

   ```java
   java复制代码    // service 层
       // ".." 代表包及其子包
       @Pointcut("within(ric.study.demo.aop.svc..*)")
       public void inSvcLayer() {}
   ```

3. **@annotation**：方法上具有特定的注解

   ```java
   java复制代码    // 指定注解
       @Pointcut("@annotation(ric.study.demo.aop.HaveAop)")
       public void withAnnotation() {}
   ```

4. bean(idOrNameOfBean)：匹配 bean 的名字（Spring AOP 独有）

   ```java
   java复制代码    // controller 层
       @Pointcut("bean(testController)")
       public void inControllerLayer() {}
   ```

> 上述是日常使用中常见的几种配置方式
>
> 有更细的匹配需求的，可以参考这篇文章：[www.baeldung.com/spring-aop-…](https://link.juejin.cn?target=https%3A%2F%2Fwww.baeldung.com%2Fspring-aop-pointcut-tutorial)

**关于 Pointcut 的配置，Spring 官方有这么一段建议：**

> When working with enterprise applications, you often want to refer to modules of the application and particular sets of operations from within several aspects. We recommend defining a "SystemArchitecture" aspect that captures common pointcut expressions for this purpose. A typical such aspect would look as follows:
> 使用企业应用程序时，您通常希望从多个方面引用应用程序的模块和特定的操作集。我们建议定义一个“系统体系结构”方面，以捕获为此目的的常见切入点表达式。典型的此类方面如下所示：
> 使用企业应用程序时，您通常希望从多个方面引用应用程序的模块和特定的操作集。我们建议定义一个“系统体系结构”方面，以捕获为此目的的常见切入点表达式。典型的此类方面如下所示：

意思就是，如果你是在开发企业级应用，Spring 建议你使用 `SystemArchitecture`这种切面配置方式，即将一些公共的PointCut 配置全部写在这个一个类里面维护。官网文档给的例子像下面这样（它文中使用 XML 配置的，所以没加@Component注解）

的，所以没加@Component注解）

```less
less复制代码package com.xyz.someapp;

import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;

@Aspect
public class SystemArchitecture {

  /**
   * A join point is in the web layer if the method is defined
   * in a type in the com.xyz.someapp.web package or any sub-package
   * under that.
   */
  @Pointcut("within(com.xyz.someapp.web..*)")
  public void inWebLayer() {}

  /**
   * A join point is in the service layer if the method is defined
   * in a type in the com.xyz.someapp.service package or any sub-package
   * under that.
   */
  @Pointcut("within(com.xyz.someapp.service..*)")
  public void inServiceLayer() {}

  /**
   * A join point is in the data access layer if the method is defined
   * in a type in the com.xyz.someapp.dao package or any sub-package
   * under that.
   */
  @Pointcut("within(com.xyz.someapp.dao..*)")
  public void inDataAccessLayer() {}

  /**
   * A business service is the execution of any method defined on a service
   * interface. This definition assumes that interfaces are placed in the
   * "service" package, and that implementation types are in sub-packages.
   * 
   * If you group service interfaces by functional area (for example, 
   * in packages com.xyz.someapp.abc.service and com.xyz.def.service) then
   * the pointcut expression "execution(* com.xyz.someapp..service.*.*(..))"
   * could be used instead.
   */
  @Pointcut("execution(* com.xyz.someapp.service.*.*(..))")
  public void businessService() {}
  
  /**
   * A data access operation is the execution of any method defined on a 
   * dao interface. This definition assumes that interfaces are placed in the
   * "dao" package, and that implementation types are in sub-packages.
   */
  @Pointcut("execution(* com.xyz.someapp.dao.*.*(..))")
  public void dataAccessOperation() {}

}
```

上面这个 SystemArchitecture 很好理解，该 Aspect 定义了一堆的 Pointcut，随后在任何需要 Pointcut 的地方都可以直接引用。

配置切点，代表着我们想让程序拦截哪一些方法，但程序需要怎么对拦截的方法进行增强，就是后面要介绍的配置 Advice。

的，所以没加@Component注解）

```less
less复制代码package com.xyz.someapp;

import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;

@Aspect
public class SystemArchitecture {

  /**
   * A join point is in the web layer if the method is defined
   * in a type in the com.xyz.someapp.web package or any sub-package
   * under that.
   */
  @Pointcut("within(com.xyz.someapp.web..*)")
  public void inWebLayer() {}

  /**
   * A join point is in the service layer if the method is defined
   * in a type in the com.xyz.someapp.service package or any sub-package
   * under that.
   */
  @Pointcut("within(com.xyz.someapp.service..*)")
  public void inServiceLayer() {}

  /**
   * A join point is in the data access layer if the method is defined
   * in a type in the com.xyz.someapp.dao package or any sub-package
   * under that.
   */
  @Pointcut("within(com.xyz.someapp.dao..*)")
  public void inDataAccessLayer() {}

  /**
   * A business service is the execution of any method defined on a service
   * interface. This definition assumes that interfaces are placed in the
   * "service" package, and that implementation types are in sub-packages.
   * 
   * If you group service interfaces by functional area (for example, 
   * in packages com.xyz.someapp.abc.service and com.xyz.def.service) then
   * the pointcut expression "execution(* com.xyz.someapp..service.*.*(..))"
   * could be used instead.
   */
  @Pointcut("execution(* com.xyz.someapp.service.*.*(..))")
  public void businessService() {}
  
  /**
   * A data access operation is the execution of any method defined on a 
   * dao interface. This definition assumes that interfaces are placed in the
   * "dao" package, and that implementation types are in sub-packages.
   */
  @Pointcut("execution(* com.xyz.someapp.dao.*.*(..))")
  public void dataAccessOperation() {}

}
```

上面这个 SystemArchitecture 很好理解，该 Aspect 定义了一堆的 Pointcut，随后在任何需要 Pointcut 的地方都可以直接引用。

配置切点，代表着我们想让程序拦截哪一些方法，但程序需要怎么对拦截的方法进行增强，就是后面要介绍的配置 Advice。

# 实践例子

## MEAVEN配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <artifactId>demo-log-aop</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>demo-log-aop</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>com.xkcoding</groupId>
        <artifactId>spring-boot-demo</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>

    <dependency>
      <groupId>com.google.guava</groupId>
      <artifactId>guava</artifactId>
    </dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-aop</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>

		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<optional>true</optional>
		</dependency>

		<dependency>
			<groupId>cn.hutool</groupId>
			<artifactId>hutool-all</artifactId>
		</dependency>

		<!-- 解析 UserAgent 信息 -->
		<dependency>
			<groupId>eu.bitwalker</groupId>
			<artifactId>UserAgentUtils</artifactId>
		</dependency>
	</dependencies>

    <build>
        <finalName>demo-log-aop</finalName>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>

```

## Logback配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
	<include resource="org/springframework/boot/logging/logback/defaults.xml"/>
	<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
		<filter class="ch.qos.logback.classic.filter.LevelFilter">
			<level>INFO</level>
		</filter>
		<encoder>
			<pattern>%date [%thread] %-5level [%logger{50}] %file:%line - %msg%n</pattern>
			<charset>UTF-8</charset>
		</encoder>
	</appender>

	<appender name="FILE_INFO" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<!--如果只是想要 Info 级别的日志，只是过滤 info 还是会输出 Error 日志，因为 Error 的级别高， 所以我们使用下面的策略，可以避免输出 Error 的日志-->
		<filter class="ch.qos.logback.classic.filter.LevelFilter">
			<!--过滤 Error-->
			<level>ERROR</level>
			<!--匹配到就禁止-->
			<onMatch>DENY</onMatch>
			<!--没有匹配到就允许-->
			<onMismatch>ACCEPT</onMismatch>
		</filter>
		<!--日志名称，如果没有File 属性，那么只会使用FileNamePattern的文件路径规则如果同时有<File>和<FileNamePattern>，那么当天日志是<File>，明天会自动把今天的日志改名为今天的日期。即，<File> 的日志都是当天的。-->
		<!--<File>logs/info.demo-log-aop.log</File>-->
		<!--滚动策略，按照时间滚动 TimeBasedRollingPolicy-->
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
			<!--文件路径,定义了日志的切分方式——把每一天的日志归档到一个文件中,以防止日志填满整个磁盘空间-->
			<FileNamePattern>logs/demo-log-aop/info.created_on_%d{yyyy-MM-dd}.part_%i.log</FileNamePattern>
			<!--只保留最近90天的日志-->
			<maxHistory>90</maxHistory>
			<!--用来指定日志文件的上限大小，那么到了这个值，就会删除旧的日志-->
			<!--<totalSizeCap>1GB</totalSizeCap>-->
			<timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
				<!-- maxFileSize:这是活动文件的大小，默认值是10MB,本篇设置为1KB，只是为了演示 -->
				<maxFileSize>2MB</maxFileSize>
			</timeBasedFileNamingAndTriggeringPolicy>
		</rollingPolicy>
		<!--<triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">-->
		<!--<maxFileSize>1KB</maxFileSize>-->
		<!--</triggeringPolicy>-->
		<encoder>
			<pattern>%date [%thread] %-5level [%logger{50}] %file:%line - %msg%n</pattern>
			<charset>UTF-8</charset> <!-- 此处设置字符集 -->
		</encoder>
	</appender>

	<appender name="FILE_ERROR" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<!--如果只是想要 Error 级别的日志，那么需要过滤一下，默认是 info 级别的，ThresholdFilter-->
		<filter class="ch.qos.logback.classic.filter.ThresholdFilter">
			<level>Error</level>
		</filter>
		<!--日志名称，如果没有File 属性，那么只会使用FileNamePattern的文件路径规则如果同时有<File>和<FileNamePattern>，那么当天日志是<File>，明天会自动把今天的日志改名为今天的日期。即，<File> 的日志都是当天的。-->
		<!--<File>logs/error.demo-log-aop.log</File>-->
		<!--滚动策略，按照时间滚动 TimeBasedRollingPolicy-->
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
			<!--文件路径,定义了日志的切分方式——把每一天的日志归档到一个文件中,以防止日志填满整个磁盘空间-->
			<FileNamePattern>logs/demo-log-aop/error.created_on_%d{yyyy-MM-dd}.part_%i.log</FileNamePattern>
			<!--只保留最近90天的日志-->
			<maxHistory>90</maxHistory>
			<timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
				<!-- maxFileSize:这是活动文件的大小，默认值是10MB,本篇设置为1KB，只是为了演示 -->
				<maxFileSize>2MB</maxFileSize>
			</timeBasedFileNamingAndTriggeringPolicy>
		</rollingPolicy>
		<encoder>
			<pattern>%date [%thread] %-5level [%logger{50}] %file:%line - %msg%n</pattern>
			<charset>UTF-8</charset> <!-- 此处设置字符集 -->
		</encoder>
	</appender>

	<root level="info">
		<appender-ref ref="CONSOLE"/>
		<appender-ref ref="FILE_INFO"/>
		<appender-ref ref="FILE_ERROR"/>
	</root>
</configuration>
```

## AOP切面配置类

```java
/**
 * <p>
 * 使用 aop 切面记录请求日志信息
 * </p>
 *
 * @author yangkai.shen
 * @author chen qi
 * @date Created in 2018-10-01 22:05
 */
@Aspect
@Component
@Slf4j
public class AopLog {
    /**
     * 切入点
     */
    @Pointcut("execution(public * com.xkcoding.log.aop.controller.*Controller.*(..))")
    public void log() {

    }

    /**
     * 环绕操作
     *
     * @param point 切入点
     * @return 原方法返回值
     * @throws Throwable 异常信息
     */
    @Around("log()")
    public Object aroundLog(ProceedingJoinPoint point) throws Throwable {

        // 开始打印请求日志
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = Objects.requireNonNull(attributes).getRequest();

        // 打印请求相关参数
        long startTime = System.currentTimeMillis();
        Object result = point.proceed();
        String header = request.getHeader("User-Agent");
        UserAgent userAgent = UserAgent.parseUserAgentString(header);

        final Log l = Log.builder()
            .threadId(Long.toString(Thread.currentThread().getId()))
            .threadName(Thread.currentThread().getName())
            .ip(getIp(request))
            .url(request.getRequestURL().toString())
            .classMethod(String.format("%s.%s", point.getSignature().getDeclaringTypeName(),
                point.getSignature().getName()))
            .httpMethod(request.getMethod())
            .requestParams(getNameAndValue(point))
            .result(result)
            .timeCost(System.currentTimeMillis() - startTime)
            .userAgent(header)
            .browser(userAgent.getBrowser().toString())
            .os(userAgent.getOperatingSystem().toString()).build();

        log.info("Request Log Info : {}", JSONUtil.toJsonStr(l));

        return result;
    }

    /**
     *  获取方法参数名和参数值
     * @param joinPoint
     * @return
     */
    private Map<String, Object> getNameAndValue(ProceedingJoinPoint joinPoint) {

        final Signature signature = joinPoint.getSignature();
        MethodSignature methodSignature = (MethodSignature) signature;
        final String[] names = methodSignature.getParameterNames();
        final Object[] args = joinPoint.getArgs();

        if (ArrayUtil.isEmpty(names) || ArrayUtil.isEmpty(args)) {
            return Collections.emptyMap();
        }
        if (names.length != args.length) {
            log.warn("{}方法参数名和参数值数量不一致", methodSignature.getName());
            return Collections.emptyMap();
        }
        Map<String, Object> map = Maps.newHashMap();
        for (int i = 0; i < names.length; i++) {
            map.put(names[i], args[i]);
        }
        return map;
    }

    private static final String UNKNOWN = "unknown";

    /**
     * 获取ip地址
     */
    public static String getIp(HttpServletRequest request) {
        String ip = request.getHeader("x-forwarded-for");
        if (ip == null || ip.length() == 0 || UNKNOWN.equalsIgnoreCase(ip)) {
            ip = request.getHeader("Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || UNKNOWN.equalsIgnoreCase(ip)) {
            ip = request.getHeader("WL-Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || UNKNOWN.equalsIgnoreCase(ip)) {
            ip = request.getRemoteAddr();
        }
        String comma = ",";
        String localhost = "127.0.0.1";
        if (ip.contains(comma)) {
            ip = ip.split(",")[0];
        }
        if (localhost.equals(ip)) {
            // 获取本机真正的ip地址
            try {
                ip = InetAddress.getLocalHost().getHostAddress();
            } catch (UnknownHostException e) {
                log.error(e.getMessage(), e);
            }
        }
        return ip;
    }

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    static class Log {
        // 线程id
        private String threadId;
        // 线程名称
        private String threadName;
        // ip
        private String ip;
        // url
        private String url;
        // http方法 GET POST PUT DELETE PATCH
        private String httpMethod;
        // 类方法
        private String classMethod;
        // 请求参数
        private Object requestParams;
        // 返回参数
        private Object result;
        // 接口耗时
        private Long timeCost;
        // 操作系统
        private String os;
        // 浏览器
        private String browser;
        // user-agent
        private String userAgent;
    }
}
```

