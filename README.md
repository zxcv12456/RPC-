# My-RPC
My-RPC是一款基于Zookeeper实现的RPC框架。网络传输基于Netty实现，并且完成注册中心设置，实现了自定义序列化方式和多种负载均衡算法。
## RPC介绍
RPC远程过程调用，它是一个通过网络从远处计算机程序上请求服务，而不需要了解底层网络技术的协议。比如两个不同的服务 A、B 部署在两台不同的机器上，服务 A 如果想要调用服务 B 中的某个方法该怎么办呢？使用 HTTP请求就可以，但是可能会比较慢而且一些优化做的并不好。而RPC框架就是为了解决这个问题。
RPC框架能实现远程调用，使得分布式或者微服务系统中不同服务之间的调用像本地调用一样简单。
## 架构
![$120C~1F6HR0QQT(` R%PAL](https://user-images.githubusercontent.com/78541558/159341951-2cc43a52-fbce-4361-bb6f-78c0632fcdbe.jpg)
                                                    完整RPC架构图
## 特性

 - 实现了基于Netty传输的网络传输方式
 - 使用Zookeeper作为注册中心，管理服务提供者信息
 - 实现了两种负载均衡算法:随机算法和轮询算法
 - 实现了两种序列化方式，Java原生序列化方式和Json序列化方式
 - 消费者和提供者使用Netty方式，会采用Netty的心跳机制，保证连接
 - 接口抽象良好,模块耦合度低，序列化器、负载均衡算法可配置
 - 实现自定义的通信协议
 - 服务提供侧自动注册服务 
 ## 项目模块概览
 
 - myRPC-client———RPC客户端
 - myRPC-codec———自定义通信协议与自定义序列化方式
 - myRPC-common———实体对象、信息格式等公共类
 - myRPC-loadbalance———负载均衡
 - myRPC-register———注册中心
 - myRPC-server———RPC服务端
 - myRPC-service———后台服务数据
 ## 传输协议
调用参数和返回值的采用如下自定义协议以防止粘包:
|消息类型（2 bytes）| 序列化方式（2bytes ） | 消息长度（4 bytes） |
| ------ | ------ | ------ |
| 序列化后的Data | 序列化后的Data | 序列化后的Data |

自定义传输协议的字段和对应的解释
| 字段 | 解释 |
| ------ | ------ |
|消息类型| 标明是请求quest还是响应sponse | 
|序列化方式 | 标明这个包的数据的序列化方式 |
|消息长度|数据字节的长度|

## 启动结果演示
![U5ZQ@A $(VN$8T $)N77~~G](https://user-images.githubusercontent.com/78541558/157439744-bac21e27-a3bb-4c31-8104-302b8477f5b9.png)
![OFUUA(ZY (M8SET4RJG46HO](https://user-images.githubusercontent.com/78541558/157439758-bd5f5c1a-4d7c-4851-b26c-677691612e0d.png)
## 设计思路
1.要想服务端可以进行多方法调用，需要把Request抽象。                                      
2.要想返回值支持多种对象，需要把Response抽象。                                          
3.要想加快调用速度，需要在Netty高性能网络框架上进行通信。                                
4.要解决粘包或者分包问题，需要自定义传输协议。                                           
5.要加快序列化速度，需要自定义序列化方式。                                              
6.要完成服务的注册与发现，要设置注册中心                                               
7.要分散服务提供者的压力，要完成负载均衡功能                                          
一个完整的RPC框架就逐步完成了                                                       
## 代码原理分析
### 公共模块
### 定义消息格式和实体对象
**1.定义Request类与Response类**

定义Request类和Response类，这些定义都要继承Serializable接口。目的是为了实现传输途中的序列化和反序列化。其中Request包含服务类名、方法名、参数列表和参数类型。  
要采用Json方式序列化时，会出现当一个类定义了某个类型的父类作为成员变量，实际存放的为某个子类型，反序列化后，属性丢失的情况。
所以我们需要在Request类和Response类中都添加一个参数类型paramsTypes和返回对象类型dataType的类成员变量，以保证反序列化的顺利完成。
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
~~~
public class RPCResponse implements Serializable {  
      
    private int code;  
 private String message;  
  private Class<?> dataType;  
 private Object data;  
  
 public static RPCResponse success(Object data) {  
        return RPCResponse.builder().code(200).data(data).dataType(data.getClass()).build();  
  }  
    public static RPCResponse fail() {  
        return RPCResponse.builder().code(500).message("服务器发生错误").build();  
  }  
}
~~~
### RPC客户端模块
### 完成RPC客户端、动态代理和Netty客户端、服务发现功能
**1.创建主客户端**                                                                                                        
   TestClient类，先创建一个netty客户端，使得Netty客户端开始启动。再创建一个代理客户端，动态代理想调用服务端的哪个类，再生成一个实例代理对象，然后完成调用。
~~~
public class TestClient {  
    public static void main(String[] args) {  
        // 构建一个使用java Socket/ netty/....传输的客户端  
  RPCClient rpcClient = new NettyRPCClient();  
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
  // 测试json调用空参数方法  
  System.out.println(userService.hello());  
  
  }  
}
~~~
**2.创建RPC动态代理类**  
RPCClientProxy类，客户端实现动态代理，但不是要真的行使服务端方法，而是在invoke方法里通过反射把要调用的服务名、方法名、与查找的参数等信息，封装成request类，并开始调用Netty中的sendRequest方法，使得Netty开始传输request请求，从而返回response对象。
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
**3.创建Netty客户端**    
在客户端通过Bootstrap来完成netty客户端的初始化。用一个静态代码块创建一个NioEventLoopGroup线程池，一个Bootstrap实例对象。通过Bootstrap进行netty客户端的初始化。   
此类中定义一个ServiceRegister类的属性，且在类的构造函数中实例化serviceRegister。先通过zookeeper注册中心中服务发现功能，得到IP地址和端口号。  
netty是异步和事件驱动的，得到端口并进行端口连接后会返回channelFuture对象，通过它来查询提交任务的执行状态和最终的结果。向channel中写队列并刷新，后等待监听端口关闭。  
在定义channel时，就自带了一个跟hashmap类似的AttributeMap的属性。调用的结果Response通过给channel设置别名返回。
~~~
public class NettyRPCClient implements RPCClient {  
    private static final Bootstrap bootstrap;  
 private static final EventLoopGroup eventLoopGroup;  
 private String host;  
 private int port;  
 private ServiceRegister serviceRegister;  
 public NettyRPCClient() {  
        this.serviceRegister = new ZkServiceRegister();  
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
        InetSocketAddress address = serviceRegister.serviceDiscovery(request.getInterfaceName());  
  host = address.getHostName();  
  port = address.getPort();  
 try {  
            ChannelFuture channelFuture  = bootstrap.connect(host, port).sync();  
  Channel channel = channelFuture.channel();  
  // 发送数据  
  channel.writeAndFlush(request);  
  channel.closeFuture().sync();  
  // 阻塞的获得结果，通过给channel设计别名，获取特定名字下的channel中的内容（这个在hanlder中设置）  
  // AttributeKey是，线程隔离的，不会由线程安全问题。  
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
**4.创建Netty客户端的ChannelInitializer类**  
每个channel都有一个对应的唯一的ChannelPipeline对象。它是一个Handler的集合。它负责处理和拦截 inbound 或者 outbound 的事件和操作，相当于一个贯穿netty的责任链。ChannelInitializer类中有initChannel方法可自定义的在ChannelPipeline中添加事件和操作。  
在TCP传输过程中会出现粘包或者分包的情况。通过自定义传输协议，添加一个MyEncode类编码器和一个MyDecode类解码器，让生成的数据包先进行补充，其中就添加了传输消息的长度和使用的序列化方式，传输后会按照消息中的传输长度和序列化方式对接收的消息进行解码。
在Pipeline中，channel会根据为in操作还是out操作，来使得从头到尾或者从尾到头行使不同的方法。  
在ChannelPipeline中添加一个NettyClientHandler类。
~~~
public class NettyClientInitializer extends ChannelInitializer<SocketChannel> {  
    @Override  
  protected void initChannel(SocketChannel ch) throws Exception {  
        ChannelPipeline pipeline = ch.pipeline();  
  // 使用自定义的编解码器  
  pipeline.addLast(new MyDecode());  
  // 编码需要传入序列化器，这里是json，还支持ObjectSerializer，也可以自己实现其他的  
  pipeline.addLast(new MyEncode(new JsonSerializer()));  
  pipeline.addLast(new NettyClientHandler());  
  }  
}
~~~
**5.创建Netty客户具体的Handler**  
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
### RPC服务端模块
### 完成RPC服务端和Netty服务端、反射、服务注册功能
**1.创建RPC主服务端**  
创建一个ServiceProvider类，把服务端服务对象的接口和对象一一对应起来，并且输入IP地址和端口。创建Netty服务端并开启端口，使得Netty服务端运行起来。
~~~
public class TestServer {  
    public static void main(String[] args) {  
        UserService userService = new UserServiceImpl();  
  BlogService blogService = new BlogServiceImpl();  
  ServiceProvider serviceProvider = new ServiceProvider("127.0.0.1", 8899);  
  serviceProvider.provideServiceInterface(userService);  
  serviceProvider.provideServiceInterface(blogService);  
  
  RPCServer RPCServer = new NettyRPCServer(serviceProvider);  
  RPCServer.start(8899);  
  }  
}
~~~
**2.创建ServiceProvider类**  
提供的服务对象不止一种，有User和Blog等多对象，且得把服务使用的IP地址和端口号注册到注册中心上完成服务注册功能。  
创建一个类，其中属性中有host属性和port属性。在构造方法里面创建一个哈希表和一个ZkServiceRegister类。用哈希表记录下用到的方法和对应的对象，可在后续知道方法后，通过查询来获得对象，从而真正调用方法。其中也调用了ZkServiceRegister的register方法，完成了服务注册功能。
~~~
public class ServiceProvider {  
    /**  
 * 一个实现类可能实现多个服务接口，  
  */  
  private Map<String, Object> interfaceProvider;  
  
 private ServiceRegister serviceRegister;  
 private String host;  
 private int port;  
  
 public ServiceProvider(String host, int port){  
        // 需要传入服务端自身的服务的网络地址  
  this.host = host;  
 this.port = port;  
 this.interfaceProvider = new HashMap<>();  
 this.serviceRegister = new ZkServiceRegister();  
  }  
  
    public void provideServiceInterface(Object service){  
        Class<?>[] interfaces = service.getClass().getInterfaces();  
  
 for(Class clazz : interfaces){  
            // 本机的映射表  
  interfaceProvider.put(clazz.getName(),service);  
  // 在注册中心注册服务  
  serviceRegister.register(clazz.getName(),new InetSocketAddress(host,port));  
  }  
  
    }  
  
    public Object getService(String interfaceName){  
        return interfaceProvider.get(interfaceName);  
  }  
}
~~~
**3.创建Netty服务端**  
在服务端用serverBootstrap来完成netty服务端的初始化。创建两个NioEventLoopGroup线程池。一个bossGroup负责建立与客户端的连接。一个workGroup负责与客户端的读写操作。跟netty客户端一样，建立连接并监听。
~~~
public class NettyRPCServer implements RPCServer {  
    private ServiceProvider serviceProvider;  
  @Override  
  public void start(int port) {  
        // netty 服务线程组boss负责建立连接， work负责具体的请求  
  NioEventLoopGroup bossGroup = new NioEventLoopGroup();  
  NioEventLoopGroup workGroup = new NioEventLoopGroup();  
  System.out.println("Netty服务端启动了...端口号为" + port );  
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
**4.创建Netty服务端的channelInitializer类**  
在设置类时多设置一个属性 ServiceProvider类。用于传递记录着接口与对应的对象的消息。其他的和Netty客户端的channelInitializer的设计一致
~~~
public class NettyServerInitializer extends ChannelInitializer<SocketChannel> {  
    private ServiceProvider serviceProvider;  
  @Override  
  protected void initChannel(SocketChannel ch) throws Exception {  
        ChannelPipeline pipeline = ch.pipeline();  
  // 使用自定义的编解码器  
  pipeline.addLast(new MyDecode());  
  // 编码需要传入序列化器，这里是json，还支持ObjectSerializer，也可以自己实现其他的  
  pipeline.addLast(new MyEncode(new JsonSerializer()));  
  pipeline.addLast(new NettyRPCServerHandler(serviceProvider));  
  }  
}
~~~
**5.创建Netty服务端具体的Handler**  
它继承于SimpleChannelInboundHandler接口，同客户端一样重写channelRead0方法。在接收到request对象时调用channelRead0方法。
其中的getResponse方法，通过接口名查询serviceProvider，得到对应的服务类。通过反射得到对应方法并使用方法，从而查询数据库返回response对象。
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
### 后台服务数据模块
### 模拟从数据库查询或者插入的环节  
UserServiceImpl等类
~~~
public class UserServiceImpl implements UserService {  
    @Override  
  public User getUserByUserId(Integer id) {  
        // 模拟从数据库中取用户的行为  
  User user = User.builder().id(id).userName("he2121").sex(true).build();  
  System.out.println("客户端查询了"+id+"用户");  
 return user;  
  }  
  
    @Override  
  public Integer insertUserId(User user) {  
        System.out.println("插入数据成功："+user);  
 return 1;  
  }  
  
    @Override  
  public String hello() {  
        return "Hello World!";  
  }  
}
~~~
### 自定义通信协议与自定义序列化方式模块
### 完成编码类和解码类和自定义序列化方式
**1.自定义JSON序列化方式**  
JSON 是一种轻量级的数据交换语言，该语言以易于让人阅读的文字为基础，用来传输由属性值或者序列性的值组成的数据对象，类似 xml，Json 比 xml更小、更快更容易解析。JSON 由于采用字符方式存储，占用相对于字节方式较大，并且序列化后类的信息会丢失，可能导致反序列化失败，所以我们这里额外的操作使得反序列化得已成功。  
原因分析:一个类定义了某个类型的父类作为成员变量，实际存放的为某个子类型， JSON 反序列化后，属性丢失的情况。因为在序列化之前类成员变量，定义的类型为父类，实际存放的为子类的信息，里面有着子类的特有属性。在反序列化时根据父类进行还原，就会出现属性丢失的情况。  
解决办法：加入标记，在Request类和Response类中都添加两个标记，参数类型paramsTypes和返回对象类型dataType的类成员变量。在反序列化的时候，先判断是否为它本身类型，不是就通过标记进行还原。  
使用fastjson进行解析和生成，通过之前的自定义传输协议的传输类型进行不同的反序列化。
~~~
public class JsonSerializer implements Serializer {  
    @Override  
  public byte[] serialize(Object obj) {  
        byte[] bytes = JSONObject.toJSONBytes(obj);  
 return bytes;  
  }  
  
    @Override  
  public Object deserialize(byte[] bytes, int messageType) {  
        Object obj = null;  
  // 传输的消息分为request与response  
  switch (messageType){  
            case 0:  
                RPCRequest request = JSON.parseObject(bytes, RPCRequest.class);  
  
  // 修bug 参数为空 直接返回  
  if(request.getParams() == null) return request;  
  
  Object[] objects = new Object[request.getParams().length];  
  // 把json字串转化成对应的对象， fastjson可以读出基本数据类型，不用转化  
  for(int i = 0; i < objects.length; i++){  
                    Class<?> paramsType = request.getParamsTypes()[i];  
 if (!paramsType.isAssignableFrom(request.getParams()[i].getClass())){  
                        objects[i] = JSONObject.toJavaObject((JSONObject) request.getParams()[i],request.getParamsTypes()[i]);  
  }else{  
                        objects[i] = request.getParams()[i];  
  }  
  
                }  
                request.setParams(objects);  
  obj = request;  
 break; case 1:  
                RPCResponse response = JSON.parseObject(bytes, RPCResponse.class);  
  Class<?> dataType = response.getDataType();  
 if(! dataType.isAssignableFrom(response.getData().getClass())){  
                    response.setData(JSONObject.toJavaObject((JSONObject) response.getData(),dataType));  
  }  
                obj = response;  
 break; default:  
                System.out.println("暂时不支持此种消息");  
 throw new RuntimeException();  
  }  
        return obj;  
  }  
  
    // 1 代表着json序列化方式  
  @Override  
  public int getType() {  
        return 1;  
  }  
}
~~~
**2.java原生序列化方式**  
使用ByteArrayOutputStream等字节数组输出、输入流类，作用是在内存中创建一个字节数组缓冲区，所有发送到输出、输入流的数据保存在该字节数组缓冲区中，然后读取数据。根据是序列化还是反序列化来进行不同的操作。
~~~
public class ObjectSerializer implements Serializer {  
  
    // 利用java IO 对象 -> 字节数组  
  @Override  
  public byte[] serialize(Object obj) {  
        byte[] bytes = null;  
  ByteArrayOutputStream bos = new ByteArrayOutputStream();  
 try {  
            ObjectOutputStream oos = new ObjectOutputStream(bos);  
  oos.writeObject(obj);  
  oos.flush();  
  bytes = bos.toByteArray();  
  oos.close();  
  bos.close();  
  } catch (IOException e) {  
            e.printStackTrace();  
  }  
  
        return bytes;  
  }  
  
    // 字节数组 -> 对象  
  @Override  
  public Object deserialize(byte[] bytes, int messageType) {  
        Object obj = null;  
  ByteArrayInputStream bis = new ByteArrayInputStream(bytes);  
 try {  
            ObjectInputStream ois = new ObjectInputStream(bis);  
  obj = ois.readObject();  
  ois.close();  
  bis.close();  
  } catch (IOException | ClassNotFoundException e) {  
            e.printStackTrace();  
  }  
        return obj;  
  }  
  
    // 0 代表java原生序列化器  
  @Override  
  public int getType() {  
        return 0;  
  }  
}
~~~
**3.自定义通信协议，完成编码类**  
该自定义编码类继承于MessageToByteEncoder类。MessageToByteEncoder类是netty编码的抽象类，其实现了channelRead方法，而我们只要实现其encode方法即可。在它的属性上，需要有一个serialize器，负责将传入的对象序列化成字节数组。  
根据我们之前自定义的通信协议的方式，先根据类型输入消息类型，在所要传输ByteBuf字节数据上写入序列化方式、传输信息的字节长度和通过序列化方法得到的序列化字节数组。
~~~
public class MyEncode extends MessageToByteEncoder {  
    private Serializer serializer;  
  
  @Override  
  protected void encode(ChannelHandlerContext ctx, Object msg, ByteBuf out) throws Exception {  
        // 写入消息类型  
  if(msg instanceof RPCRequest){  
            out.writeShort(MessageType.REQUEST.getCode());  
  }  
        else if(msg instanceof RPCResponse){  
            out.writeShort(MessageType.RESPONSE.getCode());  
  }  
        // 写入序列化方式  
  out.writeShort(serializer.getType());  
  // 得到序列化数组  
  byte[] serialize = serializer.serialize(msg);  
  // 写入长度  
  out.writeInt(serialize.length);  
  // 写入序列化字节数组  
  out.writeBytes(serialize);  
  }  
}
~~~
**4.自定义通信协议，完成解码类**  
跟编码类一样，继承；继承来自netty解码抽象类的ByteToMessageDecoder类。  
先读取一个2字节的消息类型，再读取一个2字节的序列化类型，根据序列化器中的getSerializerByCode方法得到一个序列化器实例。再读取4个字节的数据字节长度，根据长度创建并输入消息。通过实例序列化对象调用方法，反序列化返回对象。
~~~
public class MyDecode extends ByteToMessageDecoder {  
  
  
    @Override  
  protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {  
        // 1. 读取消息类型  
  short messageType = in.readShort();  
  // 现在还只支持request与response请求  
  if(messageType != MessageType.REQUEST.getCode() &&  
                messageType != MessageType.RESPONSE.getCode()){  
            System.out.println("暂不支持此种数据");  
 return;  }  
        // 2. 读取序列化的类型  
  short serializerType = in.readShort();  
  // 根据类型得到相应的序列化器  
  Serializer serializer = Serializer.getSerializerByCode(serializerType);  
 if(serializer == null)throw new RuntimeException("不存在对应的序列化器");  
  // 3. 读取数据序列化后的字节长度  
  int length = in.readInt();  
  // 4. 读取序列化数组  
  byte[] bytes = new byte[length];  
  in.readBytes(bytes);  
  // 用对应的序列化器解码字节数组  
  Object deserialize = serializer.deserialize(bytes, messageType);  
  out.add(deserialize);  
  }  
}
~~~
### 注册中心模块
### 完成注册中心的设置，实现服务注册与服务发现功能
**1.设置注册中心，完成zookeeper客户端的设置，实现注册中心的功能**  
先建立zookeeper的客户端Curator。Curator是Zookeeper开源的客户端框架。将根节点设置为MyRPC，参数sessionTimeoutMs来设定会话的超时时间。建立完成后，开启客户端。  
服务注册功能：通过给zookeeper添加子节点来实现。在过去ServiceProvider类中的方法中，将服务提供者的服务名和所在地址作为参数传入了过来。客户端将服务名设置成永久节点，把地址设为临时节点。服务下线时，就会只删除地址，不删除服务名。先检查客户端没有该服务名就添加，调用getServiceAddress方法，获得地址并作为临时节点添加。  
服务发现功能：通过客户端的getChildren方法获得，服务名的所有子节点，也就是地址。在这个几个地址中，通过选择不同负载均衡算法，来选择得到的地址。通过parseAddress方法创建将地址转化为InetSocketAddress类，最后返回。RPC客户端在NettyRPCClient中调用注册中心的serviceDiscovery方法得到地址。

~~~
public class ZkServiceRegister implements ServiceRegister {  
    // curator 提供的zookeeper客户端  
  private CuratorFramework client;  
  // zookeeper根路径节点  
  private static final String ROOT_PATH = "MyRPC";  
  // 初始化负载均衡器， 这里用的是随机， 一般通过构造函数传入  
  private LoadBalance loadBalance = new RandomLoadBalance();  
  
  // 这里负责zookeeper客户端的初始化，并与zookeeper服务端建立连接  
  public ZkServiceRegister(){  
        // 指数时间重试  
  RetryPolicy policy = new ExponentialBackoffRetry(1000, 3);  
  // zookeeper的地址固定，不管是服务提供者还是，消费者都要与之建立连接  
  // sessionTimeoutMs 与 zoo.cfg中的tickTime 有关系，  
  // zk还会根据minSessionTimeout与maxSessionTimeout两个参数重新调整最后的超时值。默认分别为tickTime 的2倍和20倍  
  // 使用心跳监听状态  
  this.client = CuratorFrameworkFactory.builder().connectString("127.0.0.1:2181")  
                .sessionTimeoutMs(40000).retryPolicy(policy).namespace(ROOT_PATH).build();  
 this.client.start();  
  System.out.println("zookeeper 连接成功");  
  }  
  
    @Override  
  public void register(String serviceName, InetSocketAddress serverAddress){  
        try {  
            // serviceName创建成永久节点，服务提供者下线时，不删服务名，只删地址  
  if(client.checkExists().forPath("/" + serviceName) == null){  
                client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath("/" + serviceName);  
  }  
            // 路径地址，一个/代表一个节点  
  String path = "/" + serviceName +"/"+ getServiceAddress(serverAddress);  
  // 临时节点，服务器下线就删除节点  
  client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath(path);  
  } catch (Exception e) {  
            System.out.println("此服务已存在");  
  }  
    }  
    // 根据服务名返回地址,服务发现  
  @Override  
  public InetSocketAddress serviceDiscovery(String serviceName) {  
        try {  
            List<String> strings = client.getChildren().forPath("/" + serviceName);  
  // 负载均衡选择器，选择一个  
  String string = loadBalance.balance(strings);  
 return parseAddress(string);  
  } catch (Exception e) {  
            e.printStackTrace();  
  }  
        return null;  
  }  
  
    // 地址 -> XXX.XXX.XXX.XXX:port 字符串  
  private String getServiceAddress(InetSocketAddress serverAddress) {  
        return serverAddress.getHostName() +  
                ":" +  
                serverAddress.getPort();  
  }  
    // 字符串解析为地址  
  private InetSocketAddress parseAddress(String address) {  
        String[] result = address.split(":");  
 return new InetSocketAddress(result[0], Integer.parseInt(result[1]));  
  }  
}
~~~
### 负载均衡模块
###  完成多个负载均衡算法
**1.随机负载均衡算法**  
使用random函数，随机生成数，实现随机负载均衡算法。
~~~
public class RandomLoadBalance implements LoadBalance{  
    @Override  
  public String balance(List<String> addressList) {  
  
        Random random = new Random();  
 int choose = random.nextInt(addressList.size());  
  System.out.println("负载均衡选择了" + choose + "服务器");  
 return addressList.get(choose);  
  }  
}
~~~
**2.轮询负载均衡算法**  
通过除余得到结果，实现轮询负载均衡算法
~~~
public class RoundLoadBalance implements LoadBalance{  
    private int choose = -1;  
  @Override  
  public String balance(List<String> addressList) {  
        choose++;  
  choose = choose%addressList.size();  
 return addressList.get(choose);  
  }  
}
~~~
