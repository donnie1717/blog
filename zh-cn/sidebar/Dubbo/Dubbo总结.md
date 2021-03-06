### RPC是什么？
RPC叫做远程过程调用，对应的是本地过程调用。在微服务中，当我们想调用到其他服务中的某个方法的时候，就需要利用到RPC。

### RPC与HTTP的区别
首先RPC与HTTP并不是一个概念的东西。RPC的概念倾向于远程过程调用这个方式，其中的通信协议可以是HTTP或者自定义的TCP协议。HTTP协议则侧重于传输协议本身。

HTTP协议相比于RPC自定义实现的协议较为冗余。

### 服务导出的过程
- 服务导出的过程起源于Spring容器初始化完毕。
- 首先会根据配置获取注册中心的URL、服务提供者协议的URL。
- 然后会根据配置决定是进行本地导出还是远程导出。
- 已远程导出为例，导出前会通过ProxyFactory.getInvoker创建真实类的代理对象。然后通过服务提供者的URL调用具体的Protocol进行导出，默认是Dubbo协议
- 首次导出会调用createServer创建Server，默认是NettyServer
- 然后将export得到expoter存入到本地缓存中
- 最终将服务提供者的信息注册到注册中心。


### 服务引入的过程
- 服务引入主要有两种方式分别是饿汉式和懒汉式，饿汉式是在Spring Bean初始化完成后调用，懒汉式则是在Bean首次注入的时候调用
- 首先，读取配置用于创建服务提供者的代理对象
- 根据配置信息决定是本地引入还是远程直连还是注册中心引入
- 以注册中心引入为例，会去构建一个RegisteryDirectory对象向注册中心注册消费者信息，并订阅服务提供者、配置、路由等节点。
- 在获取到服务提供者信息后，会调用具体的调用协议进行引入。以Dubbo协议为例会创建NettyClient用于远程通信，最后通过Cluster来包装Invoker，最终返回代理类。

### 服务调用过程
- 消费者调用服务某个方法，本质是调用服务引用创建的代理类，然后会通过cluster进行路由过滤、负载均衡机制中选择一个invoker发起远程调用，此时会记录此请求的ID等待服务端响应
- 服务端接收到请求后会通过参数找到之前服务导出存储的Map，得到响应的exporter,然后调用真正的实现类。组装好结果返回，响应会带上之前的请求ID。
- 消费者收到这个响应会通过ID去寻找之前的请求，然后将响应塞到对应的Future中，唤起等待线程，最后消费者得到响应。请求结束。

### SPI机制
SPI是一种服务发现机制，通常在框架中会定义一些接口，我们可以通过自定义实现类，并配置在META-INFO 的service目录下实现接口的灵活扩展。

为什么Dubbo不适用原生的SPI机制实现?
- JDK SPI机制默认会加载所有的接口实现，造成资源浪费。
- JDK SPI机制较为简单，无法根据配置指定使用某个实现类。同时也不支持AOP和IOC特性。



