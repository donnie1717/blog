[toc]

----

## 一、前言

![image.png](https://dubbo.apache.org/imgs/dev/dubbo-relation.jpg)

服务导出的过程中，我们已经获取了一个代理对象。服务调用就是通过调用这个代理对象的方法。


![image.png](https://dubbo.apache.org/imgs/dev/send-request-process.jpg)

Dubbo官方文档给出了服务调用的具体过程。简述一下就是客户端通过代理对象发起调用，提前构造好协议头，然后将对象序列化成协议体，通过client(Netty)进行网络传输。

服务提供者的NettyServer接收到这个请求后会分发给业务线程池。由业务线程池调用具体的实现方法。


## 二、源码分析

### 客户端调用代码
客户端调用的代码如下
```java
public static void main(String[] args) throws Exception {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring/dubbo-consumer.xml");
        context.start();
        DemoService demoService = context.getBean("demoService", DemoService.class);
        CompletableFuture<String> hello = demoService.sayHelloAsync("world");
        System.out.println("result: " + hello.get());
    }
```

在服务导出结束完成后，我们获取DemoService实际是一个代理对象。通过该代理对象完成方法调用。最终会生成一个RPCInvocation对象调用MockClusterInvoker#invoke方法。

![image.png](https://upload-images.jianshu.io/upload_images/14607771-9ca559ee2dc242ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### MockClusterInvoker#invoke
```java
@Override
    public Result invoke(Invocation invocation) throws RpcException {
        Result result = null;
        
        // 获取mock配置
        String value = getUrl().getMethodParameter(invocation.getMethodName(), MOCK_KEY, Boolean.FALSE.toString()).trim();
        if (value.length() == 0 || "false".equalsIgnoreCase(value)) {
            //no mock
            result = this.invoker.invoke(invocation);
        } else if (value.startsWith("force")) {
            if (logger.isWarnEnabled()) {
                logger.warn("force-mock: " + invocation.getMethodName() + " force-mock enabled , url : " + getUrl());
            }
            //force:direct mock
            result = doMockInvoke(invocation, null);
        } else {
            //fail-mock
            try {
                result = this.invoker.invoke(invocation);

                //fix:#4585
                if(result.getException() != null && result.getException() instanceof RpcException){
                    RpcException rpcException= (RpcException)result.getException();
                    if(rpcException.isBiz()){
                        throw  rpcException;
                    }else {
                        result = doMockInvoke(invocation, rpcException);
                    }
                }

            } catch (RpcException e) {
                if (e.isBiz()) {
                    throw e;
                }

                if (logger.isWarnEnabled()) {
                    logger.warn("fail-mock: " + invocation.getMethodName() + " fail-mock enabled , url : " + getUrl(), e);
                }
                result = doMockInvoke(invocation, e);
            }
        }
        return result;
    }
```
这个方法主要是根据mock配置决定是否调用mock方法
- mock无配置调用真实方法
- mock为force则强制走mock方法
- mock为true，真实方法调用失败后执行mock方法


### AbstractClusterInvoker#invoke

```java
@Override
    public Result invoke(final Invocation invocation) throws RpcException {
        checkWhetherDestroyed();

        // binding attachments into invocation.
        Map<String, Object> contextAttachments = RpcContext.getContext().getObjectAttachments();
        if (contextAttachments != null && contextAttachments.size() != 0) {
            ((RpcInvocation) invocation).addObjectAttachments(contextAttachments);
        }

        // 调用directory.list 主要是做路由过滤
        List<Invoker<T>> invokers = list(invocation);
        // 过滤完成通过SPI机制获取loadBalance实现类
        LoadBalance loadbalance = initLoadBalance(invokers, invocation);
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
        // 调用子类方法
        return doInvoke(invocation, invokers, loadbalance);
    }
```

这段模板代码主要逻辑是：
- 绑定attachement到invocation
- 通过RegistryDirectory过滤Invoker
- 通过SPI机制获取负载均衡实现
- 执行子类的doInvoke方法

最终这里是会调用到FailoverClusterInvoker执行doInvoker方法

### FailoverClusterInvoker#doInvoke

```java
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        // 省略代码
        Set<String> providers = new HashSet<String>(len);
        for (int i = 0; i < len; i++) {       
            // 负载均衡中选择一个Invoker
            Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
            invoked.add(invoker);
            RpcContext.getContext().setInvokers((List) invoked);
            try {
                // 执行方法
                Result result = invoker.invoke(invocation);
                // 省略代码
                return result;
            } catch (RpcException e) {
               // 省略代码
            } catch (Throwable e) {
    // 省略代码
   }
        }
        throw new RpcException();
    }
```
这个方法主要是完成了重试机制的逻辑
- 获取重试次数并循环执行
- 根据负载均衡策略选择一个Invoker
- 执行子类的doInvoke方法

最终调用到DubboInvoker的doInvoke方法

### DubboInvoker#doInvoke
```java
@Override
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        inv.setAttachment(PATH_KEY, getUrl().getPath());
        inv.setAttachment(VERSION_KEY, version);

        // 获取client
        ExchangeClient currentClient;
        if (clients.length == 1) {
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
            // 判断是否是oneWay方式调用
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
            int timeout = getUrl().getMethodPositiveParameter(methodName, TIMEOUT_KEY, DEFAULT_TIMEOUT);
            if (isOneway) {
                boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                // 发送
                currentClient.send(inv, isSent);
                // 返回null
                return AsyncRpcResult.newDefaultAsyncResult(invocation);
            } else {
                ExecutorService executor = getCallbackExecutor(getUrl(), inv);
                CompletableFuture<AppResponse> appResponseFuture =
                        currentClient.request(inv, timeout, executor).thenApply(obj -> (AppResponse) obj);
                // save for 2.6.x compatibility, for example, TraceFilter in Zipkin uses com.alibaba.xxx.FutureAdapter
                FutureContext.getContext().setCompatibleFuture(appResponseFuture);
                AsyncRpcResult result = new AsyncRpcResult(appResponseFuture, inv);
                result.setExecutor(executor);
                return result;
            }
        } catch (TimeoutException e) {
            throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } catch (RemotingException e) {
            throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```
这段代码的主要逻辑是
- 获取client对象
- 根据有无返回值判断调用方式是否是oneway
- oneway通过client发起请求，返回一个异步执行结果的返回值
- 非oneway则获取回调线程池，发送请求，返回一个Future对象。


### 服务端调用代码
客户端默认是通过Netty进行发起请求调用,对于服务端主要是通过NettyServerHandler#channelRead方法进行接收消息

```java
@Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
        handler.received(channel, msg);
    }
```

服务端接收消息默认是所有消息派发至业务线程池，也就是AllChannelHandler#received

```java
@Override
    public void received(Channel channel, Object message) throws RemotingException {
        ExecutorService executor = getPreferredExecutorService(message);
        try {
            executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
        } catch (Throwable t) {
        	if(message instanceof Request && t instanceof RejectedExecutionException){
                sendFeedback(channel, (Request) message, t);
                return;
        	}
            throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
        }
    }
```
这里将消息封装成ChannelEventRunnable扔到业务线程池执行。接下来会将消息解码后调用到HeaderExchangeHandler#handleRequest


```java
void handleRequest(final ExchangeChannel channel, Request req) throws RemotingException {
        Response res = new Response(req.getId(), req.getVersion());
        if (req.isBroken()) {
            Object data = req.getData();

            String msg;
            if (data == null) {
                msg = null;
            } else if (data instanceof Throwable) {
                msg = StringUtils.toString((Throwable) data);
            } else {
                msg = data.toString();
            }
            res.setErrorMessage("Fail to decode request due to: " + msg);
            res.setStatus(Response.BAD_REQUEST);

            channel.send(res);
            return;
        }
        // find handler by message class.
        Object msg = req.getData();
        try {
            // 执行方法
            CompletionStage<Object> future = handler.reply(channel, msg);
            future.whenComplete((appResult, t) -> {
                try {
                    if (t == null) {
                        res.setStatus(Response.OK);
                        res.setResult(appResult);
                    } else {
                        res.setStatus(Response.SERVICE_ERROR);
                        res.setErrorMessage(StringUtils.toString(t));
                    }
                    channel.send(res);
                } catch (RemotingException e) {
                    logger.warn("Send result to consumer failed, channel is " + channel + ", msg is " + e);
                }
            });
        } catch (Throwable e) {
            res.setStatus(Response.SERVICE_ERROR);
            res.setErrorMessage(StringUtils.toString(e));
            channel.send(res);
        }
    }
```
这里调用具体的handler执行reply方法，最终调用到DubboProtocol中ExchangeHandler的实现
```java
@Override
        public CompletableFuture<Object> reply(ExchangeChannel channel, Object message) throws RemotingException {

            if (!(message instanceof Invocation)) {
                throw new RemotingException(channel, "Unsupported request: "
                        + (message == null ? null : (message.getClass().getName() + ": " + message))
                        + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
            }

            Invocation inv = (Invocation) message;
            Invoker<?> invoker = getInvoker(channel, inv);
            // need to consider backward-compatibility if it's a callback
            if (Boolean.TRUE.toString().equals(inv.getObjectAttachments().get(IS_CALLBACK_SERVICE_INVOKE))) {
                String methodsStr = invoker.getUrl().getParameters().get("methods");
                boolean hasMethod = false;
                if (methodsStr == null || !methodsStr.contains(",")) {
                    hasMethod = inv.getMethodName().equals(methodsStr);
                } else {
                    String[] methods = methodsStr.split(",");
                    for (String method : methods) {
                        if (inv.getMethodName().equals(method)) {
                            hasMethod = true;
                            break;
                        }
                    }
                }
                if (!hasMethod) {
                    logger.warn(new IllegalStateException("The methodName " + inv.getMethodName()
                            + " not found in callback service interface ,invoke will be ignored."
                            + " please update the api interface. url is:"
                            + invoker.getUrl()) + " ,invocation is :" + inv);
                    return null;
                }
            }
            RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
            Result result = invoker.invoke(inv);
            return result.thenApply(Function.identity());
        }
```
这里就是获取具体的Invoker对象执行方法调用

## 三、总结
客户端通过接口调用某个方法，实际调用到代理类。代理类从cluster中获取Invoker对象，并进行router进行过滤。紧接着通过loadBalance进行负载均衡。获取到Invoker对象后会根据协议构造请求头，然后将参数序列化后构造回请求体，最后通过Client进行远程调用。


服务端通过NettryServer监听请求，根据协议反序列化成对象，再按照派发策略派发消息。默认是All，也就是所有请求扔给业务线程池。业务线程会获取Invoker对象，并调用真实类，最终将结果返回。

