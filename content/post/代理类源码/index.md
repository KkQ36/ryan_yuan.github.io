---      
author: Ryan Yuan      
title: Spring 生成的代理类的结构以及拦截器链式调用      
date: 2024-11-17      
image: image.png      
categories:      
    - SpringAOP源码解析      
---      
      
```java      
public class UserService implements IUserService {        
    @Override        
    public void queryUserName(String uId) {        
        System.out.println("查询用户名称为：ryan");        
    }        
          
    @Override        
    public void queryUserId(String name) {        
        System.out.println("查询用户ID为：10001");        
    }        
}      
```      
```java      
public class UserMapper {        
    public void query() {        
        System.out.println("query");        
    }        
}      
```      
通过上一节关于创建代理类方法的解析，上面的两个类如果被代理的话，第一个类实现了接口会使用 JDK 代理，第二个类没有实现接口，将会使用 CgLib 动态代理。      
使用下面的方法来验证一下：      
```java      
public class Main {        
        
    public static void main(String[] args) {        
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring-aop.xml");        
        IUserService service = applicationContext.getBean("userService", IUserService.class);        
        System.out.println(service.getClass());        
        service.queryUserName("10001");        
        
        System.out.println("======");        
        
        UserMapper userMapper = applicationContext.getBean("userMapper", UserMapper.class);        
        System.out.println(userMapper.getClass());        
        userMapper.query();        
    }        
        
}      
```      
**输出结果**：      
```      
class com.sun.proxy.$Proxy3      
before      
查询用户名称为：ryan      
after      
======      
class com.ryan.aop_test.bean.UserMapper$$EnhancerBySpringCGLIB$$ef0e8fc      
before      
query      
after      
```      
      
接下来，通过 debug 的方式，来具体看看方法是如何执行的：      
# JDK 生成的代理类      
调试的方法为 `UserService` 的 `queryUserName` 方法：      
`org.springframework.aop.framework.JdkDynamicAopProxy#invoke(Object proxy, Method method, Object[] args)`      
这个类实现了 `InvocationHandler` 接口，所以，我们需要重点关注这个类的 `invoke()` 方法`      
```java      
	@Override      
	@Nullable      
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {      
      
		try {      
			// ......提供默认方法      
      
			Object retVal;      
			// (1)(在下面会有详细讲解)      
			if (this.advised.exposeProxy) {      
				// Make invocation available if necessary.      
				oldProxy = AopContext.setCurrentProxy(proxy);      
				setProxyContext = true;      
			}      
			target = targetSource.getTarget();      
			Class<?> targetClass = (target != null ? target.getClass() : null);      
			      
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);      
      
			if (chain.isEmpty()) {      
				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);      
				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);      
			}      
			else {      
				MethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);      
				retVal = invocation.proceed();      
			}      
      
			// 处理返回值。。。。。。      
			return retVal;      
		}      
		finally {      
			// ......      
		}      
	}      
```      
## 暴露代理类      
关于上面代码 (1) 位置的 AopContext.setCurrentProxy(proxy) 方法：      
这个方法会在我们将 `this.advised.exposeProxy` 设置为 `true` 的时候调用，用来将 `proxy` 代理对象公开；具体来说，当代理对象的一个方法被调用时，该方法内部可能会调用目标对象的其他方法。如果这些内部调用希望也能够触发AOP拦截器（即被增强），那么这些调用需要通过代理对象来完成。这就要求代理对象在方法内部是可以被访问的；比如通过这样的方式就可以在 `methodA()` 中去通过动态代理对象去调用 `methodB()`。      
```java      
public class MyService {      
    public void methodA() {      
        System.out.println("Executing methodA");      
        // Call methodB through proxy to apply AOP advice      
        ((MyService) AopContext.currentProxy()).methodB();      
    }      
      
    public void methodB() {      
        System.out.println("Executing methodB");      
    }      
}      
```      
常用的开启方式是在配置类上加入注解，类似如下的这种形式：      
```java      
@Configuration      
@EnableAspectJAutoProxy(exposeProxy = true)      
public class AppConfig {      
    // Bean definitions      
}      
```      
## 调用链路      
对应源码中的这一部分：      
```java      
List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);      
if (chain.isEmpty()) {      
	Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);      
	retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);      
}      
else {      
	MethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);      
	retVal = invocation.proceed();      
}      
```      
我们配置的 Advisors 会被构建成一个拦截器链，在方法执行的过程中调用。      
      
如果拦截器链路为空，直接利用反射来调用。      
      
如果不为空的话，执行调用逻辑      
```      
chain = {ArrayList@2615}  size = 3      
 0 = {ExposeInvocationInterceptor@2619}       
 1 = {AfterReturningAdviceInterceptor@2620}       
 2 = {MethodBeforeAdviceInterceptor@2621}       
```      
在我们的配置下，拦截器 chain 中有这些拦截器。      
处理我们配置的 before 和 afterReturning 两个建议以外，还有一个 `ExposeInvocationInterceptor`，这是一个用于拦截方法调用的组件，并且在拦截过程中会将 `MethodInvocation` 对象公开，使其可以在方法执行期间被其他代码访问；可以将其视作 AOP 调用链的一个入口，它会将方法执行信息 `MethodInvocation` 对象存储到当前线程的 `ThreadLocal` 中，以便其他方法能够快速的通过 `currentInvocation()` 方法来访问当前方法执行对象。      
      
```java      
MethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);      
retVal = invocation.proceed();      
```      
首先创建了一个 `MethodInvocation` 类，然后调用其 `proceed()` 方法      
      
`org.springframework.aop.framework.ReflectiveMethodInvocation#proceed()`      
```java      
public Object proceed() throws Throwable {      
		// We start with an index of -1 and increment early.      
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {      
			return invokeJoinpoint();      
		}      
		Object interceptorOrInterceptionAdvice = this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);      
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {      
			// Evaluate dynamic method matcher here: static part will already have      
			// been evaluated and found to match.      
			InterceptorAndDynamicMethodMatcher dm =      
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;      
			Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());      
			if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {      
				return dm.interceptor.invoke(this);      
			}      
			else {      
				// 动态方法匹配失败，跳过这个方法继续执行下一个拦截器      
				return proceed();      
			}      
		}      
		else {      
			// It's an interceptor, so we just invoke it: The pointcut will have        
			// been evaluated statically before this object was constructed.        
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);      
		}      
	}      
```      
方法中先判断是否需要动态匹配，匹配成功后，最终都会执行到 `MethodInterceptor` 的 invoke 方法。      
`org.springframework.aop.interceptor.ExposeInvocationInterceptor.#nvoke(MethodInvocation mi)`      
```java      
public Object invoke(MethodInvocation mi) throws Throwable {        
    MethodInvocation oldInvocation = invocation.get();        
    invocation.set(mi);        
    try {        
       return mi.proceed();        
    }        
    finally {        
       invocation.set(oldInvocation);        
    }        
}      
```      
拦截器中都会包含这样一个方法：`mi.proceed()`，这个方法会再次调用 `MethodInvocation` 的 `proceed` 方法      
很明显，这是一个递归调用，递归的出口就是：      
```java      
if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {      
	return invokeJoinpoint();      
}      
```      
通过这样的方式，代理类执行了所有的拦截器逻辑，将结果返回。      
# CgLib 生成的代理类      
[【Spring AOP 源码解析】彻底搞懂 AOP 是如何执行的](https://blog.csdn.net/weixin_74895237/article/details/140379653?spm=1001.2014.3001.5501)      
      