# Java Web中的Listener

listener顾名思义就是监听的意思，作用就是监听程序中的一些变化，并根据其做出一些相应的响应。

### Servlet中的Listener

#### ServletContextListener

用来监听容器的启动和销毁，包含contextInitialized和contextDestroyed方法，都是对ServletContextEvent事件的监听，分别对应容器启动时，和容器销毁时

```java
public class MyListenerListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent servletContextEvent) {
        System.out.println("init");
    }

    @Override
    public void contextDestroyed(ServletContextEvent servletContextEvent) {
        System.out.println("destroyed");
    }
}
```

ServletContextEvent可以拿到ServletContext对象。

#### ServletContextAttributeListener 

用于监听ServletContext(application)范围内属性的变化。当application范围内的属性被添加删除，替换时，这些对应的监听器将会被触发。

```java
public class MyListenerListener implements ServletContextAttributeListener {

    @Override
    public void attributeAdded(ServletContextAttributeEvent servletContextAttributeEvent) {
        servletContextAttributeEvent.getName();
        servletContextAttributeEvent.getValue();

    }

    @Override
    public void attributeRemoved(ServletContextAttributeEvent servletContextAttributeEvent) {

    }

    @Override
    public void attributeReplaced(ServletContextAttributeEvent servletContextAttributeEvent) {

    }
}
```

ServletContextAttributeEvent可以拿到变化的属性名和值。



#### ServletRequestListener

用于监听request对象的创建和销毁

```java
public class MyListenerListener implements ServletRequestListener {

    @Override
    public void requestDestroyed(ServletRequestEvent servletRequestEvent) {
        servletRequestEvent.getServletContext();
        servletRequestEvent.getServletRequest();
        System.out.println("request对象销毁了");
    }

    @Override
    public void requestInitialized(ServletRequestEvent servletRequestEvent) {
        System.out.println("request对象创建了");
    }
}
```

ServletRequestEvent可以拿到ServletContext和Request对象。

#### ServletRequestAttributeListener 

用于监听servletRequest域属性更改的事件

```java
public class MyListenerListener implements ServletRequestAttributeListener  {
    @Override
    public void attributeAdded(ServletRequestAttributeEvent servletRequestAttributeEvent) {
        servletRequestAttributeEvent.getName();
        servletRequestAttributeEvent.getValue();
    }

    @Override
    public void attributeRemoved(ServletRequestAttributeEvent servletRequestAttributeEvent) {

    }

    @Override
    public void attributeReplaced(ServletRequestAttributeEvent servletRequestAttributeEvent) {

    }
}
```

ServletRequestAttributeEvent和ServletContext一样，可以拿到对应的键值对。

#### HttpSessionListener

用于监听当前应用中session的创建和销毁情况

```java
public class MyListenerListener implements HttpSessionListener {


    @Override
    public void sessionCreated(HttpSessionEvent httpSessionEvent) {
        System.out.println("session创建");
        httpSessionEvent.getSession();
    }

    @Override
    public void sessionDestroyed(HttpSessionEvent httpSessionEvent) {
        System.out.println("session销毁");
    }
}
```

两种情况下就会发生sessionDestoryed（会话销毁）事件：

1. 执行session.invalidate()方法时。
2. 用户长时间没有访问服务器，超过了会话最大超时时间，服务器就会自动销毁超时的session。

#### HttpSessionBindingListener

用于监听session对象绑定对象（setAttribute方法）的过程，valueBound（）和valueUnbound（）方法分别代表绑定和解绑。

```java
public class MyListenerListener implements HttpSessionBindingListener {
	private int uid;
	public int getUid() {
   		return uid;
	}
	public void setUid(int uid) {
   		this.uid = uid;
    @Override
    public void valueBound(HttpSessionBindingEvent httpSessionBindingEvent) {
        httpSessionBindingEvent.getName();
        httpSessionBindingEvent.getValue();
        httpSessionBindingEvent.getSession();
    }

    @Override
    public void valueUnbound(HttpSessionBindingEvent httpSessionBindingEvent) {

    }
}
```

如何使用？在相关业务环境下，执行下面类似操作，**不需要在web.xml指定**

```java
UsersOnlineCountListener uocl = new UsersOnlineCountListener();
uocl.setUid(obj.getUid());
session.setAttribute("uocl", uocl);//这个时候要触发valueBound方法了
```

valueBound由setAttribute触发。valueUnbound触发的情况由以下几种：

1. 执行session.setAttribute("uocl", 非uocl对象，有变化) 时。
2. 执行session.removeAttribute("uocl") 时。
3. 执行session.invalidate()时。
4. session超时后。

#### HttpSessionAttributeListener 

和request和context一样，监听session域的键值对变化。

```java
public class MyListenerListener implements HttpSessionAttributeListener {


    @Override
    public void attributeAdded(HttpSessionBindingEvent httpSessionBindingEvent) {
        httpSessionBindingEvent.getSession();
        httpSessionBindingEvent.getValue();
        httpSessionBindingEvent.getName();
    }

    @Override
    public void attributeRemoved(HttpSessionBindingEvent httpSessionBindingEvent) {

    }

    @Override
    public void attributeReplaced(HttpSessionBindingEvent httpSessionBindingEvent) {

    }
}
```



#### HttpSessionActivationListener

用于监控实现类本身，当实现类对象被添加到session中（session.setAttribute()）后，session对象序列化（钝化）前和反序列化（活化）后都会被执行，相应的方法。

实现此接口的JavaBean，可以感知自己被活化(从硬盘到内存)和钝化(从内存到硬盘)的过程。　

```java
public class Person implements Serializable, HttpSessionActivationListener {
private String name;

    public Person(String name) {
        super();
        this.name = name;
    }
    
    @Override
    public void sessionWillPassivate(HttpSessionEvent se) {
        System.out.println(this + "保存到硬盘了...");
    }
    
    @Override
    public void sessionDidActivate(HttpSessionEvent se) {
        System.out.println(this + "从硬盘读取并活化了...");
    }
    
    @Override
    public String toString() {
        return "Perosn [name=" + name + "]---"+super.toString();
    }
}
```

该监听器也**不需要在webx.ml中配置，必须配置到Tomcat服务器中**！



### Spring中的Listener

Spring的监听器是典型的事件驱动模型，里面主要涉及到三个接口ApplicationListener、ApplicationEventPublisher（ApplicationContext 实现了该接口的publishEvent，用于发布该事件） 、ApplicationEventMulticaster （主要负责事件的通知，开发时不用关心），一个抽象类ApplicationEvent，实现了EventObject接口。

可用spring内置的Event，也可自己实现Event接口。

#### Spring内置事件

*  **ContextRefreshedEvent 上下文更新事件：** 该事件会在ApplicationContext被初始化或者更新时发布。也可在调用ConfigurableApplicationContext接口中的refresh()方法时被触发。
*  **ContextStartedEvent 上下文开始事件：** 当容器调用ConfigurableApplicationContext接口中的start()方式触发。
*  **ContextStoppedEvent 上下文停止事件：** 当容器调用ConfigurableApplicationContext接口中的stop()方式触发。
*  **ContextClosedEvent 上下文关闭事件：** 当ApplicationContext被关闭时触发该事件。容器被关闭时，其所有管理的单例Bean被销毁。
*  **RequestHandledEvent 请求处理事件：** 在Web应用中，当一个http请求结束时触发该事件。

比如想在spring容器启动后做一些操作，就应做如下操作。

```java
public class InitListener implements ApplicationListener<ContextRefreshedEvent> {
    @Override
    public void onApplicationEvent(ContextRefreshedEvent contextRefreshedEvent) {
        if(event.getApplicationContext().getParent() == null){
            //业务逻辑写在里面，保证业务代码只执行一次
            
        }
    }
}
```

但这里有一个问题，InitListener的onApplicationEvent方法会被执行两次。在web项目中，系统会存在两个容器，一个是root application context，另一个就是我们自己的projectName-servlet context（root application context 的子容器）。

#### 自定义事件监听

##### 自定义Event

```java
public class MyEvent extends ApplicationEvent{
    
    private String name;
    private int id;

    public MyEvent(Object source) {
        super(source);
    }
    public MyEvent(Object source,int id,String name) {
        super(source);
        this.id = id;
        this.name = name; 
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }


    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }
}

```

##### 自定义Listener

```java
public class MyListener implements ApplicationListener<MyEvent> {
    @Override
    public void onApplicationEvent(MyEvent myEvent) {
        System.out.println("myEvent事件发生了");
    }
}
```

##### 实际触发事件的地方

```java
public class Main {

    public static void main(String[] args) {

        ApplicationContext context = new FileSystemXmlApplicationContext("classpath:spring.xml");
        MyEvent event = new MyEvent("事件源",1,"测试");
        context.publishEvent(event);
    }
}
```

