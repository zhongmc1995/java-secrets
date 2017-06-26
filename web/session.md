## 前言
最近在研究权限控制框架shiro，其中涉及到session相关的知识，发现自己对session的理解存在偏差，因此写下这篇文章来纪录
## session
Session 是用于保持状态的基于Web服务器的方法。Session 允许通过将对象存储在Web服务器的内存中在整个用户会话过程中保持任何对象。
在web项目中session被封装成HTTPSession接口，其中定义了操作和获取session的方法，常用的有： 
* public String getId(); 这个方法用来获取session的id
* public long getCreationTime(); 获取session创建的时间
* public ServletContext getServletContext(); 获取servlet的上下文
* public void setMaxInactiveInterval(int interval); 设置session的过期时间，设置小于等于零表示永远不会过期
* public void setAttribute(String name, Object value); 给session设置数据，name/value
* public Object getValue(String name); 获取key为name的存储在session中的value
## HTTPSession的实现原理和个人理解
在解释原理之前，先打个简单的比方，方便理解，日常生活中当我去银行办理业务的时候，在首次去银行时，因为还没有账号，所以需要开一个账号，然后获得银行卡，而银行这边的数据库中留下了我的账号，我的钱是保存在银行的账号中，而我带走的是我的卡号。当我再次去银行时，只需要带上我的卡，而无需再次开一个账号了。只要带上我的卡，那么正常情况下我在银行操作的一定是我的账号，在这个例子中银行就相当于服务器，而银行卡就相当于服务器给客户端的sessionId。
当首次使用session时，服务器端要创建session，session是保存在服务器端，而给客户端的session的id（一个cookie中保存了sessionId）。客户端带走的是sessionId，而数据是保存在session中。当客户端再次访问服务器时，在请求中会带上sessionId，这个sessionId保存在cookies中，而服务器会通过sessionId找到对应的session，而无需再创建新的session，这整个过程都是依赖cookies的，如果在浏览器cookie被禁用的情况下，这个过程是无法工作的，这个时候需要靠其他的手段达到这个效果。
## session和浏览器
浏览器中在cookie存的是sessionId，浏览器至始至终都没有存session对象，session对象一般是存在服务端的内存中，在session
做了持久化的时候可能在数据库中，浏览器和服务端两者是通过每个request中的cookie中存的sessionId来对用户的状态进行判断的
我们可以通过session中的方法来设置过期时间，不设置的情况下默认是30分钟，通常可以在web.xml中设置时常
```xml
    <session-config>
        <session-timeout>30</session-timeout>
    </session-config>
```
## 注意
创建session的情况：
* 访问jsp页面，session会被创建，在jsp被编译成servlet是默认是注入了session
* servlet中手动操作了session，session会被创建
* 同时注意访问静态资源是不存在session这个概念的，因此也不存在session创不创建的问题


