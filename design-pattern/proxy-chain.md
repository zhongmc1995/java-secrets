## 基于CGlib的链式代理
链式代理，在我理解看来就是，多个代理类对同一个目标对象进行代理操作。在实际的业务中，往往要实现的需求复杂，
就要实现对同一个类进行不同的横切(AOP中称为切面),但是在CGlib中不能嵌套代理，也就是说，不能在对一个是代理
产生的类进行代理。
## 嵌套代理测试
```java
/**
 * Created by zhongmc on 2017/5/23.
 */
public class CGlibTest {
    public static <T> T createProxy(final Class<?> targetClass){
        return (T) Enhancer.create(targetClass, new MethodInterceptor() {

            public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                System.out.println("hello");
                Object res = methodProxy.invokeSuper(o,objects);
                System.out.println("after");
                return  res;
            }
        });
    }
    public static void main(String[] args) {
        BookService proxy = createProxy(BookService.class);
        BookService proxy1 = createProxy(proxy.getClass());
        proxy1.addBook("java");
    }
}
```
这样编译虽然能通过，但是在运行是就会报错：
Exception in thread "main" net.sf.cglib.core.CodeGenerationException: java.lang.reflect.InvocationTargetException-->null
这个错误就是因为CGlib不能嵌套代理所导致的
## 链式代理实现对同一对象进行代理
示例代码：
先定义一个接口：
```java
/**
 * Created by zhongmc on 2017/5/22.
 * 代理接口
 */
public interface Proxy {
    Object doProxy(ProxyChain proxyChain) throws Throwable;
}
```
定义一个该接口的抽象类
```java

/**
 * Created by zhongmc on 2017/5/22.
 * 切面抽象类
 */
public abstract class AspectProxy implements Proxy {
    private static final Logger LOGGER = LoggerFactory.getLogger(AspectProxy.class);

    /**
     * 重写doProxy方法
     * @param proxyChain
     * @return
     * @throws Throwable
     */
    public Object doProxy(ProxyChain proxyChain) throws Throwable {
        Class targetClass = proxyChain.getTargetClass();
        Method targetMethod = proxyChain.getTargetMethod();
        Object[] params = proxyChain.getParams();
        begin();
        Object result = null;
        try {

            if (intercept(targetClass,targetMethod,params)){
                before(targetClass,targetMethod,params);
                result = proxyChain.doProxyChain();
                after(targetClass,targetMethod,params,result);
            }else {
                result = proxyChain.doProxyChain();
            }

        }catch (Exception e){
            LOGGER.error("proxy failure" ,e);
            error(targetClass,targetMethod,params,e);
        }finally {
            end();
        }
        return result;
    }

    /**
     * 代理开始
     */
    public void begin(){

    }

    /**
     * 拦截
     * @param cls
     * @param method
     * @param params
     * @return
     * @throws Throwable
     */
    public boolean intercept(Class<?> cls, Method method,Object[] params) throws Throwable{
        return true;
    }

    /**
     * 前置增强方法
     * @param cls
     * @param method
     * @param params
     * @throws Throwable
     */
    public void before(Class<?> cls, Method method,Object[] params) throws Throwable{

    }

    /**
     * 后置增强方法
     * @param cls
     * @param method
     * @param params
     * @throws Throwable
     */
    public void after(Class<?> cls, Method method,Object[] params,Object result) throws Throwable{

    }

    /**
     * 代理异常处理方法
     * @param cls
     * @param method
     * @param params
     * @param throwable
     */
    public void error(Class<?> cls, Method method,Object[] params,Throwable throwable){

    }

    /**
     * 代理结束
     */
    public void end(){

    }
}
```
这个抽象类就是需要用户实现的切面

然后定义一个代理链
```java
/**
 * Created by zhongmc on 2017/5/22.
 * 代理链
 */
public class ProxyChain {
    private final static Logger LOGGER = LoggerFactory.getLogger(ProxyChain.class);

    private Class<?> targetClass;
    private Object target;
    private Method targetMethod;
    private Object[] params;
    private MethodProxy methodProxy;
    private List<Proxy> proxyList = new ArrayList<Proxy>();
    private Object methodResult;

    private int cursor = 0;

    public Class getTargetClass() {
        return targetClass;
    }

    public void setTargetClass(Class targetClass) {
        this.targetClass = targetClass;
    }

    public Object getTarget() {
        return target;
    }

    public void setTarget(Object target) {
        this.target = target;
    }

    public Method getTargetMethod() {
        return targetMethod;
    }

    public void setTargetMethod(Method targetMethod) {
        this.targetMethod = targetMethod;
    }

    public Object[] getParams() {
        return params;
    }

    public void setParams(Object[] params) {
        this.params = params;
    }

    public MethodProxy getMethodProxy() {
        return methodProxy;
    }

    public void setMethodProxy(MethodProxy methodProxy) {
        this.methodProxy = methodProxy;
    }

    public List<Proxy> getProxyList() {
        return proxyList;
    }

    public void setProxyList(List<Proxy> proxyList) {
        this.proxyList = proxyList;
    }

    public void setMethodResult(Object methodResult) {
        this.methodResult = methodResult;
    }

    public Object getMethodResult() {
        return methodResult;
    }

    /**
     * 构造器
     * @param targetClass
     * @param target
     * @param targetMethod
     * @param methodProxy
     * @param proxyList
     */
    public ProxyChain(Class<?> targetClass, Object target, Method targetMethod, Object[] params,MethodProxy methodProxy, List<Proxy> proxyList) {
        this.targetClass = targetClass;
        this.target = target;
        this.targetMethod = targetMethod;
        this.params = params;
        this.methodProxy = methodProxy;
        this.proxyList = proxyList;
        //LOGGER.info("target classname is "+targetClass.getName()+" , target instance is "+target);
    }

    public Object doProxyChain() throws Throwable{
        int proxyChainLen = proxyList.size();
        LOGGER.info("the length of proxy chain is "+proxyChainLen);
        Object result;
        if (cursor<proxyChainLen){
            Proxy proxy = proxyList.get(cursor++);
            result = proxy.doProxy(this);
        }else{
            result = methodProxy.invokeSuper(target, params);
        }
        return result;
    }
}
```
上面这段代码是链式代理的核心
最后就是定义一个代理工厂：
```java

/**
 * Created by zhongmc on 2017/5/22.
 * 代理管理器
 */
public class ProxyManager {
    private final static Logger LOGGER = LoggerFactory.getLogger(ProxyManager.class);
    public static <T> T createProxy(final Class<?> targetClass, final List<Proxy> proxyList){
        LOGGER.info("create "+targetClass.getName()+" proxy");
        return (T)Enhancer.create(targetClass, new MethodInterceptor() {
            public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                ProxyChain proxyChain = new ProxyChain(targetClass,o,method, objects,methodProxy,proxyList);
                return proxyChain.doProxyChain();
            }
        });
    }
}
```
通过一个匿名内部类来实现 CGLib 的 MethodInterceptor 接口，并填充 intercept() 方法。将该方法的所有入口参数都传递到创建的 ProxyChain 对象中，外加该类的两个成员变量：targetClass 与 proxyList。然后调用 ProxyChain 的 doProxyChain() 方法，可以想象，调用是一连串的，当调用完毕后，可直接获取方法返回值。
这样就能实现链式代理了
## 测试
先定义两个切面
```java
/**
 * Created by zhongmc on 2017/5/22.
 */
public class ControllerAfterAspect extends AspectProxy {
    private final static Logger LOGGER = LoggerFactory.getLogger(ControllerAfterAspect.class);

    @Override
    public void before(Class<?> cls, Method method, Object[] params) throws Throwable {
        System.out.println("ControllerAfterAspect before");
    }
    @Override
    public void after(Class<?> cls, Method method, Object[] params,Object result) throws Throwable {
        System.out.println("ControllerAfterAspect after");
    }
}
```

```java
/**
 * Created by zhongmc on 2017/5/22.
 */
public class ControllerBeforeAspect extends AspectProxy {
    private final static Logger LOGGER = LoggerFactory.getLogger(ControllerBeforeAspect.class);
    @Override
    public void before(Class<?> cls, Method method, Object[] params) throws Throwable {
        System.out.println("ControllerBeforeAspect before");
    }

    @Override
    public void after(Class<?> cls, Method method, Object[] params,Object result) throws Throwable {
        System.out.println("ControllerBeforeAspect after");
    }
}
```
测试代码：
```java
/**
 * Created by zhongmc on 2017/5/23.
 */
public class AOPTest {
    public static void main(String[] args) {
        List<Proxy> proxies = new ArrayList<Proxy>();
        proxies.add(new ControllerBeforeAspect());
        proxies.add(new ControllerAfterAspect());
        BookService proxy = ProxyManager.createProxy(BookService.class, proxies);
        String res = proxy.addBook("Java");
        System.out.println(res);
    }
}
```
## 结果
   1. ControllerBeforeAspect before
   2. ControllerAfterAspect before
   3. add java book
   4. ControllerAfterAspect after
   5. ControllerBeforeAspect after
   6. success

**注**:参考：https://my.oschina.net/huangyong/blog/170494

