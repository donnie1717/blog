[toc]

----

## 前言

![image.png](https://dubbo.apache.org/imgs/dev/dubbo-relation.jpg)

Dubbo服务启动过程中伴随着服务注册的过程，也就是服务导出。本篇文章主要是记录一下Dubbo的服务导出过程。Dubbo服务导出开始于Spring容器发布刷新事件。注：本篇文章选取的源代码版本是2.7.6。


## 源码分析

 ### DubboBootstrapApplicationListener#onApplicationContextEvent
```java
    @Override
    public void onApplicationContextEvent(ApplicationContextEvent event) {
        if (event instanceof ContextRefreshedEvent) {
            onContextRefreshedEvent((ContextRefreshedEvent) event);
        } else if (event instanceof ContextClosedEvent) {
            onContextClosedEvent((ContextClosedEvent) event);
        }
    }
    private void onContextRefreshedEvent(ContextRefreshedEvent event) {
        dubboBootstrap.start();
    }
```
这段代码的主要逻辑是在Spring容器启动或者刷新执行Dubbo初始化。

- Spring容器启动或者刷新发布ContextRefreshedEvent
- Dubbo通过监听该事件执行DubboBootstrap#start方法

### DubboBootstrap#start

```java
    /**
     * Start the bootstrap
     */
    public DubboBootstrap start() {
        if (started.compareAndSet(false, true)) {
            initialize();
            if (logger.isInfoEnabled()) {
                logger.info(NAME + " is starting...");
            }
            // 1. export Dubbo Services
            exportServices();

            // Not only provider register
            if (!isOnlyRegisterProvider() || hasExportedServices()) {
                // 2. export MetadataService
                exportMetadataService();
                //3. Register the local ServiceInstance if required
                registerServiceInstance();
            }

            referServices();

            if (logger.isInfoEnabled()) {
                logger.info(NAME + " has started.");
            }
        }
        return this;
    }
    
    private void exportServices() {
        configManager.getServices().forEach(sc -> {
            // TODO, compatible with ServiceConfig.export()
            ServiceConfig serviceConfig = (ServiceConfig) sc;
            serviceConfig.setBootstrap(this);

            if (exportAsync) {
                ExecutorService executor = executorRepository.getServiceExporterExecutor();
                Future<?> future = executor.submit(() -> {
                    sc.export();
                });
                asyncExportingFutures.add(future);
            } else {
                sc.export();
                exportedServices.add(sc);
            }
        });
    }
```
这个方法主要是做一些初始化，我们重点关注exportServices这个服务导出的方法。该方法的逻辑也很简单，主要是根据开关选择同步导出还是异步导出，最终核心方法都指向了ServiceConfig#export

### ServiceConfig#export
```java
public synchronized void export() {
        if (!shouldExport()) {
            return;
        }

        if (bootstrap == null) {
            bootstrap = DubboBootstrap.getInstance();
            bootstrap.init();
        }

        // 检查配置
        checkAndUpdateSubConfigs();

        //init serviceMetadata
        serviceMetadata.setVersion(version);
        serviceMetadata.setGroup(group);
        serviceMetadata.setDefaultGroup(group);
        serviceMetadata.setServiceType(getInterfaceClass());
        serviceMetadata.setServiceInterfaceName(getInterface());
        serviceMetadata.setTarget(getRef());

        // 判断是否需要延迟暴露服务
        if (shouldDelay()) {
            DELAY_EXPORT_EXECUTOR.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
        } else {
            // 执行服务导出
            doExport();
        }
        // 服务导出成功业务逻辑处理
        // 分发服务成功导出事件
        exported();
    }
```
主要业务逻辑：
- 检查配置，将未填写的配置填充默认值
- 初始化Service的原数据
- 根据参数判断是否需要延迟暴露服务
- 执行服务导出
- 服务导出后的业务逻辑处理，主要是分发事件


### ServiceConfig#doExport
```java
protected synchronized void doExport() {
        if (unexported) {
            throw new IllegalStateException("The service " + interfaceClass.getName() + " has already unexported!");
        }
        if (exported) {
            return;
        }
        exported = true;

        if (StringUtils.isEmpty(path)) {
            path = interfaceName;
        }
        // 导出服务
        doExportUrls();
    }
```
该方法没有多少逻辑，主要是判断是否需要导出服务。`<dubbo:provider>`提供了参数可以取消导出服务用于本地调试。
```xml
<dubbo:service interface="org.apache.dubbo.demo.DemoService" ref="demoService" export="false"/>
```

### ServiceConfig#doExportUrls
```java
private void doExportUrls() {
        // 将当前的service添加到ServiceRepository
        ServiceRepository repository = ApplicationModel.getServiceRepository();
        ServiceDescriptor serviceDescriptor = repository.registerService(getInterfaceClass());
        repository.registerProvider(
                getUniqueServiceName(),
                ref,
                serviceDescriptor,
                this,
                serviceMetadata
        );

        // 加载注册中心链接
        List<URL> registryURLs = ConfigValidationUtils.loadRegistries(this, true);

        // 遍历 protocols，并在每个协议下导出服务
        for (ProtocolConfig protocolConfig : protocols) {
            String pathKey = URL.buildKey(getContextPath(protocolConfig)
                    .map(p -> p + "/" + path)
                    .orElse(path), group, version);
            // In case user specified path, register service one more time to map it to path.
            repository.registerService(pathKey, interfaceClass);
            // TODO, uncomment this line once service key is unified
            serviceMetadata.setServiceKey(pathKey);
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
    }
```
主要逻辑：
- 将当前的Service信息添加至ServiceRepository，(ApplicationModel保存着服务提供者和调用者的基本信息)
- 加载注册中心的链接（根据用户配置将注册中心的地址转化为URL）
- 遍历protocols，并在每个写一下导出服务

### ServiceConfig#doExportUrlsFor1Protocol

```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
        

        Map<String, String> map = new HashMap<String, String>();
        // 省略代码 主要是将ProtocolConfig中的信息添加至Map中，用于构造URL
        
        // export service
        String host = findConfigedHosts(protocolConfig, registryURLs, map);
        Integer port = findConfigedPorts(protocolConfig, name, map);
        // 构造URL
        URL url = new URL(name, host, port, getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), map);
        // 省略代码
        }
```
该方法主要是用于构造协议的URL，主要逻是将一些信息及配置对象字段放在Map中，Map中的内容将传递给URL对象。（上面代码中为快速理解整体流程，省略了具体的参数获取源码）

**URL对象内容大致如下**
```
dubbo://127.0.0.1:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=127.0.0.1&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=org.apache.dubbo.demo.DemoService&metadata-type=remote&methods=sayHello,sayHelloAsync&pid=3983&qos.port=22222&release=&side=provider&timestamp=1630134753068
```

构造好URL对象，接下来开始服务导出相关代码

```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    String scope = url.getParameter(SCOPE_KEY);
        // don't export when none is configured
        if (!SCOPE_NONE.equalsIgnoreCase(scope)) {

            // export to local if the config is not remote (export to remote only when config is remote)
            if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) {
                exportLocal(url);
            }
            // export to remote if the config is not local (export to local only when config is local)
            if (!SCOPE_LOCAL.equalsIgnoreCase(scope)) {
                if (CollectionUtils.isNotEmpty(registryURLs)) {
                    for (URL registryURL : registryURLs) {
                        //if protocol is only injvm ,not register
                        if (LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
                            continue;
                        }
                        url = url.addParameterIfAbsent(DYNAMIC_KEY, registryURL.getParameter(DYNAMIC_KEY));
                        URL monitorUrl = ConfigValidationUtils.loadMonitor(this, registryURL);
                        if (monitorUrl != null) {
                            url = url.addParameterAndEncoded(MONITOR_KEY, monitorUrl.toFullString());
                        }
                        if (logger.isInfoEnabled()) {
                            if (url.getParameter(REGISTER_KEY, true)) {
                                logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                            } else {
                                logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                            }
                        }

                        // For providers, this is used to enable custom proxy to generate invoker
                        String proxy = url.getParameter(PROXY_KEY);
                        if (StringUtils.isNotEmpty(proxy)) {
                            registryURL = registryURL.addParameter(PROXY_KEY, proxy);
                        }

                        Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
                        DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                        Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);
                        exporters.add(exporter);
                    }
                    // 不存在注册中心则仅导出本地服务
                } else {
                    if (logger.isInfoEnabled()) {
                        logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                    }
                    Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, url);
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                    Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);
                    exporters.add(exporter);
                }
                /**
                 * @since 2.7.0
                 * ServiceData Store
                 */
                WritableMetadataService metadataService = WritableMetadataService.getExtension(url.getParameter(METADATA_KEY, DEFAULT_METADATA_STORAGE_TYPE));
                if (metadataService != null) {
                    metadataService.publishServiceDefinition(url);
                }
            }
        }
        this.urls.add(url);

}
```
这段代码的主要逻辑如下：
- 获取scope参数
- scope参数如果为none则不进行导出
- scope != remote 导出到本地
- scopre != local 导出到远程


接下来看一下导出服务到本地的方法
```java
private void exportLocal(URL url) {
        // 构建URL,协议头为injvm
        URL local = URLBuilder.from(url)
                .setProtocol(LOCAL_PROTOCOL)
                .setHost(LOCALHOST_VALUE)
                .setPort(0)
                .build();
        // 创建Invoker 并调用InjvmProtocol.export方法
        Exporter<?> exporter = PROTOCOL.export(
                PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, local));
        exporters.add(exporter);
        logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry url : " + local);
    }
```
主要逻辑：
- 构造injvm协议头的URL
- 创建 Invoker并调用InjvmProtocol.export方法

接下来看一下InjvmProtocol.export方法
```java
@Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        // 创建InjvmExporter
        return new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);
    }
```
这边逻辑较为简单，创建InjvmExporter，并将该Exporter添加至Map中，key为服务名

接下来看一下服务导出到远程的方法,这边我们直接看RegistryProtocol#export方法

```java
@Override
    public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        // 获取注册中心的URL 示例如下
        // zookeeper://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.2&export=dubbo%3A%2F%2F127.0.0.1%3A20880%2Forg.apache.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddemo-provider%26bind.ip%3D127.0.0.1%26bind.port%3D20880%26deprecated%3Dfalse%26dubbo%3D2.0.2%26dynamic%3Dtrue%26generic%3Dfalse%26interface%3Dorg.apache.dubbo.demo.DemoService%26metadata-type%3Dremote%26methods%3DsayHello%2CsayHelloAsync%26pid%3D4292%26qos.port%3D22222%26release%3D%26side%3Dprovider%26timestamp%3D1630136961051&metadata-type=remote&pid=4292&qos.port=22222&timestamp=1630136958853
        URL registryUrl = getRegistryUrl(originInvoker);
        // url to export locally
        // 获取服务提供者的URL
        // dubbo://127.0.0.1:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=127.0.0.1&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=org.apache.dubbo.demo.DemoService&metadata-type=remote&methods=sayHello,sayHelloAsync&pid=4292&qos.port=22222&release=&side=provider&timestamp=1630136961051
        URL providerUrl = getProviderUrl(originInvoker);

        // Subscribe the override data
        // FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call
        //  the same service. Because the subscribed is cached key with the name of the service, it causes the
        //  subscription information to cover.
        // 获取订阅URL
        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
        // 创建监听器
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
        overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);

        providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);
        //export invoker
        // 导出服务
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);

        // url to registry
        final Registry registry = getRegistry(originInvoker);
        final URL registeredProviderUrl = getUrlToRegistry(providerUrl, registryUrl);
        // decide if we need to delay publish
        // 判断是否要注册服务
        boolean register = providerUrl.getParameter(REGISTER_KEY, true);
        if (register) {
            // 服务注册
            register(registryUrl, registeredProviderUrl);
        }

        // Deprecated! Subscribe to override rules in 2.6.x or before.
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

        exporter.setRegisterUrl(registeredProviderUrl);
        exporter.setSubscribeUrl(overrideSubscribeUrl);

        notifyExport(exporter);
        //Ensure that a new exporter instance is returned every time export
        return new DestroyableExporter<>(exporter);
    }
```
主要业务逻辑：
- 获取注册中心URL
- 获取服务提供者的URL
- 获取订阅URL
- 创建订阅监听器
- 导出服务
- 判断是否需要服务注册，如果需要则进行服务注册

该方法中主要有两个导出服务（org.apache.dubbo.registry.integration.RegistryProtocol#doLocalExport）和注册服务（org.apache.dubbo.registry.integration.RegistryProtocol#register）

先看一下导出服务org.apache.dubbo.registry.integration.RegistryProtocol#doLocalExport方法
```java
private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker, URL providerUrl) {
        String key = getCacheKey(originInvoker);

        // 创建一个Exporter对象
        return (ExporterChangeableWrapper<T>) bounds.computeIfAbsent(key, s -> {
            Invoker<?> invokerDelegate = new InvokerDelegate<>(originInvoker, providerUrl);
            // 通过Dubbo协议进行导出得到一个exporter对象
            return new ExporterChangeableWrapper<>((Exporter<T>) protocol.export(invokerDelegate), originInvoker);
        });
    }
```
org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol#export

```java
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        // 获取URL
        URL url = invoker.getUrl();

        // export service. key=org.apache.dubbo.demo.DemoService:20880
        String key = serviceKey(url);
        // 创建exporter存入缓存中
        DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
        exporterMap.put(key, exporter);

        //export an stub service for dispatching event
        Boolean isStubSupportEvent = url.getParameter(STUB_EVENT_KEY, DEFAULT_STUB_EVENT);
        Boolean isCallbackservice = url.getParameter(IS_CALLBACK_SERVICE, false);
        if (isStubSupportEvent && !isCallbackservice) {
            String stubServiceMethods = url.getParameter(STUB_EVENT_METHODS_KEY);
            if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
                if (logger.isWarnEnabled()) {
                    logger.warn(new IllegalStateException("consumer [" + url.getParameter(INTERFACE_KEY) +
                            "], has set stubproxy support event ,but no stub methods founded."));
                }

            }
        }
        // 创建监听服务器 默认是NettyServer
        openServer(url);
        optimizeSerialization(url);

        return exporter;
    }
```
主要逻辑：
- 获取URL创建Exporter对象
- 将export对象存入缓存
- 首次导出创建监听服务器


接下来看一下服务怎么注册到注册中心上（Zookeeper）

org.apache.dubbo.registry.integration.RegistryProtocol#register
```java
public void register(URL registryUrl, URL registeredProviderUrl) {
        // 获取Registry实例
        Registry registry = registryFactory.getRegistry(registryUrl);
        registry.register(registeredProviderUrl);

        ProviderModel model = ApplicationModel.getProviderModel(registeredProviderUrl.getServiceKey());
        model.addStatedUrl(new ProviderModel.RegisterStatedURL(
                registeredProviderUrl,
                registryUrl,
                true
        ));
    }
```
最终会调用到ZookeeperRegistry的doRegister方法，在Zookeeper中创建节点。
```java
@Override
    public void doRegister(URL url) {
        try {
            zkClient.create(toUrlPath(url), url.getParameter(DYNAMIC_KEY, true));
        } catch (Throwable e) {
            throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```
最终我们连上Zookeeper查看

![image.png](https://upload-images.jianshu.io/upload_images/14607771-f192797255b213e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 总结
最终用一张图来过一下Dubbo服务导出的流程图

![image.png](https://upload-images.jianshu.io/upload_images/14607771-e1e22387d188097d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


服务导出始于Spring容器刷新完毕，导出过程主要是根据配置获取服务提供者的URL，注册中心URL。

导出前会使用ProxyFactory#getInvoker生成服务提供者的代理对象。然后根据注册中心的URL执行注册。注册过程主要是调用服务提供者的URL 进行对应的协议导出。

以DubboProtocol为例，首次导出会建立NettyServer等待客户端连接。最终返回一个Exporter对象存放至缓存中。

导出完成后会将URL注册到注册中心中。



