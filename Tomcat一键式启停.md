# Tomcat一键式启停



一个请求在Tomcat中的执行过程：

<img src="https://static001.geekbang.org/resource/image/12/9b/12ad9ddc3ff73e0aacf2276bcfafae9b.png" alt="img" style="zoom: 50%;" />

LifeCycle接口

第一，需要从父组件开始init和start，直至整个tomcat启动起来，然后关闭时从父组件开始调用stop和destroy，最终tomcat关闭，这是组合模式。因此LifeCycle存在这些方法：

​    <img src="https://static001.geekbang.org/resource/image/a1/5c/a1fcba6105f4235486bdba350d58bb5c.png" alt="img" style="zoom:33%;" />

第二，可扩展性，开闭原则是指不能修改已有的类，可以定义新的类。上面四个方法的调用时由父组件的状态变化触发的。因此把组件的生命周期定义成一个个状态，通过监听器调用这些方法，这是观察者模式。因此需要在接口LifeCycle中添加增加和删除监听器的方法 addLifeCycleListener removeLifeCycleListener以及增加状态类型：

<img src="https://static001.geekbang.org/resource/image/dd/c0/dd0ce38fdff06dcc6d40714f39fc4ec0.png" alt="img" style="zoom:33%;" />

第三，重用性。

<img src="https://static001.geekbang.org/resource/image/67/d9/6704bf8a3e10e1d4cfb35ba11e6de5d9.png" alt="img" style="zoom:33%;" />

LifeCycleBase实现了LifeCycle接口，并将生命管理的这四个方法交给继承他的子类来实现，而生命状态的转变与维护，生命事件的触发以及监听器的添加等放在基类中，**这个不太懂，基类是指LifeCycleBase吗？可以看看源码。**

第四，监听器的注册。Tomcat自定义的监听器在父组件创建子组件的时候注册到子组件中；自己可以在server.xml中创建监听器并注册到容器组件。

总结 生命周期管理类图

<img src="https://static001.geekbang.org/resource/image/de/90/de55ad3475e714acbf883713ee077690.png" alt="img" style="zoom:33%;" />

图中的 StandardServer、StandardService 等是 Server 和 Service 组件的具体实现类，它们都继承了 LifecycleBase。StandardEngine、StandardHost、StandardContext 和 StandardWrapper 是相应容器组件的具体实现类，因为它们都是容器，所以继承了 ContainerBase 抽象基类，而 ContainerBase 实现了 Container 接口，也继承了 LifecycleBase 类，它们的生命周期管理接口(LifeCycle)和功能接口(Container)是分开的，这也符合设计中接口分离的原则。



# Tomcat高层

## 1. Tomcat启动过程中，高层的启动

<img src="https://static001.geekbang.org/resource/image/57/4d/578edfe9c06856324084ee193243694d.png" alt="img" style="zoom: 67%;" />

​		1.Tomcat 本质上是一个 Java 程序，因此startup.sh脚本会启动一个 JVM 来运行 Tomcat 的启动类 Bootstrap。

​		2.Bootstrap 的主要任务是**初始化 Tomcat 的类加载器**，并且创建 Catalina。关于 Tomcat 为什么需要自己的类加载器，我会在专栏后面详细介绍。

​		3.Catalina 是一个启动类，它通过解析server.xml、创建相应的组件，并调用 Server 的 start 方法。

​		4.Server 组件的职责就是管理 Service 组件，它会负责调用 Service 的 start 方法。

​		5.Service 组件的职责就是管理连接器和顶层容器 Engine，因此它会调用连接器和 Engine 的 start 方法。

## 2. Catalina

​	Catalina 的主要任务就是创建 Server，它不是直接 new 一个 Server 实例就完事了，而是需要解析server.xml，把在server.xml里配置的各种组件一一创建出来，接着调用 Server 组件的 init 方法和 start 方法，这样整个 Tomcat 就启动起来了。作为“管理者”，Catalina 还需要处理各种“异常”情况，比如当我们通过“Ctrl  +  C”关闭 Tomcat 时，Tomcat 将如何优雅的停止并且清理资源呢？因此 Catalina 在 JVM 中注册一个“**关闭钩子**”。

```java

public void start() {
    //1. 如果持有的Server实例为空，就解析server.xml创建出来
    if (getServer() == null) {
        load();
    }
    //2. 如果创建失败，报错退出
    if (getServer() == null) {
        log.fatal(sm.getString("catalina.noServer"));
        return;
    }

    //3.启动Server
    try {
        getServer().start();
    } catch (LifecycleException e) {
        return;
    }

    //创建并注册关闭钩子
    if (useShutdownHook) {
        if (shutdownHook == null) {
            shutdownHook = new CatalinaShutdownHook();
        }
        Runtime.getRuntime().addShutdownHook(shutdownHook);
    }

    //用await方法监听停止请求
    if (await) {
        await();
        stop();
    }
}
```

关闭钩子其实是一个线程，JVM在停止（Ctrl + C）之前会尝试执行这个线程的run方法。Tomcat的关闭方法就是执行Server的stop方法。代码如下：

```java

protected class CatalinaShutdownHook extends Thread {

    @Override
    public void run() {
        try {
            if (getServer() != null) {
                Catalina.this.stop();
            }
        } catch (Throwable ex) {
           ...
        }
    }
}
```

## 3. Server

​		Server 组件的具体实现类是 StandardServer，它管理着众多的Service，包括Service的存储和生命周期。每次加入一个Service都是新创建一个原来Service数量+1的数组，是为了节省空间。然后就开始启动这个Service。代码如下

```

@Override
public void addService(Service service) {

    service.setServer(this);

    synchronized (servicesLock) {
        //创建一个长度+1的新数组
        Service results[] = new Service[services.length + 1];
        
        //将老的数据复制过去
        System.arraycopy(services, 0, results, 0, services.length);
        results[services.length] = service;
        services = results;

        //启动Service组件
        if (getState().isAvailable()) {
            try {
                service.start();
            } catch (LifecycleException e) {
                // Ignore
            }
        }

        //触发监听事件
        support.firePropertyChange("service", null, service);
    }

}
```

​		除此之外，Server 组件还有一个重要的任务是启动一个 Socket 来监听停止端口，这就是为什么你能通过 shutdown 命令来关闭 Tomcat。不知道你留意到没有，上面 Catalina 的启动方法的最后一行代码就是调用了 Server 的 await 方法。在 await 方法里会创建一个 Socket 监听 8005 端口，并在一个死循环里接收 Socket 上的连接请求，如果有新的连接到来就建立连接，然后从 Socket 中读取数据；如果读到的数据是停止命令“SHUTDOWN”，就退出循环，进入 stop 流程。

## 4. Service组件

StandardService 继承了 LifecycleBase 抽象类，他还管理了容器组件，代码如下：

```java

public class StandardService extends LifecycleBase implements Service {
    //名字
    private String name = null;
    
    //Server实例
    private Server server = null;

    //连接器数组
    protected Connector connectors[] = new Connector[0];
    private final Object connectorsLock = new Object();

    //对应的Engine容器
    private Engine engine = null;
    
    //映射器及其监听器
    protected final Mapper mapper = new Mapper();
    protected final MapperListener mapperListener = new MapperListener(this);
```

​		那为什么还有一个 MapperListener？这是因为 Tomcat 支持热部署，当 Web 应用的部署发生变化时，Mapper 中的映射信息也要跟着变化，MapperListener 就是一个监听器，它监听容器的变化，并把信息更新到 Mapper 中，这是典型的观察者模式。

**Service的启动方法如下：**Service 先启动了 Engine 组件，再启动 Mapper 监听器，最后才是启动连接器关闭顺序与启动顺序相反。

```

protected void startInternal() throws LifecycleException {

    //1. 触发启动监听器
    setState(LifecycleState.STARTING);

    //2. 先启动Engine，Engine会启动它子容器
    if (engine != null) {
        synchronized (engine) {
            engine.start();
        }
    }
    
    //3. 再启动Mapper监听器
    mapperListener.start();

    //4.最后启动连接器，连接器会启动它子组件，比如Endpoint
    synchronized (connectorsLock) {
        for (Connector connector: connectors) {
            if (connector.getState() != LifecycleState.FAILED) {
                connector.start();
            }
        }
    }
}
```

## 5. Engine组件

Engine继承了 ContainerBase 基类，这个基类里面存储Host容器，以及定义了Host启动和终止方法，Engine里面做了啥呢？就是将清楚分发给Host，让他处理。

```java

final class StandardEngineValve extends ValveBase {

    public final void invoke(Request request, Response response)
      throws IOException, ServletException {
  
      //拿到请求中的Host容器
      Host host = request.getHost();
      if (host == null) {
          return;
      }
  
      // 调用Host容器中的Pipeline中的第一个Valve
      host.getPipeline().getFirst().invoke(request, response);
  }
  
}
```

​		这个基础阀实现非常简单，就是把请求转发到 Host 容器。你可能好奇，从代码中可以看到，处理请求的 Host 容器对象是从请求中拿到的，请求对象中怎么会有 Host 容器呢？这是因为请求到达 Engine 容器中之前，Mapper 组件已经对请求进行了路由处理，Mapper 组件通过请求的 URL 定位了相应的容器，并且把容器对象保存到了请求对象中。