---
author: Ryan Yuan
title: Spring AOP 代理类创建方法解析
description: wrapIfNessary 方法梳理
date: 2024-11-17
image: image.png
categories:
    - SpringAOP源码解析
---
[Spring代理类创建流程梳理]({{< ref "../Spring代理类创建流程梳理/index.md" >}} "关于我们")

沿着上节的流程，详细的来看看 `wrapIfNecessary()` 方法：
```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {  
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {  
       return bean;  
    }  
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {  
       return bean;  
    }  
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {  
       this.advisedBeans.put(cacheKey, Boolean.FALSE);  
       return bean;  
    }  
  
    // 如果该类的 Advisor 不为空，创建代理类
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);  
    if (specificInterceptors != DO_NOT_PROXY) {  
       this.advisedBeans.put(cacheKey, Boolean.TRUE);  
       Object proxy = createProxy(  
             bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));  
       this.proxyTypes.put(cacheKey, proxy.getClass());  
       return proxy;  
    }  
  
    this.advisedBeans.put(cacheKey, Boolean.FALSE);  
    return bean;  
}
```
# 不生成代理类的情况
```java
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {  
       return bean;  
    }  
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {  
       return bean;  
    }  
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {  
       this.advisedBeans.put(cacheKey, Boolean.FALSE);  
       return bean;  
    }  
```
以下的四种情况将不会生成代理类：
- BeanName 不为空且在 targetSourcedBeans 集合中，直接返回原始 bean。
- AdvisedBeans 缓存中存在 cacheKey 且值为 false，直接返回原始 bean。
- Bean 属于基础设施类或满足跳过条件，将 cacheKey 标记为 false 并返回原始 bean。
- 没有与 Bean 匹配的 Advisor
# 获取适配类的 Advisors
```java
Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);  
```

`org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator#getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, @Nullable TargetSource targetSource)`
```java
@Override  
@Nullable  
protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {  
    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);  
    if (advisors.isEmpty()) {  
       return DO_NOT_PROXY;  
    }  
    return advisors.toArray();  
}
```

`org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator#findEligibleAdvisors(Class<?> beanClass, String beanName)`
```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {  
    List<Advisor> candidateAdvisors = findCandidateAdvisors();  
    List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);  
    extendAdvisors(eligibleAdvisors);  
    if (!eligibleAdvisors.isEmpty()) {  
       eligibleAdvisors = sortAdvisors(eligibleAdvisors);  
    }  
    return eligibleAdvisors;  
}
```
首先调用 `findCandidateAdvisors` 拿到所有的候选者 `Advisor`，再调用 `findAdvisorsThatCanApply` 选出适配该 Bean 的 `Advisor`。

最终会调用到这个方法：
`org.springframework.aop.support.AopUtils#findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz)`
```java
public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {  
    if (candidateAdvisors.isEmpty()) {  
       return candidateAdvisors;  
    }  
    List<Advisor> eligibleAdvisors = new ArrayList<>();  
    for (Advisor candidate : candidateAdvisors) {  
       if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {  
          eligibleAdvisors.add(candidate);  
       }  
    }  
    boolean hasIntroductions = !eligibleAdvisors.isEmpty();  
    for (Advisor candidate : candidateAdvisors) {  
       if (candidate instanceof IntroductionAdvisor) {  
          // already processed  
          continue;  
       }  
       if (canApply(candidate, clazz, hasIntroductions)) {  
          eligibleAdvisors.add(candidate);  
       }  
    }  
    return eligibleAdvisors;  
}
```
这个方法 findAdvisorsThatCanApply 用于过滤和筛选出可以应用于特定目标类 clazz 的增强器（Advisor），即 **找出与该类匹配的 Advisor 列表**。
首先遍历 candidateAdvisors 列表，筛选出所有的 IntroductionAdvisor。
	对于每个 IntroductionAdvisor，调用 canApply(candidate, clazz) 方法判断它是否适用于目标类 clazz。
	如果适用，就将它添加到 eligibleAdvisors 列表中。
再次遍历 candidateAdvisors 列表，排除 IntroductionAdvisor，只处理普通的 Advisor。
	对于每个普通的 Advisor，调用 canApply(candidate, clazz, hasIntroductions) 方法检查它是否适用于目标类 clazz。
	如果适用，将其添加到 eligibleAdvisors 列表中。
# 创建代理类
```java
    if (specificInterceptors != DO_NOT_PROXY) {  
       this.advisedBeans.put(cacheKey, Boolean.TRUE);  
       Object proxy = createProxy(  
             bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));  
       this.proxyTypes.put(cacheKey, proxy.getClass());  
       return proxy;  
    }  
```
如果 `specificInterceptors` 不为 null 就会去创建代理；

`org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#createProxy(Class<?> beanClass, @Nullable String beanName,  @Nullable Object[] specificInterceptors, TargetSource targetSource)`
```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,  
       @Nullable Object[] specificInterceptors, TargetSource targetSource) {  
  
    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {  
       AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);  
    }  
  
    ProxyFactory proxyFactory = new ProxyFactory();  
    proxyFactory.copyFrom(this);  
  
    if (proxyFactory.isProxyTargetClass()) {  
       // Explicit handling of JDK proxy targets and lambdas (for introduction advice scenarios)  
       if (Proxy.isProxyClass(beanClass) || ClassUtils.isLambdaClass(beanClass)) {  
          // Must allow for introductions; can't just set interfaces to the proxy's interfaces only.  
          for (Class<?> ifc : beanClass.getInterfaces()) {  
             proxyFactory.addInterface(ifc);  
          }  
       }  
    }  
    else {  
       // No proxyTargetClass flag enforced, let's apply our default checks...  
       if (shouldProxyTargetClass(beanClass, beanName)) {  
          proxyFactory.setProxyTargetClass(true);  
       }  
       else {  
          evaluateProxyInterfaces(beanClass, proxyFactory);  
       }  
    }  
  
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);  
    proxyFactory.addAdvisors(advisors);  
    proxyFactory.setTargetSource(targetSource);  
    customizeProxyFactory(proxyFactory);  
  
    proxyFactory.setFrozen(this.freezeProxy);  
    if (advisorsPreFiltered()) {  
       proxyFactory.setPreFiltered(true);  
    }  
  
    // Use original ClassLoader if bean class not locally loaded in overriding class loader  
    ClassLoader classLoader = getProxyClassLoader();  
    if (classLoader instanceof SmartClassLoader && classLoader != beanClass.getClassLoader()) {  
       classLoader = ((SmartClassLoader) classLoader).getOriginalClassLoader();  
    }  
    return proxyFactory.getProxy(classLoader);  
}
```
该方法的返回值为：`proxyFactory.getProxy(classLoader);`，即调用 `proxyFactory` 的创建代理方法。
## 代理工厂的初始化配置
整个方法其实就是对代理工厂的一个构建，在调用 `proxyFactory.copyFrom(this); `，会将 `Creator` 的配置复制到代理工厂中：
`org.springframework.aop.framework.ProxyConfig#copyFrom(ProxyConfig other)`
```java
public void copyFrom(ProxyConfig other) {  
    Assert.notNull(other, "Other ProxyConfig object must not be null"); 
    this.proxyTargetClass = other.proxyTargetClass;  
    this.optimize = other.optimize;  
    this.exposeProxy = other.exposeProxy;  
    this.frozen = other.frozen;  
    this.opaque = other.opaque;  
}
```
`proxyTargetSource`：设置是否直接代理目标类，而不是仅代理特定的接口。默认值为 "false"。
- 将其设置为 "true" 时，会强制对 TargetSource 所暴露的目标类进行代理。
- 如果目标类是一个接口，则会为该接口创建一个 JDK 代理。
- 如果目标类是一个普通的类，则会为该类创建一个 CGLIB 代理。

`optimize`：设置代理是否应执行“积极优化”。
- “积极优化”的具体含义因代理类型而异，但通常会涉及某种权衡，默认值为 "false"。
- 在 Spring 当前的代理选项中，此标志实际上会强制使用 CGLIB 代理，但不会执行任何类验证检查（如对 final 方法的检查等）。

`opaque`：设置由此配置创建的代理是否应该禁止被强制转换为 {@link Advised} 以查询代理状态。
- 默认值为 "false"，这意味着任何 AOP 代理都可以被强制转换为 {@link Advised}。

`frozen`：决定代理的配置是否可以在运行时修改。
- 默认情况下，配置是可变的，调用者可以通过将代理转换为 Advised 接口来修改通知链或其他配置。
- 当其设置为 true 的时候：
	- 配置被锁定，不能再动态更改通知或切面配置。
	- 增强系统的稳定性和性能，因为代理不需要支持动态修改。
## 根据是否代理目标类执行不同的操作
对应源码的这一部分：
```java
if (proxyFactory.isProxyTargetClass()) {  
       // Explicit handling of JDK proxy targets and lambdas (for introduction advice scenarios)  
       if (Proxy.isProxyClass(beanClass) || ClassUtils.isLambdaClass(beanClass)) {  
          // Must allow for introductions; can't just set interfaces to the proxy's interfaces only.  
          for (Class<?> ifc : beanClass.getInterfaces()) {  
             proxyFactory.addInterface(ifc);  
          }  
       }  
    }  
    else {  
       // No proxyTargetClass flag enforced, let's apply our default checks...  
       if (shouldProxyTargetClass(beanClass, beanName)) {  
          proxyFactory.setProxyTargetClass(true);  
       }  
       else {  
          evaluateProxyInterfaces(beanClass, proxyFactory);  
       }  
    }  
```
上面我们提到，`proxyTargetSource`：设置是否直接代理目标类，而不是仅代理特定的接口。默认值为 "false"。
	如果为 true 的话，对 JDK 代理类和 lambda 接口的实现类做一些特殊的处理，也就是保存接口的类型。
	如果设置为 false 的话，执行默认的操作
		首先检查 Bean 定义中是否明确制定了需要代理类而不是接口，如果是的话，将 `proxyTargetSource` 设置为 true。
		反之调用 `evaluateProxyInterfaces`，评估 Bean 是否有实现的接口，我们说的如果 **代理类没有实现接口，使用 CgLib 代理，反之，使用 JDK 代理正是这个方法的原因**。

全限定名：`org.springframework.aop.framework.ProxyProcessorSupport#evaluateProxyInterfaces(Class<?> beanClass, ProxyFactory proxyFactory)`
方法注释：
```java
/**  
 * Check the interfaces on the given bean class and apply them to the {@link ProxyFactory},  
 * if appropriate. * <p>Calls {@link #isConfigurationCallbackInterface} and {@link #isInternalLanguageInterface}  
 * to filter for reasonable proxy interfaces, falling back to a target-class proxy otherwise. * @param beanClass the class of the bean  
 * @param proxyFactory the ProxyFactory for the bean  
 */

/**
 * 检查给定 Bean 类上的接口，并在合适的情况下将这些接口应用到 {@link ProxyFactory}。
 * 
 * <p>会调用 {@link #isConfigurationCallbackInterface} 和 {@link #isInternalLanguageInterface}
 * 方法来过滤出合理的代理接口。如果没有找到合理的接口，则会退回到基于目标类（target-class）的代理。
 * 
 * @param beanClass    Bean 的类
 * @param proxyFactory 用于创建代理的 {@link ProxyFactory}
 */
```
```java
protected void evaluateProxyInterfaces(Class<?> beanClass, ProxyFactory proxyFactory) {  
    Class<?>[] targetInterfaces = ClassUtils.getAllInterfacesForClass(beanClass, getProxyClassLoader());  
    boolean hasReasonableProxyInterface = false;  
    for (Class<?> ifc : targetInterfaces) {  
       if (!isConfigurationCallbackInterface(ifc) && !isInternalLanguageInterface(ifc) &&  
             ifc.getMethods().length > 0) {  
          hasReasonableProxyInterface = true;  
          break;  
       }  
    }  
    if (hasReasonableProxyInterface) {  
       // Must allow for introductions; can't just set interfaces to the target's interfaces only.  
       for (Class<?> ifc : targetInterfaces) {  
          proxyFactory.addInterface(ifc);  
       }  
    }  
    else {  
       proxyFactory.setProxyTargetClass(true);  
    }  
}
```
这个方法的作用是从目标类（beanClass）中提取需要代理的接口，并将它们添加到代理工厂（ProxyFactory）中，而如果没有找到合适的接口（即所有接口都被过滤掉），则会使用目标类进行代理（即启用 CGLIB 动态代理）。
使用 isConfigurationCallbackInterface 和 isInternalLanguageInterface 方法对接口进行筛选，确保只代理合理的接口。
- 合理的接口通常是用户定义的接口，而非 Spring 内部使用的回调接口或与语言机制相关的内部接口。
这两个方法将这些接口排除在外：
```java
	protected boolean isConfigurationCallbackInterface(Class<?> ifc) {
		return (InitializingBean.class == ifc || DisposableBean.class == ifc || Closeable.class == ifc ||
				AutoCloseable.class == ifc || ObjectUtils.containsElement(ifc.getInterfaces(), Aware.class));
	}
	protected boolean isInternalLanguageInterface(Class<?> ifc) {
		return (ifc.getName().equals("groovy.lang.GroovyObject") ||
				ifc.getName().endsWith(".cglib.proxy.Factory") ||
				ifc.getName().endsWith(".bytebuddy.MockAccess"));
	}
```
如果筛选后没有适合的接口，则会回退到基于目标类的代理方式（ **触发 CGLIB 动态代理** ）。
## 填充代理工厂的其他属性
对应源码的这一部分：
```java
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);  
    proxyFactory.addAdvisors(advisors);  
    proxyFactory.setTargetSource(targetSource);  
    customizeProxyFactory(proxyFactory);  
  
    proxyFactory.setFrozen(this.freezeProxy);  
    if (advisorsPreFiltered()) {  
       proxyFactory.setPreFiltered(true);  
    }  
  
    // Use original ClassLoader if bean class not locally loaded in overriding class loader  
    ClassLoader classLoader = getProxyClassLoader();  
    if (classLoader instanceof SmartClassLoader && classLoader != beanClass.getClassLoader()) {  
       classLoader = ((SmartClassLoader) classLoader).getOriginalClassLoader();  
    }  
```
获取并构建一个 **Advisor** 列表，这个 `specificInterceptors` 是从 `wrapIfNecessory`中传递下来的，然后将这些 Advisor 添加到代理工厂。
Spring 使用 SmartClassLoader 检测目标类的类加载器和代理类的类加载器是否一致：
	如果代理类的类加载器与目标类不一致，可能会导致类加载问题，因此 Spring 会尝试调整使用的类加载器。
## 正式开始创建代理类
```java
return proxyFactory.getProxy(classLoader);
```
其最终会调用到这个方法：

全限定名：`org.springframework.aop.framework.DefaultAopProxyFactory#createAopProxy(AdvisedSupport config)`
接口定义
```java
	/**
	 * Create an {@link AopProxy} for the given AOP configuration.
	 * @param config the AOP configuration in the form of an
	 * AdvisedSupport object
	 * @return the corresponding AOP proxy
	 * @throws AopConfigException if the configuration is invalid
	 */
	 
	/**
	 * 为给定的 AOP 配置创建一个 {@link AopProxy} 实例。
	 * 
	 * @param config 以 {@link AdvisedSupport} 对象形式表示的 AOP 配置信息
	 * @return 对应的 AOP 代理
	 * @throws AopConfigException 如果配置无效，则抛出此异常
	 / 
```
```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {  
    if (!NativeDetector.inNativeImage() &&  
          (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config))) {  
       Class<?> targetClass = config.getTargetClass();  
       if (targetClass == null) {  
          throw new AopConfigException("TargetSource cannot determine target class: " +  
                "Either an interface or a target is required for proxy creation.");  
       }  
       if (targetClass.isInterface() || Proxy.isProxyClass(targetClass) || ClassUtils.isLambdaClass(targetClass)) {  
          return new JdkDynamicAopProxy(config);  
       }  
       return new ObjenesisCglibAopProxy(config);  
    }  
    else {  
       return new JdkDynamicAopProxy(config);  
    }  
}
```
传入的参数 `AdvisedSupport` 实际上就是我们上面构建的代理工厂，其实现了 `AdvisedSupport` 接口。
我们再来梳理一下这个配置中对一些值：
- isOptimize：默认 false，不进行智能代理优化
- isProxyTargetClass：**类或者Creator 指定需要代理的是目标类** 或 **类没有实现任何可用接口** 时为 true，其他时候为 false
### 创建 CgLib 代理类
对应源码的这一部分：
```java
    if (!NativeDetector.inNativeImage() &&  
          (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config))) {  
       Class<?> targetClass = config.getTargetClass();  
       if (targetClass == null) {  
          throw new AopConfigException("TargetSource cannot determine target class: " +  
                "Either an interface or a target is required for proxy creation.");  
       }  
       if (targetClass.isInterface() || Proxy.isProxyClass(targetClass) || ClassUtils.isLambdaClass(targetClass)) {  
          return new JdkDynamicAopProxy(config);  
       }  
       return new ObjenesisCglibAopProxy(config);  
    }  
```
`!NativeDetector.inNativeImage()`  用于检测当前环境能否使用 CgLib 代理，如果允许的话，下面的条件符合任意一条都会出发 CgLib 代理：
- isOptimize 智能代理优化
- isProxyTargetClass 需要代理目标类
- hasNoUserSuppliedProxyInterfaces，检测接口，如果发现用户没有提供任何自定义接口，Spring 可能会选择使用 CGLIB 代理。
还有一些特殊的情况，创建的是 JDK 代理类，这个放到下面去说。
### 创建 JDK 代理类
当下面的条件不符合
```java
!NativeDetector.inNativeImage() && (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config))
```
或者代理的类是一个接口、代理的类已经是一个 JDK 代理类、代理的类是一个 lambda 表达式的实现类
```java
targetClass.isInterface() || Proxy.isProxyClass(targetClass) || ClassUtils.isLambdaClass(targetClass)
```
的时候，会创建 JDK 代理类。
