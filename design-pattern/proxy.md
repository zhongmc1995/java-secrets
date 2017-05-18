# 代理模式
代理，或者称Proxy，通俗的讲就是，自己不做事情，让别人代替你做。在设计模式中
代理分为两种，静态代理和动态代理，其中在Java中，动态代理又分为JDK代理和第三方库
CGlib代理。面向切面编程(AOP)是代理模式代表之作,Spring AOP中大量的使用了代理模式的思想。
## 准备工作
定义一个接口IBookService，其中有一个添加的方法。
```java
/**
 * Created by zhongmc on 2017/5/17.
 */
public interface IBookService {
    String addBook(String name);
}
```
在定义一个其的实现类
```java
/**
 * Created by zhongmc on 2017/5/17.
 */
public class BookService implements IBookService {
    public String addBook(String name){
        System.out.println("add "+name+" book");
        return "success";
    }
}
```
目前的需求就是在BookService执行添加图书的方式前后做相应的操作。
## 静态代理
静态代理指的是在编码阶段的行为。
## 实例代码
```java
/**
 * Created by zhongmc on 2017/5/18.
 * 静态代理
 */
public class BookServiceProxyByStatic implements IBookService {
    //代理类的目标对象
    private IBookService bookService;

    public BookServiceProxyByStatic(IBookService bookService){
        this.bookService = bookService;
    }

    public void before(){
        System.out.println("execute method before");
    }

    public void after(){
        System.out.println("execute method after");
    }

    @Override
    public String addBook(String name) {
        before();
        String result = bookService.addBook(name);
        after();
        return result;
    }
    //测试
    public static void main(String[] args) {
        BookServiceProxyByStatic proxy = new BookServiceProxyByStatic(new BookService());
        proxy.addBook("Java");
    }
}
```
**注**：这种纯粹靠硬编码实现的代理存在着明显的弊端，就是灵活性不
够强同时必须还要实现统一的接口，设想有很多类都需要实现相同的操作，那么要编写的代理也要很多。
## JDK代理
在Java中基础类库中自带代理操作的接口，我们只需要实现其接口就可以其代理。
## 示例代码
```java
/**
 * Created by zhongmc on 2017/5/18.
 * JDK动态代理
 */
public class BookServiceProxyByJDK implements InvocationHandler {
    //代理目标对象
    private Object target;
    public BookServiceProxyByJDK(Object target){
        this.target = target;
    }

    public void before(){
        System.out.println("execute method before");
    }

    public void after(){
        System.out.println("execute method after");
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        before();
        Object result = method.invoke(target, args);
        after();
        return result;
    }

    public <T> T getProxy(){
        return (T)Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(),
                this);
    }

    //测试
    public static void main(String[] args) {
        BookServiceProxyByJDK proxy = new BookServiceProxyByJDK(new BookService());
        IBookService bookService = proxy.getProxy();//这里必须要用接口去接收
        bookService.addBook("java");
    }
}
```
**注**：JDK动态代理的类也必须实现统一的接口，在某些没有接口的情况下就无法实现代理了。这种情况下就必须使用CGlib实现代理了。
## CGlib代理
JDK实现动态代理需要实现类通过接口定义业务方法，对于没有接口的类，如何实现动态代理呢，这就需要CGLib了。CGLib	采用了非常底层的字节码技术，其原理是通过字节码技术为目标对象创建一个子类对象，并在子类对象中拦截所有父类方法的调用，然后在方法调用前后调用后都可以加入自己想要执行的代码。
需要这种方法只是需要俩个第三方jar包: cglib-x.x.x.jar和asm-x.x.x.jar
同时很多框架已经把这些jar包整合到一起了，比如spring框架的spring-core-x.x.x.RELEASE.jar,这一个jar包就包括上述俩个jar包的大多数功能
## 示例代码
```java
/**
 * Created by zhongmc on 2017/5/18.
 * CGlib代理测试
 */
public class BookServiceProxyByCGlib implements MethodInterceptor {

    private final static BookServiceProxyByCGlib INSTANCE = new BookServiceProxyByCGlib();

    public static BookServiceProxyByCGlib getInstance(){
        return INSTANCE;
    }

    //获取要代理的对象
    public <T> T getProxy(Class<T> cls){
        return (T) Enhancer.create(cls,this);
    }

    public void before(){
        System.out.println("execute method before");
    }

    public void after(){
        System.out.println("execute method after");
    }

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        before();
        //被代理对象执行的方法
        Object result = methodProxy.invokeSuper(o, objects);
        after();
        return result;
    }
    //测试
    public static void main(String[] args) {
        /*BookServiceProxyByCGlib proxy = new BookServiceProxyByCGlib();
        BookService b = proxy.getProxy(BookService.class);
        b.addBook("Java");*/

        BookService b = BookServiceProxyByCGlib.getInstance().getProxy(BookService.class);
        b.addBook("Java");

    }
}
```
**注**：为了避免反复的new BookServiceProxyByCGlib对象，这个运用了单列模式。CGLib代理可以代理任意一个类，这种方式灵活，适用大部分场景。




