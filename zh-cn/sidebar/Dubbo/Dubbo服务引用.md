[toc]

----

## 前言

![image.png](https://dubbo.apache.org/imgs/dev/dubbo-relation.jpg)

在服务导出过程中，provider将自己的服务暴露了出来，并注册到注册中心中。对于服务引用无非就是从注册中心中订阅服务Provider的地址，然后封装一个调用类进行调用。

## 源码分析
### 服务引用时机
Dubbo服务引用主要有两种方式，饿汉式和懒汉式。

如果Reference中配置了init = true，引用的方式就为饿汉式，即在ReferenceBean创建完毕，通过afterPropertiesSet方法进行初始化

org.apache.dubbo.config.spring.ReferenceBean#afterPropertiesSet
```java
public void afterPropertiesSet() throws Exception {

        // Initializes Dubbo's Config Beans before @Reference bean autowiring
        prepareDubboConfigBeans();

        // lazy init by default.
        if (init == null) {
            init = false;
        }

        // eager init if necessary.
        // 根据init配置决定是否进行服务引用
        if (shouldInit()) {
            getObject();
        }
    }
    
    @Override
    public Object getObject() {
        return get();
    }
    
    public synchronized T get() {
        if (destroyed) {
            throw new IllegalStateException("The invoker of ReferenceConfig(" + url + ") has already destroyed!");
        }
        if (ref == null) {
            init();
        }
        return ref;
    }
```

懒汉式初始化主要是实现FactoryBean的getObject方法，在获取Bean的时候会调用该方法进行初始化
```java
// 实现FactoryBean需要实现的方法，在真正要获取Bean的时候容器会调用该方法
    @Override
    public Object getObject() {
        return get();
    }
    public synchronized T get() {
        if (destroyed) {
            throw new IllegalStateException("The invoker of ReferenceConfig(" + url + ") has already destroyed!");
        }
        // 引用为空进行初始化
        if (ref == null) {
            init();
        }
        return ref;
    }
```

### org.apache.dubbo.config.ReferenceConfig#init
找到入口后就开始看服务引用的初始化方法

```java
public synchronized void init() {
        if (initialized) {
            return;
        }

        if (bootstrap == null) {
            bootstrap = DubboBootstrap.getInstance();
            bootstrap.init();
        }

        checkAndUpdateSubConfigs();

        checkStubAndLocal(interfaceClass);
        ConfigValidationUtils.checkMock(interfaceClass, this);

        Map<String, String> map = new HashMap<String, String>();
        // 省略代码，将配置信息填充至MAP

        // 创建代理对象
        ref = createProxy(map);

        serviceMetadata.setTarget(ref);
        serviceMetadata.addAttribute(PROXY_CLASS_REF, ref);
        ConsumerModel consumerModel = repository.lookupReferredService(serviceMetadata.getServiceKey());
        consumerModel.setProxyObject(ref);
        consumerModel.init(attributes);

        initialized = true;

        // dispatch a ReferenceConfigInitializedEvent since 2.7.4
        dispatch(new ReferenceConfigInitializedEvent(this, invoker));
    }
```
该方法逻辑较长，整体的逻辑主要是
- 检查并填充默认配置
- 将配置填充至Map
- 创建代理对象作为ReferenceBean的ref
- 分发服务引用完成事件

![Map填充字段.png](https://upload-images.jianshu.io/upload_images/14607771-3e0adef20edb663e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们重点关注代理对象怎么创建，也就是createProxy方法

### org.apache.dubbo.config.ReferenceConfig#createProxy

```java
private T createProxy(Map<String, String> map) {
        // 判断是否进行本地引用
        if (shouldJvmRefer(map)) {
            URL url = new URL(LOCAL_PROTOCOL, LOCALHOST_VALUE, 0, interfaceClass.getName()).addParameters(map);
            // 创建InjvmInvoker实例
            invoker = REF_PROTOCOL.refer(interfaceClass, url);
            if (logger.isInfoEnabled()) {
                logger.info("Using injvm service " + interfaceClass.getName());
            }
        // 远程引用
        } else {
            urls.clear();
            // 配置了URL信息可能为点对点服务直连或者注册中心地址
            if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
                String[] us = SEMICOLON_SPLIT_PATTERN.split(url);
                if (us != null && us.length > 0) {
                    for (String u : us) {
                        URL url = URL.valueOf(u);
                        if (StringUtils.isEmpty(url.getPath())) {
                            url = url.setPath(interfaceName);
                        }
                        if (UrlUtils.isRegistry(url)) {
                            urls.add(url.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                        } else {
                            urls.add(ClusterUtils.mergeUrl(url, map));
                        }
                    }
                }
            // 没有配置URL 走的是注册中心
            } else { // assemble URL from register center's configuration
                // if protocols not injvm checkRegistry
                if (!LOCAL_PROTOCOL.equalsIgnoreCase(getProtocol())) {
                    checkRegistry();
                    List<URL> us = ConfigValidationUtils.loadRegistries(this, false);
                    if (CollectionUtils.isNotEmpty(us)) {
                        for (URL u : us) {
                            URL monitorUrl = ConfigValidationUtils.loadMonitor(this, u);
                            if (monitorUrl != null) {
                                map.put(MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                            }
                            urls.add(u.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                        }
                    }
                    if (urls.isEmpty()) {
                        throw new IllegalStateException("No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
                    }
                }
            }

            if (urls.size() == 1) {
                invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
            } else {
                List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
                URL registryURL = null;
                for (URL url : urls) {
                    invokers.add(REF_PROTOCOL.refer(interfaceClass, url));
                    if (UrlUtils.isRegistry(url)) {
                        registryURL = url; // use last registry url
                    }
                }
                if (registryURL != null) { // registry url is available
                    // for multi-subscription scenario, use 'zone-aware' policy by default
                    URL u = registryURL.addParameterIfAbsent(CLUSTER_KEY, ZoneAwareCluster.NAME);
                    // The invoker wrap relation would be like: ZoneAwareClusterInvoker(StaticDirectory) -> FailoverClusterInvoker(RegistryDirectory, routing happens here) -> Invoker
                    invoker = CLUSTER.join(new StaticDirectory(u, invokers));
                } else { // not a registry url, must be direct invoke.
                    invoker = CLUSTER.join(new StaticDirectory(invokers));
                }
            }
        }
```
createProxy主要逻辑有：
- 判断是否是本地引用，如果是则获取内存中的Exporter创建Invoker对象
- 如果有配置URL，可能配置了点对点或者注册中心的地址将URL添加至List
- 没有配置URL则获取注册中心URL
- 通过SPI调用对应的Protocal#refer方法

### org.apache.dubbo.registry.integration.RegistryProtocol#refer
```java
  @Override
  @SuppressWarnings("unchecked")
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        url = getRegistryUrl(url);
        // 获取Register实例
        Registry registry = registryFactory.getRegistry(url);
        if (RegistryService.class.equals(type)) {
            return proxyFactory.getInvoker((T) registry, type, url);
        }

        // group="a,b" or group="*"
        Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(REFER_KEY));
        String group = qs.get(GROUP_KEY);
        if (group != null && group.length() > 0) {
            if ((COMMA_SPLIT_PATTERN.split(group)).length > 1 || "*".equals(group)) {
                return doRefer(getMergeableCluster(), registry, type, url);
            }
        }
        // 执行导出
        return doRefer(cluster, registry, type, url);
    }
```
主要逻辑：
- 获取注册中心实例（ZookeeperRegistry）
- 调用doRefer进行真正的refer

查看doRefer方法
```java
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
        directory.setRegistry(registry);
        directory.setProtocol(protocol);
        // all attributes of REFER_KEY
        Map<String, String> parameters = new HashMap<String, String>(directory.getConsumerUrl().getParameters());
        // 生成服务消费者URL
        // consumer://127.0.0.1/org.apache.dubbo.demo.DemoService?application=demo-consumer&check=false&dubbo=2.0.2&init=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello,sayHelloAsync&pid=53921&qos.port=33333&side=consumer&sticky=false&timestamp=1631455798609
        URL subscribeUrl = new URL(CONSUMER_PROTOCOL, parameters.remove(REGISTER_IP_KEY), 0, type.getName(), parameters);
        // 向注册中心注册消费者 consumer目录创建新节点
        if (directory.isShouldRegister()) {
            directory.setRegisteredConsumerUrl(subscribeUrl);
            registry.register(directory.getRegisteredConsumerUrl());
        }
        directory.buildRouterChain(subscribeUrl);
        // 订阅目录providers、configurators、routers目录，订阅好后会触发DubboProtocol的refer方法
        directory.subscribe(toSubscribeUrl(subscribeUrl));

        // 利用cluster封装directory 本质是封装多个Invoker
        Invoker<T> invoker = cluster.join(directory);
        List<RegistryProtocolListener> listeners = findRegistryProtocolListeners(url);
        if (CollectionUtils.isEmpty(listeners)) {
            return invoker;
        }

        RegistryInvokerWrapper<T> registryInvokerWrapper = new RegistryInvokerWrapper<>(directory, cluster, invoker, subscribeUrl);
        for (RegistryProtocolListener listener : listeners) {
            listener.onRefer(this, registryInvokerWrapper);
        }
        return registryInvokerWrapper;
    }
```

主要逻辑：
- 生成RegistryDirectory对象
- 生成消费者的URL
- 消费者注册到注册中心
- 订阅目录providers、configurators、routers目录，订阅好后会触发DubboProtocol的refer方法
- 利用cluster封装多个Invoker返回

在完成订阅后，RegisteryDirectory会执行notify方法，最终此处会调用DubboProtocol的refer方法

![image.png](https://upload-images.jianshu.io/upload_images/14607771-f4d4ebd137694af2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol#protocolBindingRefer

```java
@Override
    public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {
        optimizeSerialization(url);

        // create rpc invoker.
        DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
        invokers.add(invoker);

        return invoker;
    }
```

这里最终会创建一个DubboInvoker对象。既然是远程过程调用，必然涉及到远程连接，我们接着看这里的getClient就是根据注册中心的地址创建NettyClient连接。不再详细展开。


## 总结
Dubbo服务引用主要有两种方式，饿汉式和懒汉式。饿汉式是在Bean初始化的时候进行引用。懒汉式则是在Bean首次注入过程中引用。

服务引用过程中，会通过配置组装URL并决定服务引用的方式，通常有本地引用、远程直连和注册中心引用。

以注册中心引用为例，首先会创建一个RegisterDirectory，然后会向注册中心注册自己，接下来会去订阅Provider信息。订阅完成后，会根据Provider服务提供的协议进行引用。以DubboProtocol为例，会创建Invoker对象并创建netty client通过其进行网络连接。最后会通过cluster合并Invoker对象。

