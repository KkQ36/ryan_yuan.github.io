---
author: Ryan Yuan
title: 实现一个简单的 AOP 切面
date: 2024-11-12
image: pawel-czerwinski-8uZPynIu-rQ-unsplash.jpg
categories:
    - SpringAOP源码解析
---

# 前言
AOP：意为面向切面编程，通过预编译的方式和运行期间动态代理实现程序功能功能的统一维护。

具体：[[🐿️【AOP】AOP 核心概念]]

既然要实现 AOP 就一定离不开代理，Spring 中使用两种代理方式：CgLib 动态代理和 AOP 动态代理，还通过引入 `AspectJ` 来简化切面编程。

AOP 执行过程可以简单理解为对 切点(JointPoint) 执行 通知(Advice) 中定义的逻辑，这就引出两个核心的问题需要解决：
- 通知在何处执行？
- 如何判断当前执行的方法是否符合切点逻辑？

动态代理可以让我们自由的在方法执行前后插入自己的逻辑，这就解决了通知在何处执行的问题；

但是动态代理（无论是 JDKProxy 的 `invoke()` 还是 CgLib 的 `interctpt()`）都是在被代理的类中的**任何方法**执行前执行的，所以判断该方法是否需要执行切点逻辑需要我们来实现。

![[【spring-aop】实现一个简单的 AOP 切面 架构图.png|800]]
上面展示的是一个代理类处理 AOP 的逻辑，其中 `MethodInterceptor` 是方法执行前的一个拦截器；`MethodMatcher` 是一个方法匹配器，通过它来判断接下来要执行的方法之前是否需要处理代理逻辑；
如果被代理的方法符合切点表达式，则按照顺序执行定义的代理逻辑，这样一个简单的 AOP 逻辑就打通了。

# 案例
## 案例准备
```java
public class UserService implements IUserService {  
  
    public void queryUserName(String uId) {  
        System.out.println("查询用户名称为：ryan");  
    }  
  
    public void queryUserId(String name) {  
        System.out.println("查询用户ID为：10001");  
    }  
  
}
```
被代理的 UserService 类。

## 定义方法匹配器
```java
class MethodMatcherImpl implements MethodMatcher {  
  
    private final PointcutExpression pointcutExpression;  
  
    MethodMatcherImpl(String expression) {  
        PointcutParser parser = PointcutParser.getPointcutParserSupportingAllPrimitivesAndUsingContextClassloaderForResolution();  
        pointcutExpression = parser.parsePointcutExpression(expression);  
    }  
  
    @Override  
    public boolean matches(Method method, Class<?> aClass) {  
        return pointcutExpression.matchesMethodExecution(method).alwaysMatches();  
    }  
  
    @Override  
    public boolean matches(Method method, Class<?> aClass, Object[] objects) {  
        return false;  
    }  
  
    @Override  
    public boolean isRuntime() {  
        return false;  
    }  
  
}
```
- 这是一个方法匹配类，用于检测输入的方法是否符合切点表达式

## 定义方法拦截器
```java
class CgLibInterceptor implements MethodInterceptor {  
  
    private final Object target;  
    private final MethodMatcher methodMatcher;  
    CgLibInterceptor(Object target, String expression) {  
        this.methodMatcher = new MethodMatcherImpl(expression);  
        this.target = target;  
    }  
  
    @Override  
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {  
        if (methodMatcher.matches(method, target.getClass())) {  
            System.out.println("cglib代理：执行切面逻辑");  
            return methodProxy.invoke(target, objects);  
        }  
        return methodProxy.invoke(target, objects);  
    }  
}
```
- 提供给 CgLib 的 CallBack 类，根据方法是否匹配切点表达式来决定是否执行切面逻辑。

## 创建代理类
```java
public class ProxyTest {  
  
    final static UserService target = new UserService();  
  
    public static void main(String[] args) throws IOException, InterruptedException {  
        Enhancer enhancer = new Enhancer();  
        enhancer.setSuperclass(UserService.class);  
  
        enhancer.setCallback(new CgLibInterceptor(target,  
                "execution(* com.ryan.aop_test.service.impl.UserService.queryUserName(..))"));  
  
        UserService proxyService = (UserService) enhancer.create();  
  
        proxyService.queryUserName("10001");  
        System.out.println("======");  
        proxyService.queryUserId("ryan");  
  
    }  
  
}
```
- 测试类，这里使用切点表达式匹配 `UserService` 类中的 `queryUserName` 方法
- 输出案例中，也只有这个方法会被代理

输出结果
```
cglib代理
查询用户名称为：ryan
======
查询用户ID为：10001
```