## SpringMVC源码分析 -- DispatcherServlet
## 类继承关系
DispatcherServlet <-- FrameworkServlet <-- HttpServletBean <-- HttpServlet <-- GenericServlet <-- Servlet(接口)
源码分析从最底层的类(接口)开始分析，我们都知道一个servlet容器初始化在init()方法中，发送一个request请求
在servlet中的service()方法中进行处理，顺着这个过程来对DispatcherServlet源码进行分析。
## Servlet
servlet接口是web开发的基石，其中的init()和service()方法是接下来要重点分析的对象。
## GenericServlet
从这个类的命名可以看出，这个通用servlet的实现，这个类实现了接口servlet的大部分接口，我们重点分析init()和
service()方法，
```java
public void init(ServletConfig config) throws ServletException {
    this.config = config;
    this.init();
}

public void init() throws ServletException {
}
public abstract void service(ServletRequest var1, ServletResponse var2) 
        throws ServletException, IOException;
```
init()方法设置了ServletConfig之后就什么都没有做，接下来就是定义了一个模板方法init()给子类实现的
service()方法定义为了抽象方法，子类必须实现。
## HttpServlet
HttpServlet主要是实现了父类中的service()方法，并且将请求类型的不同，定义了处理不同请求的方法，doHead(),doPost(),deGet()等等
这些方法将service()方法的工作进行了抽象，非常符合单一职责原则，通过service()中接受到的request的类型不同，将请求
分发到对应请求类型的方法中做处理，service()方法代码如下：
```java
protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    String method = req.getMethod();
    long errMsg;
    //判断请求的类型
    if(method.equals("GET")) {
        //判断请求的资源是否被修改过
        errMsg = this.getLastModified(req);
        if(errMsg == -1L) {
            this.doGet(req, resp);
        } else {
            long ifModifiedSince = req.getDateHeader("If-Modified-Since");
            if(ifModifiedSince < errMsg) {
                this.maybeSetLastModified(resp, errMsg);
                this.doGet(req, resp);
            } else {
                //未被修改，直接返回304
                resp.setStatus(304);
            }
        }
    } else if(method.equals("HEAD")) {
        errMsg = this.getLastModified(req);
        this.maybeSetLastModified(resp, errMsg);
        this.doHead(req, resp);
    } else if(method.equals("POST")) {
        this.doPost(req, resp);
    } else if(method.equals("PUT")) {
        this.doPut(req, resp);
    } else if(method.equals("DELETE")) {
        this.doDelete(req, resp);
    } else if(method.equals("OPTIONS")) {
        this.doOptions(req, resp);
    } else if(method.equals("TRACE")) {
        this.doTrace(req, resp);
    } else {
        //未找到
        String errMsg1 = lStrings.getString("http.method_not_implemented");
        Object[] errArgs = new Object[]{method};
        errMsg1 = MessageFormat.format(errMsg1, errArgs);
        resp.sendError(501, errMsg1);
    }

}
```
上面的代码处理get请求，剩下的都是直接分发到对应的方法中进行处理，get请求方法判断了请求的资源是否被修改过，
如果未被修改则返回304，资源从浏览器缓存中获取，如果被修改过，则在doGet()方法中处理。
## HttpServletBean
从这个类开始才是真正springmvc定义的类了，这个类实现了GenericServlet类中的模板方法init()
```java
public final void init() throws ServletException {
    if(this.logger.isDebugEnabled()) {
        this.logger.debug("Initializing servlet \'" + this.getServletName() + "\'");
    }

    try {
        HttpServletBean.ServletConfigPropertyValues ex = new HttpServletBean.ServletConfigPropertyValues(this.getServletConfig(), this.requiredProperties);
        BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
        ServletContextResourceLoader resourceLoader = new ServletContextResourceLoader(this.getServletContext());
        bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, this.getEnvironment()));
        this.initBeanWrapper(bw);
        bw.setPropertyValues(ex, true);
    } catch (BeansException var4) {
        if(this.logger.isErrorEnabled()) {
            this.logger.error("Failed to set bean properties on servlet \'" + this.getServletName() + "\'", var4);
        }

        throw var4;
    }

    this.initServletBean();
    if(this.logger.isDebugEnabled()) {
        this.logger.debug("Servlet \'" + this.getServletName() + "\' configured successfully");
    }

}

protected void initBeanWrapper(BeanWrapper bw) throws BeansException {
}

protected void initServletBean() throws ServletException {
}
```
从这个类中可看出，init()方法中主要是对自己进行了设值，同时也定义了两个模板方法initBeanWrapper()和initServletBean()
供子类继承使用。
## FrameworkServlet
这个类重写了HttpServletBean中的和initServletBean()模板方法，代码：
```java
protected final void initServletBean() throws ServletException {
    this.getServletContext().log("Initializing Spring FrameworkServlet \'" + this.getServletName() + "\'");
    if(this.logger.isInfoEnabled()) {
        this.logger.info("FrameworkServlet \'" + this.getServletName() + "\': initialization started");
    }

    long startTime = System.currentTimeMillis();
    try {
        //1、初始化Web上下文
        this.webApplicationContext = this.initWebApplicationContext();
        //模板方法
        this.initFrameworkServlet();
    } catch (ServletException var5) {
        this.logger.error("Context initialization failed", var5);
        throw var5;
    } catch (RuntimeException var6) {
        this.logger.error("Context initialization failed", var6);
        throw var6;
    }

    if(this.logger.isInfoEnabled()) {
        long elapsedTime = System.currentTimeMillis() - startTime;
        this.logger.info("FrameworkServlet \'" + this.getServletName() + "\': initialization completed in " + elapsedTime + " ms");
    }

}
protected WebApplicationContext initWebApplicationContext() {
    //查找ROOT上下文（ContextLoaderListener加载的，不如通常用的Spring上下文）
    WebApplicationContext rootContext = WebApplicationContextUtils.getWebApplicationContext(this.getServletContext());
    WebApplicationContext wac = null;
    if(this.webApplicationContext != null) {
        wac = this.webApplicationContext;
        if(wac instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext attrName = (ConfigurableWebApplicationContext)wac;
            if(!attrName.isActive()) {
                if(attrName.getParent() == null) {
                    attrName.setParent(rootContext);
                }

                this.configureAndRefreshWebApplicationContext(attrName);
            }
        }
    }

    if(wac == null) {
        //如果上不操作为找到上下文，则查找已经绑定的上下文
        wac = this.findWebApplicationContext();
    }

    if(wac == null) {
        //如果还是为绑定上下文则直接指定ContextLoaderListener加载的上下文
        wac = this.createWebApplicationContext(rootContext);
    }
    //上下文刷新
    if(!this.refreshEventReceived) {
        //模板方法
        this.onRefresh(wac);
    }

    if(this.publishContext) {
        String attrName1 = this.getServletContextAttributeName();
        this.getServletContext().setAttribute(attrName1, wac);
        if(this.logger.isDebugEnabled()) {
            this.logger.debug("Published WebApplicationContext of servlet \'" + this.getServletName() + "\' as ServletContext attribute with name [" + attrName1 + "]");
        }
    }

    return wac;
}
```
这个方法中主要是查找并设置父容器，从initWebApplicationContext（）方法可以看出，基本上如果ContextLoaderListener加载了上下文将作为根上下文（DispatcherServlet的父容器）。
最后调用了onRefresh()方法执行容器的一些初始化，这个方法由子类实现，来进行扩展。
除此之位这个类还重写了HttpServlet中的service()和do**()方法，do**()方法全部都给了processRequest()进行处理。
```java
protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
    HttpMethod httpMethod = HttpMethod.resolve(request.getMethod());
    if(HttpMethod.PATCH != httpMethod && httpMethod != null) {
        super.service(request, response);
    } else {
        this.processRequest(request, response);
    }

}

protected final void processRequest(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
  //... 
  this.doService(request, response); 
  //...
}
```
processRequest代码中的其他代码可以先忽略，重点的就是模板方法doService(),这个方法是DispatcherServlet中主要的处理
方法。
## DispatcherServlet
这个类主要用作职责调度工作，本身主要用于控制流程，主要职责如下：
* 文件上传解析，如果请求类型是multipart将通过MultipartResolver进行文件上传解析；
* 通过HandlerMapping，将请求映射到处理器（返回一个HandlerExecutionChain，它包括一个处理器、多个HandlerInterceptor拦截器）；
* 通过HandlerAdapter支持多种类型的处理器(HandlerExecutionChain中的处理器)；
* 通过ViewResolver解析逻辑视图名到具体视图实现；
* 本地化解析；
* 渲染具体的视图等；
* 如果执行过程中遇到异常将交给HandlerExceptionResolver来解析。
重要的方法如下：
```java
//重写父类的模板方法
protected void onRefresh(ApplicationContext context) {
    this.initStrategies(context);
}

protected void initStrategies(ApplicationContext context) {
    //初始化各个组件
    this.initMultipartResolver(context);
    this.initLocaleResolver(context);
    this.initThemeResolver(context);
    this.initHandlerMappings(context);
    this.initHandlerAdapters(context);
    this.initHandlerExceptionResolvers(context);
    this.initRequestToViewNameTranslator(context);
    this.initViewResolvers(context);
    this.initFlashMapManager(context);
}

protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
    //...
    //设置request属性
    request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.getWebApplicationContext());
    request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
    request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
    request.setAttribute(THEME_SOURCE_ATTRIBUTE, this.getThemeSource());
    FlashMap inputFlashMap1 = this.flashMapManager.retrieveAndUpdate(request, response);
    if(inputFlashMap1 != null) {
        request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap1));
    }

    request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
    request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);

    try {
        //请求分发
        this.doDispatch(request, response);
    } finally {
        if(!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted() && attributesSnapshot1 != null) {
            this.restoreAttributesAfterInclude(request, attributesSnapshot1);
        }

    }

}
//这个方法，通过request找到控制器，做处理并返回结果
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        try {
            ModelAndView err = null;
            Object dispatchException = null;

            try {
                //检查request是否有文件上传
                processedRequest = this.checkMultipart(request);
                multipartRequestParsed = processedRequest != request;
                //通过HandlerMapping，将请求映射到处理器（返回一个HandlerExecutionChain，
                //	它包括一个处理器、多个HandlerInterceptor拦截器）；
                mappedHandler = this.getHandler(processedRequest);
                if(mappedHandler == null || mappedHandler.getHandler() == null) {
                    //没有找到处理器，抛出异常，或着返回404
                    this.noHandlerFound(processedRequest, response);
                    return;
                }
                //通过处理器来获取处理器适配器，处理器适配器可以支持多种类型的处理器
                HandlerAdapter err1 = this.getHandlerAdapter(mappedHandler.getHandler());
                String method = request.getMethod();
                boolean isGet = "GET".equals(method);
                if(isGet || "HEAD".equals(method)) {
                    //get和head请求检查请求资源是否改变
                    long lastModified = err1.getLastModified(request, mappedHandler.getHandler());
                    if(this.logger.isDebugEnabled()) {
                        this.logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
                    }
                    
                    if((new ServletWebRequest(request, response)).checkNotModified(lastModified) && isGet) {
                        return;
                    }
                }
                //HandlerInterceptor执行afterCompletion方法
                if(!mappedHandler.applyPreHandle(processedRequest, response)) {
                    return;
                }
                //处理完，返回到视图
                err = err1.handle(processedRequest, response, mappedHandler.getHandler());
                if(asyncManager.isConcurrentHandlingStarted()) {
                    return;
                }

                this.applyDefaultViewName(processedRequest, err);
                //HandlerInterceptor执行postHandle方法
                mappedHandler.applyPostHandle(processedRequest, response, err);
            } catch (Exception var20) {
                dispatchException = var20;
            } catch (Throwable var21) {
                dispatchException = new NestedServletException("Handler dispatch failed", var21);
            }

            this.processDispatchResult(processedRequest, response, mappedHandler, err, (Exception)dispatchException);
        } catch (Exception var22) {
            this.triggerAfterCompletion(processedRequest, response, mappedHandler, var22);
        } catch (Throwable var23) {
            this.triggerAfterCompletion(processedRequest, response, mappedHandler, new NestedServletException("Handler processing failed", var23));
        }

    } finally {
        if(asyncManager.isConcurrentHandlingStarted()) {
            if(mappedHandler != null) {
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        } else if(multipartRequestParsed) {
            this.cleanupMultipart(processedRequest);
        }

    }
}
```
从上面方法中可以看出，这个方法可以简单的认为主要的工作是初始化组件，并通过request找到处理器进行处理并返回结果，抛出异常后将交给HandlerExceptionResolver来解析处理，
方法中的大量代码都是在做准备工作，在初始化组件的过程中DispatcherServlet中使用了许多特殊的Bean，这些默认的Bean
在DispatcherServlet.properties都做了配置，Bean如下：
* Controller：处理器/页面控制器，做的是MVC中的C的事情，但控制逻辑转移到前端控制器了，用于对请求进行处理；
* HandlerMapping：请求到处理器的映射，如果映射成功返回一个HandlerExecutionChain对象（包含一个Handler处理器（页面控制器）对象、多个HandlerInterceptor拦截器）对象；如BeanNameUrlHandlerMapping将URL与Bean名字映射，映射成功的Bean就是此处的处理器；
* HandlerAdapter：HandlerAdapter将会把处理器包装为适配器，从而支持多种类型的处理器，即适配器设计模式的应用，从而很容易支持很多类型的处理器；如SimpleControllerHandlerAdapter将对实现了Controller接口的Bean进行适配，并且掉处理器的handleRequest方法进行功能处理；
* ViewResolver：ViewResolver将把逻辑视图名解析为具体的View，通过这种策略模式，很容易更换其他视图技术；如InternalResourceViewResolver将逻辑视图名映射为jsp视图；
* LocalResover：本地化解析，因为Spring支持国际化，因此LocalResover解析客户端的Locale信息从而方便进行国际化；
* ThemeResovler：主题解析，通过它来实现一个页面多套风格，即常见的类似于软件皮肤效果；
* MultipartResolver：文件上传解析，用于支持文件上传；
* HandlerExceptionResolver：处理器异常解析，可以将异常映射到相应的统一错误界面，从而显示用户友好的界面（而不是给用户看到具体的错误信息）；
* RequestToViewNameTranslator：当处理器没有返回逻辑视图名等相关信息时，自动将请求URL映射为逻辑视图名；
* FlashMapManager：用于管理FlashMap的策略接口，FlashMap用于存储一个请求的输出，当进入另一个请求时作为该请求的输入，通常用于重定向场景。
