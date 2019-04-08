# Spring 核心类解读

## `ConfigurationClassPostProcessor`

### 类定义
```java
public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor,
		PriorityOrdered, ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware

public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor
```

`ConfigurationClassPostProcessor` 用来解析以 `@Configuration` 注解的配置类，Spring 的 `AnnotationConfigApplicationContext` 上下文在加载初始化的时候，构造器中会 new 一个 `AnnotatedBeanDefinitionReader` 对象，这个对象初始化是会调用 `AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);` 方法，将 `ConfigurationClassPostProcessor` 注入到 Spring 中

### 核心方法




## `AnnotationConfigApplicationContext`
使用 `@Configuration` 注解的类构造时
1. 注册 `ConfigurationClassPostProcessor` 用来处理 `@Configuration` 注解的类
1. 将 `@Configuration` 注解的类注入到 Spring 中
1. 调用 `AbstractApplicationContext` refresh 方法
1. 调用 refresh 方法中的 invokeBeanFactoryPostProcessors 方法，这个方法执行刚刚注入的 `ConfigurationClassPostProcessor` 的 postProcessBeanDefinitionRegistry
1. ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry 这个方法 get 到所有容器中已经注入的 Bean，然后找到用 `@Configuration` 注解的类作为 configCandidates
1. ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry 方法中内部构造一个 `ConfigurationClassParser` 用来转化以 `@Configuration` 注解的类中定义的 Bean
1. 然后使用 `ConfigurationClassBeanDefinitionReader` 来 loadBeanDefinitions ，参数为上一步解析到的 configClasses
1. loadBeanDefinitions方法 中调用 loadBeanDefinitionsForConfigurationClass 方法，loadBeanDefinitionsForConfigurationClass 才是加载的核心方法，源码如下：
```java
private void loadBeanDefinitionsForConfigurationClass(
        ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

    if (trackedConditionEvaluator.shouldSkip(configClass)) {
        String beanName = configClass.getBeanName();
        if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
            this.registry.removeBeanDefinition(beanName);
        }
        this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
        return;
    }
    // 检查是有有 @Import 注解
    if (configClass.isImported()) {
        registerBeanDefinitionForImportedConfigurationClass(configClass);
    }
    // @Bean
    for (BeanMethod beanMethod : configClass.getBeanMethods()) {
        loadBeanDefinitionsForBeanMethod(beanMethod);
    }
    // @ImportResource
    loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
    // 检查 @Import 中的类是否为 ImportBeanDefinitionRegistrar 的子类，然后加载
    loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```

## `@Import`

Spring 中 EnableXXX 注解基本上都是使用 Import 来实现的

### 使用

```java 
public class TestService {

    public void test() {
        System.out.println("test");
    }
}

public class TestServiceSelector implements ImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[] { TestService.class.getName() };
    }
}

@Configuration
@Import(TestServiceSelector.class)
public class SpringLeaningConfiguration {
}

public class AnnotationConfigApplicationContextTest {

    public static void main(String[] args) {
       /* AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext
       (SpringLearningApplication.class);
        TestService testService = context.getBean(TestService.class);
        testService.test();

        System.out.println("------------- display bean start ------------------");
        String[] beanDefinitionNames = context.getBeanDefinitionNames();
        for (String beanDefinitionName : beanDefinitionNames) {
            Object bean = context.getBean(beanDefinitionName);
            System.out.println(beanDefinitionName + ": " + bean.getClass().getName());
        }

        System.out.println("------------- display bean end ------------------");*/

        AnnotationConfigApplicationContext configApplicationContext =
                new AnnotationConfigApplicationContext(
                "com.example.springlearning.autoconfiguration");
        TestService test = configApplicationContext.getBean(TestService.class);
        test.test();
    }
}
```

#### 源码分析
1. 在使用 basepackage 构造 `AnnotationConfigApplicationContext` 的时候，内部会创建 `AnnotatedBeanDefinitionReader` 和 `ClassPathBeanDefinitionScanner` 类，`ClassPathBeanDefinitionScanner` 就是类的扫描器实现，然后调用 scanner.scan(basePackages) 方法，因此类 `SpringLeaningConfiguration` 在会扫描到
1. 然后调用 `AbstractApplicationContext` 的 refresh 中的 invokeBeanFactoryPostProcessors 方法，执行 `ConfigurationClassPostProcessor` postProcessBeanDefinitionRegistry 的方法，然后该方法中会使用 `ConfigurationClassParser` 来 parse 配置类，也就是使用 `@Configuration` 注解的类
1. 在 parse 配置类的过程中，会调用 processImports 方法，该方法就是处理 @Import 注解的核心
1. 找到 `@Import` 注解中的类，在例子中也就是 `TestServiceSelector`，然后通过反射调用其中的方法，返回需要注入的类名称，其中 `ImportBeanDefinitionRegistrar` 也在这段逻辑中做处理
1. 将 selectImports 到的类封装成 `ConfigurationClass` 然后 add 到 `ConfigurationClassParser` 中到 this.configurationClasses 中
1. `ConfigurationClassBeanDefinitionReader` 将 get 到 this.configurationClasses 使用 loadBeanDefinitionsForConfigurationClass 方法进行加载
1. 这个方法 loadBeanDefinitionsForConfigurationClass 会对 `@Import`、`@Bean`、`@ImportResource` 和 `ImportBeanDefinitionRegistrar` 类型进行分别加载

#### Spring 中的使用
- EnableAspectJAutoProxy
- EnableScheduling
- etc


## `BeanPostProcessor`

### 分析
`BeanPostProcessor` 用来 Bean 实例化过程中被 Spring 容器回调到方法，当中有 postProcessBeforeInitialization 和 postProcessAfterInitialization 两个钩子方法，调用时机在
`AbstractAutowireCapableBeanFactory` 的 initializeBean 方法中，initializeBean 逻辑如下：

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    }
    else {
        // set BeanNameAware，BeanClassLoaderAware，BeanFactoryAware
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // invoke BeanPostProcessor.postProcessBeforeInitialization
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        // invoke InitializingBean，init method
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                (mbd != null ? mbd.getResourceDescription() : null),
                beanName, "Invocation of init method failed", ex);
    }
    if (mbd == null || !mbd.isSynthetic()) {
        // invoke BeanPostProcessor.postProcessAfterInitialization
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}
```

### Spring 中的运用

- ApplicationContextAwareProcessor
```java 
class ApplicationContextAwareProcessor implements BeanPostProcessor {

	private final ConfigurableApplicationContext applicationContext;

	private final StringValueResolver embeddedValueResolver;


	/**
	 * Create a new ApplicationContextAwareProcessor for the given context.
	 */
	public ApplicationContextAwareProcessor(ConfigurableApplicationContext applicationContext) {
		this.applicationContext = applicationContext;
		this.embeddedValueResolver = new EmbeddedValueResolver(applicationContext.getBeanFactory());
	}


	@Override
	@Nullable
	public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
		AccessControlContext acc = null;

		if (System.getSecurityManager() != null &&
				(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
						bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
						bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}

		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		}
		else {
			invokeAwareInterfaces(bean);
		}

		return bean;
	}

	private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof EnvironmentAware) {
				((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
			}
			if (bean instanceof EmbeddedValueResolverAware) {
				((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
			}
			if (bean instanceof ResourceLoaderAware) {
				((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
			}
			if (bean instanceof ApplicationEventPublisherAware) {
				((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
			}
			if (bean instanceof MessageSourceAware) {
				((MessageSourceAware) bean).setMessageSource(this.applicationContext);
			}
			if (bean instanceof ApplicationContextAware) {
				((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
			}
		}
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) {
		return bean;
	}

}
```
- AutowiredAnnotationBeanPostProcessor

## AOP 分析

### 例子

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
http://www.springframework.org/schema/aop
http://www.springframework.org/schema/aop/spring-aop-3.0.xsd">
    <bean id="daoImpl" class="org.xrq.action.aop.DaoImpl" />
    <bean id="timeHandler" class="org.xrq.action.aop.TimeHandler" />
    <aop:config proxy-target-class="true">
        <aop:aspect id="time" ref="timeHandler">
            <aop:pointcut id="addAllMethod" expression="execution(* org.xrq.action.aop.Dao.*(..))" />
            <aop:before method="printTime" pointcut-ref="addAllMethod" />
            <aop:after method="printTime" pointcut-ref="addAllMethod" />
        </aop:aspect>
    </aop:config>
</beans>
```
Spring 容器对代理的 Bean，在创建解析 BeanDefinition 的时候做了特殊的处理，代码直接定位到 DefaultBeanDefinitionDocumentReader.parseBeanDefinitions

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    if (delegate.isDefaultNamespace(root)) {
        NodeList nl = root.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (node instanceof Element) {
                Element ele = (Element) node;
                if (delegate.isDefaultNamespace(ele)) {
                    parseDefaultElement(ele, delegate);
                }
                else {
                    delegate.parseCustomElement(ele);
                }
            }
        }
    }
    else {
        delegate.parseCustomElement(root);
    }
}
```
在处理 AOP Bean 时，通过 namespaceURI，查找到 AOP 处理解析 Bean 对 handler，然后通过 handler 解析并且注册 Bean，AOP 的 handler 为 `AopNamespaceHandler`
- registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
- registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
- registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());
- registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
`ConfigBeanDefinitionParser` 基本流程就是：
1. 注入 `AspectJAwareAdvisorAutoProxyCreator` Bean
1. 解析 ASPECT 节点
1. 解析 ADVISOR 节点
1. 解析 POINTCUT 节点
1. 注入 Bean
