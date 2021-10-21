# RPC框架
## 介绍
RPC远程过程调用，它是一个通过网络从远处计算机程序上请求服务，而不需要了解底层网络技术的协议。比如两个不同的服务 A、B 部署在两台不同的机器上，服务 A 如果想要调用服务 B 中的某个方法该怎么办呢？使用 HTTP请求就可以，但是可能会比较慢而且一些优化做的并不好。而RPC框架就是为了解决这个问题。
RPC框架能实现远程调用，使得分布式或者微服务系统中不同服务之间的调用像本地调用一样简单。
## 完成的功能
服务端可实现多个不同对象服务的查询，能提供不同的服务，客户端实现查找与插入等不同的方法。在实际使用过程中，调用方法的速度快，并能完整传输数据，解决TCP粘包或者分包问题。
## 设计思路
1.要想服务端可以进行多方法调用，需要把Request抽象。  
2.要想返回值支持多种对象，需要把Response抽象。  
3.要想加快调用速度，需要在netty高性能网络框架上进行通信。
## 结果演示
![I_V({4PV)F 3H__U@%HSXP2](https://user-images.githubusercontent.com/78541558/138284992-d60a6f48-8b49-4756-9c56-bfbb0c401365.png)
![AA$1U0(Z6({D74 JWL@X0OV](https://user-images.githubusercontent.com/78541558/138285023-1cbe88fa-a19a-4fe8-8125-a0000bb2a700.png)
## 运行原理
### 1 定义消息格式和完成客户端和服务端的动态代理与反射
**1.1 定义Request类与Response类和user与blog对象**  
定义request类，这些定义都要继承Serializable接口。目的是为了实现传输途中的序列化和反序列化。
~~~
public class RPCRequest implements Serializable {  
    // 服务类名，客户端只知道接口名，在服务端中用接口名指向实现类  
  private String interfaceName;  
  // 方法名  
  private String methodName;  
  // 参数列表  
  private Object[] params;  
  // 参数类型  
  private Class<?>[] paramsTypes;  
}
~~~
定义response类
~~~
public class RPCResponse implements Serializable {  
      
    private int code;  
 private String message;  
 private Object data;  
  
 public static RPCResponse success(Object data) {  
        return RPCResponse.builder().code(200).data(data).build();  
  }  
    public static RPCResponse fail() {  
        return RPCResponse.builder().code(500).message("服务器发生错误").build();  
  }  
}
~~~
**1.2 建立ServiceProvider类**
  
提供的服务对象不止一种，有User和Blog等多对象。用一个创建一个类，用哈希表记录下用到的方法和对应的对象。可在后续知道方法后，通过查询来获得对象，从而真正调用方法。
~~~
public class ServiceProvider {  
    /**  
 * 一个实现类可能实现多个接口  
  */  
  private Map<String, Object> interfaceProvider;  
  
 public ServiceProvider(){  
        this.interfaceProvider = new HashMap<>();  
  }  
  
    public void provideServiceInterface(Object service){  
        Class<?>[] interfaces = service.getClass().getInterfaces();  
  
 for(Class clazz : interfaces){  
            interfaceProvider.put(clazz.getName(),service);  
  }  
  
    }  
  
    public Object getService(String interfaceName){  
        return interfaceProvider.get(interfaceName);  
  }  
}
~~~
**1.3 实现RPCClientProxy类和NettyServerInitializer类下的getResponse方法**  
RPCClientProxy类，客户端实现动态代理，但不是要真的行使服务端方法，而是在invoke方法里通过反射把要调用的服务名、方法名、与查找的参数等信息，封装成request类，并使用调用netty开始运作，使用sendRequest方法，从而返回response对象。
~~~
public class RPCClientProxy implements InvocationHandler {  
    private RPCClient client;  
  
  // jdk 动态代理， 每一次代理对象调用方法，会经过此方法增强（反射获取request对象，socket发送至客户端）  
  @Override  
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
        // request的构建，使用了lombok中的builder，代码简洁  
  RPCRequest request = RPCRequest.builder().interfaceName(method.getDeclaringClass().getName())  
                .methodName(method.getName())  
                .params(args).paramsTypes(method.getParameterTypes()).build();  
  //数据传输  
  RPCResponse response = client.sendRequest(request);  
  //System.out.println(response);  
  return response.getData();  
  }  
    <T>T getProxy(Class<T> clazz){  
        Object o = Proxy.newProxyInstance(clazz.getClassLoader(), new Class[]{clazz}, this);  
 return (T)o;  
  }  
}
~~~
getResponse方法，通过接口名查询哈希表，得到对应的服务类。通过反射得到对应方法并使用方法，从而查询数据库返回response对象。
~~~
RPCResponse getResponse(RPCRequest request) {  
    // 得到服务名  
  String interfaceName = request.getInterfaceName();  
  // 得到服务端相应服务实现类  
  Object service = serviceProvider.getService(interfaceName);  
  // 反射调用方法  
  Method method = null;  
 try {  
        method = service.getClass().getMethod(request.getMethodName(), request.getParamsTypes());  
  Object invoke = method.invoke(service, request.getParams());  
 return RPCResponse.success(invoke);  
  } catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {  
        e.printStackTrace();  
  System.out.println("方法执行错误");  
 return RPCResponse.fail();  
  }  
}
~~~
**1.4 UserServiceImpl等多对象模拟类**  
UserServiceImpl类模拟从数据库查询或者插入的环节
~~~
public class UserServiceImpl implements UserService {  
    @Override  
  public User getUserByUserId(Integer id) {  
        // 模拟从数据库中取用户的行为  
  User user = User.builder().id(id).userName("zxcv").sex(true).build();  
  System.out.println("客户端查询了"+id+"用户");  
 return user;  
  }  
  
    @Override  
  public Integer insertUserId(User user) {  
        System.out.println("插入数据成功："+user);  
 return 1;  
  }  
}
~~~
### 2 构建netty客户端
**2.1 创建NettyRPCClient类**  
在客户端通过Bootstrap来完成netty客户端的初始化。用一个静态代码块创建一个NioEventLoopGroup线程池，一个Bootstrap实例对象。通过Bootstrap进行netty客户端的初始化。  
netty是异步和事件驱动的，在进行端口连接后会返回channelFuture对象，通过它来查询提交任务的执行状态和最终的结果。向channel中写队列并刷新，后等待监听端口关闭。  
在定义channel时，就自带了一个跟hashmap类似的AttributeMap的属性。调用的结果Response可通过给channel设置别名返回。

~~~
public class NettyRPCClient implements RPCClient {  
    private static final Bootstrap bootstrap;  
 private static final EventLoopGroup eventLoopGroup;  
 private String host;  
 private int port;  
 public NettyRPCClient(String host, int port) {  
        this.host = host;  
 this.port = port;  
  }  
    // netty客户端初始化，重复使用  
  static {  
        eventLoopGroup = new NioEventLoopGroup();  
  bootstrap = new Bootstrap();  
  bootstrap.group(eventLoopGroup).channel(NioSocketChannel.class)  
                .handler(new NettyClientInitializer());  
  }  
  
    /**  
 * 这里需要操作一下，因为netty的传输都是异步的，你发送request，会立刻返回一个值， 而不是想要的相应的response  
 */  @Override  
  public RPCResponse sendRequest(RPCRequest request) {  
        try {  
            ChannelFuture channelFuture  = bootstrap.connect(host, port).sync();  
  Channel channel = channelFuture.channel();  
  // 发送数据  
  channel.writeAndFlush(request);  
  channel.closeFuture().sync();  
  // 阻塞的获得结果，通过给channel设计别名，获取特定名字下的channel中的内容（这个在hanlder中设置）  
  // AttributeKey是，线程隔离的，不会由线程安全问题。  
  // 实际上不应通过阻塞，可通过回调函数  
  AttributeKey<RPCResponse> key = AttributeKey.valueOf("RPCResponse");  
  RPCResponse response = channel.attr(key).get();  
  
  System.out.println(response);  
 return response;  
  } catch (InterruptedException e) {  
            e.printStackTrace();  
  }  
        return null;  
  }  
}
~~~
**2.2 创建客户端的ChannelInitializer类**  
每个channel都有一个对应的唯一的ChannelPipeline对象。它是一个Handler的集合。它负责处理和拦截 inbound 或者 outbound 的事件和操作，相当于一个贯穿netty的链。ChannelInitializer类中有initChannel方法可自定义的在ChannelPipeline中添加事件和操作。  
在TCP传输过程中会出现粘包或者分包的情况。通过netty自带的LengthFieldBasedFrameDecoder类解码和LengthFieldPrepender类编码,让生成的数据包先添加一个长度字段，传输后会会按照参数指定的包长度偏移量数据对接收到的数据进行解码。  
再添加ObjectEncoder与ObjectDecoder类，使用的是java的序列化方式。在Pipeline中，channel会根据为in操作还是out操作，来使得从头到尾或者从尾到头行使不同的方法。  
在ChannelPipeline中添加一个NettyClientHandler类。
~~~
public class NettyClientInitializer extends ChannelInitializer<SocketChannel> {  
    @Override  
  protected void initChannel(SocketChannel ch) throws Exception {  
        ChannelPipeline pipeline = ch.pipeline();  
  // 消息格式 [长度][消息体]  
  pipeline.addLast(new LengthFieldBasedFrameDecoder(Integer.MAX_VALUE,0,4,0,4));  
  // 计算当前待大宋消息的长度，写入到前4个字节中  
  pipeline.addLast(new LengthFieldPrepender(4));  
  pipeline.addLast(new ObjectEncoder());  
  
  pipeline.addLast(new ObjectDecoder(new ClassResolver() {  
            @Override  
  public Class<?> resolve(String className) throws ClassNotFoundException {  
                return Class.forName(className);  
  }  
        }));  
  
  pipeline.addLast(new NettyClientHandler());  
  }  
}
~~~
**2.3 创建netty客户端具体的handler**  
它继承于SimpleChannelInboundHandler接口，用于接收对端传输过来的消息，对其通过不同的方法进行操作。我们重写了其中的channelRead0方法，在接收数据后，用channel的AttributeMap属性添加Response对象。
~~~
public class NettyClientHandler extends SimpleChannelInboundHandler<RPCResponse> {  
    @Override  
  protected void channelRead0(ChannelHandlerContext ctx, RPCResponse msg) throws Exception {  
        // 接收到response, 给channel设计别名，让sendRequest里读取response  
  AttributeKey<RPCResponse> key = AttributeKey.valueOf("RPCResponse");  
  ctx.channel().attr(key).set(msg);  
  ctx.channel().close();  
  }  
  
    @Override  
  public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {  
        cause.printStackTrace();  
  ctx.close();  
  }  
}
~~~
### 3 构建netty服务端
**3.1 创建NettyRPCServer类**  
在服务端用serverBootstrap来完成netty服务端的初始化。创建两个NioEventLoopGroup线程池。一个bossGroup负责建立与客户端的连接。一个workGroup负责与客户端的读写操作。跟netty客户端一样，建立连接并监听。
~~~
public class NettyRPCServer implements RPCServer {  
    private ServiceProvider serviceProvider;  
  @Override  
  public void start(int port) {  
        // netty 服务线程组boss负责建立连接， work负责具体的请求  
  NioEventLoopGroup bossGroup = new NioEventLoopGroup();  
  NioEventLoopGroup workGroup = new NioEventLoopGroup();  
  System.out.println("Netty服务端启动了...");  
 try {  
            // 启动netty服务器  
  ServerBootstrap serverBootstrap = new ServerBootstrap();  
  // 初始化  
  serverBootstrap.group(bossGroup,workGroup).channel(NioServerSocketChannel.class)  
                    .childHandler(new NettyServerInitializer(serviceProvider));  
  // 同步阻塞  
  ChannelFuture channelFuture = serverBootstrap.bind(port).sync();  
  // 死循环监听  
  channelFuture.channel().closeFuture().sync();  
  } catch (InterruptedException e) {  
            e.printStackTrace();  
  } finally {  
            bossGroup.shutdownGracefully();  
  workGroup.shutdownGracefully();  
  }  
    }  
  
    @Override  
  public void stop() {  
    }  
}
~~~
**3.2 创建服务端的ChannelInitializer类**  
与客户端的ChannelInitializer类建立一样，在设置类时多设置一个属性
ServiceProvider类。用于传递记录着接口与对应的对象的消息。
~~~
public class NettyServerInitializer extends ChannelInitializer<SocketChannel> {  
    private ServiceProvider serviceProvider;  
  @Override  
  protected void initChannel(SocketChannel ch) throws Exception {  
        ChannelPipeline pipeline = ch.pipeline();  
  // 消息格式 [长度][消息体], 解决粘包问题  
  pipeline.addLast(new LengthFieldBasedFrameDecoder(Integer.MAX_VALUE,0,4,0,4));  
  // 计算当前待大宋消息的长度，写入到前4个字节中  
  pipeline.addLast(new LengthFieldPrepender(4));  
  
  // 这里使用的还是java 序列化方式， netty的自带的解码编码支持传输这种结构  
  pipeline.addLast(new ObjectEncoder());  
  pipeline.addLast(new ObjectDecoder(new ClassResolver() {  
            @Override  
  public Class<?> resolve(String className) throws ClassNotFoundException {  
                return Class.forName(className);  
  }  
        }));  
  
  pipeline.addLast(new NettyRPCServerHandler(serviceProvider));  
  }  
}
~~~
**3.3 创建netty服务端具体的handler**  
它继承于SimpleChannelInboundHandler接口，同客户端一样重写channelRead0方法。在接收到request对象时调用channelRead0方法，在getResponse中通过哈希表得到调用的对象，通过反射得到方法并使用方法。
~~~
public class NettyRPCServerHandler extends SimpleChannelInboundHandler<RPCRequest> {  
    private ServiceProvider serviceProvider;  
  
  
  @Override  
  protected void channelRead0(ChannelHandlerContext ctx, RPCRequest msg) throws Exception {  
        //System.out.println(msg);  
  RPCResponse response = getResponse(msg);  
  ctx.writeAndFlush(response);  
  ctx.close();  
  }  
  
    @Override  
  public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {  
        cause.printStackTrace();  
  ctx.close();  
  }  
  
    RPCResponse getResponse(RPCRequest request) {  
        // 得到服务名  
  String interfaceName = request.getInterfaceName();  
  // 得到服务端相应服务实现类  
  Object service = serviceProvider.getService(interfaceName);  
  // 反射调用方法  
  Method method = null;  
 try {  
            method = service.getClass().getMethod(request.getMethodName(), request.getParamsTypes());  
  Object invoke = method.invoke(service, request.getParams());  
 return RPCResponse.success(invoke);  
  } catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {  
            e.printStackTrace();  
  System.out.println("方法执行错误");  
 return RPCResponse.fail();  
  }  
    }  
}
~~~
### 4 构建主客户端和主服务端
TestClient类，先创建一个netty客户端，地址和要开启的接口。创建一个代理客户端，动态代理想调用服务端的那个类，再生成一个实例代理对象，然后完成调用。
~~~
public class TestClient {  
    public static void main(String[] args) {  
        // 构建一个使用java Socket/ netty/....传输的客户端  
  RPCClient rpcClient = new NettyRPCClient("127.0.0.1", 8899);  
  // 把这个客户端传入代理客户端  
  RPCClientProxy rpcClientProxy = new RPCClientProxy(rpcClient);  
  // 代理客户端根据不同的服务，获得一个代理类， 并且这个代理类的方法以或者增强（封装数据，发送请求）  
  UserService userService = rpcClientProxy.getProxy(UserService.class);  
  // 调用方法  
  User userByUserId = userService.getUserByUserId(10);  
  System.out.println("从服务端得到的user为：" + userByUserId);  
  
  User user = User.builder().userName("张三").id(100).sex(true).build();  
  Integer integer = userService.insertUserId(user);  
  System.out.println("向服务端插入数据："+integer);  
  
  BlogService blogService = rpcClientProxy.getProxy(BlogService.class);  
  
  Blog blogById = blogService.getBlogById(10000);  
  System.out.println("从服务端得到的blog为：" + blogById);  
  
  }  
}
~~~
TestServer类，创建一个ServiceProvider类，把服务端服务对象的接口和对象一一对应起来。开启端口，netty服务端开始运行。
~~~
public class TestServer {  
    public static void main(String[] args) {  
        UserService userService = new UserServiceImpl();  
  BlogService blogService = new BlogServiceImpl();  

  ServiceProvider serviceProvider = new ServiceProvider();  
  serviceProvider.provideServiceInterface(userService);  
  serviceProvider.provideServiceInterface(blogService);  
  
  RPCServer RPCServer = new NettyRPCServer(serviceProvider);  
  RPCServer.start(8899);  
  }  
}
~~~
