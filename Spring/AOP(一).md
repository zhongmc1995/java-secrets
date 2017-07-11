## Spring AOP
### Java中的动态代理总结
在AOP学习之前，有必要对java中的动态代理进行系统的总结下，因为JDK动态代理与CGLib动态代理均是实现Spring AOP的基础
java的动态代理分为JDK动态代理和Cglib动态代理：
* JDK动态代理只能代理实现了接口的目标类，未实现接口的目标类是无法用JDK动态代理的
* Cglib动态代理是一个第三方的代理库，需要额外引入Jar包，它采用了非常底层的字节码技术，其原理是通过字节码技术为一个类创建子类，
并在子类中采用方法拦截的技术拦截所有父类方法的调用，顺势织入横切逻辑代理的目标类无需实现
任何接口即可产生代理对象
* 通常情况下Cglib创建代理的速度比较慢，但创建代理后的运行速度却非常快，而JDK代理却正好相反，创建代理对象快
运行速度稍微慢些，因此，通常在不需要频繁创建代理对象的情况下（比如单利模式），适合使用Cglib代理，反之，适合使用JDK代理
### 准备工作
之后的例子项目是基于Maven构建的，主要依赖Spring，pom.xml如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.zmc</groupId>
    <artifactId>aop</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>4.3.7.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>4.3.7.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-beans</artifactId>
            <version>4.3.7.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-aop</artifactId>
            <version>4.3.7.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjweaver</artifactId>
            <version>1.8.9</version>
        </dependency>
    </dependencies>
    
</project>
```
spring-core是其他模块的基础，spring-beans提供了IOC和DI等功能，spring-context提供了容器的上下文对象，
可以用于获取spring容器中的bean，spring-aop是spring aop的核心，其依赖aspectj。
再定义几个基础类用于代理使用，定义一个BookService接口,其中定义了一个添加书的方法：
```java
/**
 * Created by zhongmc on 2017/7/7.
 */
public interface BookService {
    Book addBook(Book book);

}

class Book implements Serializable {
    private String id;
    private String name;
    private String isbn;
    public Book(String id, String name, String isbn) {
        this.id = id;
        this.name = name;
        this.isbn = isbn;
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getIsbn() {
        return isbn;
    }

    public void setIsbn(String isbn) {
        this.isbn = isbn;
    }

    @Override
    public String toString() {
        return "Book{" +
                "id='" + id + '\'' +
                ", name='" + name + '\'' +
                ", isbn='" + isbn + '\'' +
                '}';
    }
}
```
接口的实现类BookServiceImpl:
```java
/**
 * Created by zhongmc on 2017/7/7.
 */
@Service("bookService")
public class BookServiceImpl {
    public Book addBook(Book book) throws Exception {
        if (book != null) {
            System.out.println("add book : " + book);
            return book;
        }else{
            throw new Exception("book is null not allowed");
        }
    }
}
```
这样准备工作就完成了，BookServiceImpl实现了BookService接口，其中只有一个添加书的方法
### Spring AOP
在Spring AOP中定义了三种增强（通知），前置增强（Before Advice），后置增强（After Advice），环绕增强（Around Advice）
这三种增强就好比代理方法中的before()和after()方法，通常写自己需要横切的增强逻辑；前置增强是在方法调用前被调用，后置增强
在方法执行完返回值前被调用，环绕增强可以简单的认为就是同时有前置和后置增强的效果
### 前置、后置、环绕增强（编程式）
#### 前置增强
前置增强需要实现一个MethodBeforeAdvice接口，这个接口是spring aop提供的，其中定义了一个增强的方法before(Method method, Object[] args, Object target)
* method：代理的方法
* args：代理方法中的参数
* target：代理的目标类
下面来实现一个前置增强：
```java
/**
 * Created by zhongmc on 2017/7/7.
 * 前置增强
 */

public class BeforeAdvice implements MethodBeforeAdvice {
    /**
     *
     * @param method 代理的方法
     * @param args 代理方法的参数列表
     * @param target 代理的目标类
     * @throws Throwable
     */
    public void before(Method method, Object[] args, Object target) throws Throwable {
        System.out.println("method1 execute before...");
    }
}
```
前置增强的逻辑比较简单，直接输出日志方便查看，测试类：
```java
/**
 * Created by zhongmc on 2017/7/7.
 */
public class Main {
   public static void main(String[] args) throws Exception {
        ProxyFactory proxyFactory = new ProxyFactory();
        proxyFactory.setTarget(new BookServiceImpl());
        proxyFactory.setInterfaces(CountService.class);
        proxyFactory.setProxyTargetClass(true);
        proxyFactory.addAdvice(new BeforeAdvice());
        BookServiceImpl proxy = (BookServiceImpl)proxyFactory.getProxy();
        Book book = proxy.addBook(new Book("1","平凡的世界","12001011-21200-1221133"));
    }
}
```
同时在编写一个后置增强类：
```java
public class CAfterReturningAdvice implements AfterReturningAdvice {
    /**
     *
     * @param o
     * @param method
     * @param objects
     * @param o1
     * @throws Throwable
     */
    public void afterReturning(Object o, Method method, Object[] objects, Object o1) throws Throwable {
        System.out.println("method1 execute after...");
    }
}
```
在测试方法中加入proxyFactory.addAdvice(new CAfterReturningAdvice())即可，得到测试结果：   
method1 execute before...   
add book : Book{id='1', name='平凡的世界', isbn='12001011-21200-1221133'}   
method1 execute after...   
环绕增强可以有两种实现方法，第一种同时实现MethodBeforeAdvice和AfterReturningAdvice接口，并重写其中
方法，但是为了方便起见直接实现MethodInterceptor接口即可，注意这个接口是org.aopalliance.intercept.MethodInterceptor
中的接口，是AOP联盟定义的一个接口。实现一个环绕增强：
```java
/**
 * Created by zhongmc on 2017/7/7.
 * 前后置增强 环绕增强
 */
public class AroundAdvice implements MethodInterceptor {
    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        before();
        Object result = methodInvocation.proceed();
        after();
        return result;
    }
    //方法调用前横切的逻辑
    public void before(){
        System.out.println("method execute before");
    }
    //方法调用后结果返回前调用的逻辑
    public void after(){
        System.out.println("method execute after");
    }
}
```
同时在测试类中加入proxyFactory.addAdvice(new AroundAdvice())，测试结果如下：    
method execute before    
method1 execute before...    
add book : Book{id='1', name='平凡的世界', isbn='12001011-21200-1221133'}    
method1 execute after...    
method execute after    
### 前置、后置、环绕增强（配置式）
配置式主要是写bean.xml配置文件，配置文件如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:context="http://www.springframework.org/schema/context"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
	   http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
	http://www.springframework.org/schema/context
	http://www.springframework.org/schema/context/spring-context-4.0.xsd">
	<context:component-scan base-package="com.zmc"/>

	<bean id="bookServiceProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
		<!-- 代理的目标类 -->
		<property name="target" ref="bookService"/>
		<!-- interceptorNames接收一个string数组，可以配置多个增强 -->
		<!--property name="interceptorNames" value="aroundAdvice"/>-->
		<property name="interceptorNames">
			<list>
				<value>aroundAdvice</value>
				<value>beforeAdvice</value>
				<value>cAfterReturningAdvice</value>
			</list>
		</property>

		<!-- 代理目标类在未继承接口的情况下，必须设置为proxyTargetClass=true
		 	就是说代理类，代理类就用的是cglib动态代理，默认为false，也就是代理
		 	接口，用的是jdk动态代理
		 -->
		<property name="proxyTargetClass" value="true"/>
	</bean>

</beans>
```
配置文件中运用了包扫描，切记要在BookServiceImpl类上加入@Service注解，并且在增强类上加入@Component
注解，这样spring在启动的时候就可以扫描到这些bean
测试类：
```java
/**
 * Created by zhongmc on 2017/7/7.
 */
public class TestBasedOnXmlConfig {
    public static void main(String[] args) throws Exception {
        ApplicationContext context =
                new ClassPathXmlApplicationContext("classpath:spring-bean.xml");
        //得到代理后的对象（增强后的对象）
        BookServiceImpl bookServiceProxy = (BookServiceImpl)context.getBean("bookServiceProxy");
        bookServiceProxy.addBook(new Book("1","平凡的世界","12001011-21200-1221133"));
    }
}
```
最终的测试结果和硬编码的方式一致
