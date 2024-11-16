---
title: Chinese Test
description: 这是一个副标题
date: 2020-09-09
slug: test-chinese
image: helena-hertz-wWZzXlDpMog-unsplash.jpg
categories:
    - aop
---

# 什么是 AOP？
AOP（Aspect Oriented Programming），意为**面向切面编程**，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。

AOP是OOP的延续，是软件开发中的一个热点，也是 Spring 框架中的一个重要内容，是函数式编程的一种衍生范型。

利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，同时提高了开发的效率。

举一个简单的例子，如果我们想要在每次调用一个类中的方法之前输出一个日志,那如果我们要在这些方法之前加一个鉴权呢？同样，按照前面的方式，无非就是多 copy 一次嘛

就这样，通过多重 copy 的方式，最终完成了这个业务，而这也就代表着，每次扩展新的方法，你都要去进行 n 次复制粘贴；每次要更改鉴权或者日志的逻辑，你都要去进行 n 次复制粘贴；每次。。。。。。

听起来都让人非常头大，但如果我们将逻辑改成这样呢？

现在来看看 AOP 能实现怎样的效果：
![[AOP 案例.png]]

这样，我们虽然每次都不用去复制代码了，但还是需要在我们需要的位置去调用这个接口，而调用这个代码的位置其实就是 **切面**。我们用具体的代码来看一下，下面实现了一个统一的接口 BeforeAdvice，然后在每个方法之前去调用它。

```java
public interface BeforeAdvice {
    void before();
}

public class LoggingBeforeAdvice implements BeforeAdvice {
    @Override
    public void before() {
        System.out.println("方法调用之前的日志");
    }
}

public class MyService {
    private BeforeAdvice beforeAdvice = new LoggingBeforeAdvice();

    public void myMethodA() {
        beforeAdvice.before();
        // 业务逻辑
        System.out.println("执行 myMethod1A");
    }

    public void myMethodB() {
        beforeAdvice.before();
        // 业务逻辑
        System.out.println("执行 myMethodB");
    }
}
```
# AOP 核心概念
**连接点（Join Point）**：连接点是程序执行中的一个点，这个点可以是方法调用、方法执行、异常抛出等。在 Spring AOP 中，连接点主要是指方法的调用或执行。连接点是通知的实际应用点。

**切点（PointCut）**：由于连接点可能很多（比如一个类中的所有方法），想要把所有连接点罗列出现显然有些困难；切点是一个定义，决定了 **在哪些连接点上应用切面逻辑**。
  切点通过切点表达式（例如：execution(* com.example.service.*.*(..))）来指定匹配的方法和类。切点表达式用于筛选连接点，使得通知只在特定的连接点上执行。
	切点表达式的书写格式：
    [[🐋【AOP】切点表达式]]

**通知（Advice）**：通知是在切点处执行的代码。通知定义了具体的横切逻辑，决定了在方法执行的什么阶段（之前、之后、环绕等）插入横切逻辑。通知有五种类型，我们会在下一部分进行详细的了解；通知就是在 何时 执行 怎样 的逻辑。

**切面（Aspect）**：切面是 AOP 的核心模块，它封装了跨越多个类的关注点，例如日志记录、事务管理或安全控制。切面通过通知（Advice）和切点（Pointcut）来定义在何时、何地应用这些关注点；可以将切面看作是切点（Pointcut）和通知（Advice）的组合。
  切面定义了在何处（切点）以及何时（通知）应用横切逻辑。

# 五种通知类型
前置通知（Before Advice）
在目标方法执行之前执行的通知。可以用来执行日志记录、安全检查等。
```java
@Aspect
@Component
public class LoggingAspect {
    @Before("execution(* com.example.service.*.*(..))")
    public void beforeAdvice() {
        System.out.println("前置通知：方法调用之前执行");
    }
}
```

后置通知（After Advice）
在目标方法执行之后执行的通知，无论方法是成功返回还是抛出异常。常用于清理资源等。
```java
@Aspect
@Component
public class LoggingAspect {
    @After("execution(* com.example.service.*.*(..))")
    public void afterAdvice() {
        System.out.println("后置通知：方法调用之后执行");
    }
}
```

返回后通知（After Returning Advice）
在目标方法成功返回结果之后执行的通知。可以用来记录返回值或对返回值进行处理。
```java
@Aspect
@Component
public class LoggingAspect {
    @AfterReturning(pointcut = "execution(* com.example.service.*.*(..))", returning = "result")
    public void afterReturningAdvice(Object result) {
        System.out.println("返回后通知：方法返回值为 " + result);
    }
}
```

抛出异常后通知（After Throwing Advice）
在目标方法抛出异常后执行的通知。可以用来记录异常信息或执行异常处理逻辑。
```java
@Aspect
@Component
public class LoggingAspect {
    @AfterThrowing(pointcut = "execution(* com.example.service.*.*(..))", throwing = "exception")
    public void afterThrowingAdvice(Exception exception) {
        System.out.println("抛出异常后通知：异常为 " + exception.getMessage());
    }
}
```

环绕通知（Around Advice）
环绕通知在目标方法执行的前后都执行，可以完全控制目标方法的执行，包括决定是否执行目标方法，以及在目标方法执行前后添加自定义逻辑。环绕通知最为强大和灵活。
```java
@Aspect
@Component
public class LoggingAspect {
    @Around("execution(* com.example.service.*.*(..))")
    public Object aroundAdvice(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("环绕通知：方法调用之前");
        Object result = joinPoint.proceed();  // 执行目标方法
        System.out.println("环绕通知：方法调用之后");
        return result;
    }
}
```

这五种通知类型的执行顺序是这样的：
	**前置通知**（Before Advice）
	**环绕通知**（Around Advice）的前半部分
	**目标方法执行**
	**环绕通知**（Around Advice）的后半部分
	**返回后通知**（After Returning Advice）（如果目标方法成功返回）
	**抛出异常后通知**（After Throwing Advice）（如果目标方法抛出异常）
	**后置通知**（After Advice）
