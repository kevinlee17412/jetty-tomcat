# Tomcat整体架构

Tomcat有两个核心功能：

- 处理socket连接，负责网络字节流与request对象和response对象之间的转换。
- 加载和管理socket，并处理具体的request请求。

因此Tomcat设计了两个核心组件：连接器（connector）和容器（container）。

<img src="https://static001.geekbang.org/resource/image/ee/d6/ee880033c5ae38125fa91fb3c4f8cad6.jpg" alt="img" style="zoom:33%;" />

​		最外层就是一个Server，即Tomcat实例，里面包含多个Service组件，使我们可以通过不同端口访问一台机器上部署的不同应用。一个Service组件里面包含多个连接器和一个容器，连接器和容器之间通过标准的ServletRequest和ServletResponse通信。多种连接器是为了实现支持多种 I/O 模型和应用层协议。

## 1、连接器

**连接器功能：**

1. 监听网络端口。
2. 接受网络连接请求。
3. 读取网络请求字节流。
4. 根据具体应用层协议（HTTP/AJP）解析字节流，生成统一的 Tomcat Request 对象。
5. 将 Tomcat Request 对象转成标准的 ServletRequest。
6. 调用 Servlet 容器，得到 ServletResponse。
7. 将 ServletResponse 转成 Tomcat Response 对象。
8. 将 Tomcat Response 转成网络字节流。
9. 将响应字节流写回给浏览器。

**连接器组件：**

<img src="https://static001.geekbang.org/resource/image/6e/ce/6eeaeb93839adcb4e76c15ee93f545ce.jpg" alt="img" style="zoom:33%;" />

​		从上图可以看到连接器核心组件有三个：EndPoint、Processor和Adapter，作用分别是：网络通信（nio，nio2）、应用层协议解析（Http，AJP）和Tomcat Request/Response 与 ServletRequest/ServletResponse 的转化。其中EndPoint和Processor组合在一起称为ProtocolHandler，接口实现如下图所示：

<img src="https://static001.geekbang.org/resource/image/13/55/13850ee56c3f09cbabe9892e84502155.jpg" alt="img" style="zoom:33%;" />

可以看到通信io和应用层协议相互组合。

从源码角度看连接器组件：

<img src="https://static001.geekbang.org/resource/image/30/cf/309cae2e132210489d327cf55b284dcf.jpg" alt="img" style="zoom: 33%;" />

​		其中 Acceptor 用于监听 Socket 连接请求。SocketProcessor 用于处理接收到的 Socket 请求，它实现 Runnable 接口，在 run 方法里调用协议处理组件 Processor 进行处理。为了提高处理能力，SocketProcessor 被提交到线程池来执行。而这个线程池叫作执行器（Executor)。Processor 通过解析生成 Request 对象后，会调用 Adapter 的 Service 方法。

​		Tomcat使用自己的Request对象来存放请求，ProtocolHandler 接口负责解析请求并生成 Tomcat Request 类。连接器调用 CoyoteAdapter 的 sevice 方法，传入的是 Tomcat Request 对象，CoyoteAdapter 负责将 Tomcat Request 转成 ServletRequest，再调用容器的 service 方法。

# 2、容器

​		Tomcat有四个容器，分别是Engine、Host、Context 和 Wrapper。关系是包含关系，如下图所示。

<img src="https://static001.geekbang.org/resource/image/cc/ed/cc968a11925591df558da0e7393f06ed.jpg" alt="img" style="zoom:33%;" />

​		设计成多层容器的目的是为了使得 Servlet 容器具有很好的灵活性。Tomcat 是怎么管理这些容器的呢？这些容器形成一个树形结构，使用组合模式来管理这些容器。

<!--扩展组合模式--区别于继承-->

```java
public class Fruit{
    public void drink(){
        System.out.println("drink water");
    }
    public void grow(){
        System.out.println('growing');
    }
}

public class Tomato{
    private Fruit f;
    public Tomato(Fruit f){
        this.f = f;
    }
    public void drink(){
        f.Drink();
    }
    public void breath(){
        System.out.println("growing leaves");
    }
}
```

## 2.1 一个请求如何找到对应的Servlet



<img src="https://static001.geekbang.org/resource/image/be/96/be22494588ca4f79358347468cd62496.jpg" alt="img" style="zoom:33%;" />

​		首先一个通过协议和端口号找到对应的连接器，一个连接器对应一个容器，即确定了Service和Engine，然后通过域名确定host容器，通过URL地址确定context容器，Context 确定后，Mapper 再根据web.xml中配置的 Servlet 映射路径来找到具体的 Wrapper 和 Servlet。

<img src="https://static001.geekbang.org/resource/image/b0/ca/b014ecce1f64b771bd58da62c05162ca.jpg" alt="img" style="zoom:33%;" />

​		请求的传递是通过责任链模式----pipeline，每一个容器有一个pipeline，pipeline中保存了该容器的value链，通过调用pipeline的getBasic得到下一个容器value链的末尾value，value通过getNext得到下一个value，value通过invoke方法处理请求。整个调用过程由连接器中的 Adapter 触发的，它会调用 Engine 的第一个 Valve：

```java
// Calling the container
connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);
```

```java
public interface Valve {
  public Valve getNext();
  public void setNext(Valve valve);
  public void invoke(Request request, Response response)
}
```

```java

public interface Pipeline extends Contained {
  public void addValve(Valve valve);
  public Valve getBasic();
  public void setBasic(Valve valve);
  public Valve getFirst();
}
```





