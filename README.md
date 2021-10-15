# RPC框架
## 介绍
RPC远程过程调用，它是一个通过网络从远处计算机程序上请求服务，而不需要了解底层网络技术的协议。比如两个不同的服务 A、B 部署在两台不同的机器上，服务 A 如果想要调用服务 B 中的某个方法该怎么办呢？使用 HTTP请求就可以，但是可能会比较慢而且一些优化做的并不好。而RPC框架就是为了解决这个问题。
RPC框架能实现远程调用，使得分布式或者微服务系统中不同服务之间的调用像本地调用一样简单。
## 完成的功能
服务端可实现多个不同对象服务的查询，能提供不同的服务，客户端实现查找与插入等不同的方法。在实际使用过程中，调用方法的速度快。
## 设计思路
1.要想服务端可以进行多方法调用，需要把Request抽象。
2.要想返回值支持多种对象，需要把Response抽象。
3.要想加快调用速度，需要在netty高性能网络框架上进行通信。
## 运行原理
### 1.定义Request类与Response类和user与blog对象
定义request类，这些定义都要继承Serializable接口。目的是为了实现传输途中的序列号和反序列化。
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
### 2.实现RPCClientProxy类和UserServiceImpl等多对象模拟类
客户端实现动态代理，但不是要真的行使服务端方法，而是在invoke方法里通过反射把要调用的服务名、方法名、与查找的参数等信息，封装成request类，并使用调用netty开始运作，使用sendRequest方法，从而返回response对象。
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
