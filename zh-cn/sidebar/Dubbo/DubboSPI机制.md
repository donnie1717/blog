[toc]

----

## 1.前言
 SPI是一种服务发现机制，本质是将接口实现类的全限定名配置在文件中，由服务加载器读取配置文件，加载实现类。Dubbo源码中有大量的SPI实现。所以阅读Dubbo源码前有必要了解一下SPI机制的实现。本篇文章主要是记录一下Dubbo SPI机制的基本使用及部分源码。

## 2.JDK SPI机制
Java为我们提供了SPI机制的一种实现，具体的实现步骤如下。

### 使用示例
1.自定接口com.dqzhou.starter.spi.Robot
```java
public interface Robot {
    void sayHello();
}
```
2.实现接口com.dqzhou.starter.spi.Bumblebee
```java
public class Bumblebee implements Robot {
    @Override
    public void sayHello() {
        System.out.println("Hello, I am Bumblebee.");
    }
}
```
com.dqzhou.starter.spi.OptimusPrime
```java
public class OptimusPrime implements Robot {
    @Override
    public void sayHello() {
        System.out.println("Hello, I am Optimus Prime.");
    }
}
```
3.在META-INF/services文件夹下创建一个Robot权限限定名（com.dqzhou.starter.spi.Robot）文件。
文件内容为实现类名称。
```
com.dqzhou.starter.spi.OptimusPrime
com.dqzhou.starter.spi.Bumblebee
```
4.接口调用
```java
public class JavaDemo {
    public static void main(String[] args) {
        ServiceLoader<Robot> serviceLoader = ServiceLoader.load(Robot.class);
        System.out.println("Java SPI");
        serviceLoader.forEach(Robot::sayHello);
    }
}
```

## 3.Dubbo SPI
### 使用示例
1.自定义接口及实现类
```java
// 需要对测试接口使用@SPI声明
@SPI
public interface Robot {
    void sayHello();
}
public class Bumblebee implements Robot {
    @Override
    public void sayHello() {
        System.out.println("Hello, I am Bumblebee.");
    }
}
public class OptimusPrime implements Robot {
    @Override
    public void sayHello() {
        System.out.println("Hello, I am Optimus Prime.");
    }
}
```
2.在META-INF/dubbo创建Robot的全限定名文件，内容如下：
```
optimusPrime = org.apache.dubbo.spi.impl.OptimusPrime
bumblebee = org.apache.dubbo.spi.impl.Bumblebee
```
3.编写测试类
```java
public class DubboSpiTest {
    @Test
    public void testSayHello() {
        ExtensionLoader<Robot> extensionLoader = ExtensionLoader.getExtensionLoader(Robot.class);
        Robot optimusPrime = extensionLoader.getExtension("optimusPrime");
        optimusPrime.sayHello();
        Robot bumblebee = extensionLoader.getExtension("bumblebee");
        bumblebee.sayHello();
    }
}
```

### Dubbo为什么不使用Java SPI?
- JDK的SPI只能通过遍历来查找所有的扩展点，可能一次性加载所有的扩展点，就会造成资源浪费
- Java SPI机制较为简单，不支持IOC、AOP等特性

### SPI机制源码分析
**ExtensionLoader.getExtensionLoader：根据扩展点接口来获取扩展加载器**
上一节简单的看了下Dubbo SPI是怎么使用的，接下来直接看一下ExtensionLoader.getExtensionLoader。

```java
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        // 参数校验
        if (type == null) {
            throw new IllegalArgumentException("Extension type == null");
        }
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
        }
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type (" + type +
                    ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
        }
        // 从内存中获取ExtensionLoader
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        // 不存在则创建ExtensionLoader
        if (loader == null) {
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
```
该方法的逻辑不复杂，主要是为了获取该接口的的ExtensionLoader
- 参数校验（非空、接口、标识@SPI注解）
- 从内存中获取该接口的ExtensionLoader
- 如果获取不到则创建一个默认ExtensionLoader返回

**ExtensionLoader.getExtension:根据扩展名获取扩展对象**

接下来我们从ExtensionLoader.getExtension方法作为入口，对扩展对象的获取进行分析。
```java
public T getExtension(String name) {
        // 参数校验
        if (StringUtils.isEmpty(name)) {
            throw new IllegalArgumentException("Extension name == null");
        }
        // name为true返回默认扩展对象
        if ("true".equals(name)) {
            return getDefaultExtension();
        }
        // 创建一个Holder
        final Holder<Object> holder = getOrCreateHolder(name);
        // 从Holder中获取对象，不存在则加载
        Object instance = holder.get();
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    // 创建一个扩展对象
                    instance = createExtension(name);
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }
```
方法主要逻辑：
- 参数校验
- 从Holder对象中直接获取
- 不存在则创建扩展对象

**createExtension:根据扩展名创建扩展接口实现类对象**

接下来看一下createExtension方法的逻辑
```java
private T createExtension(String name) {
        // 从配置文件中加载所有的扩展类
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            // 通过反射创建实例
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
 // 向实例中注入依赖   
 injectExtension(instance);
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (CollectionUtils.isNotEmpty(wrapperClasses)) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            initExtension(instance);
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                    type + ") couldn't be instantiated: " + t.getMessage(), t);
        }
    }
```
该方法的主要逻辑如下
- getExtensionClasses加载所有的扩展类，Map<String, Class<?>>
- 通过反射创建实例
- 向实例注入依赖
- 将扩展对象包裹在相应的Wrapper对象


**加载所有的扩展类**
直接看getExtensionClasses方法
```java
private Map<String, Class<?>> getExtensionClasses() {
        // 从内存中取
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                // 取不到则进行加载，并设置到内存中
                if (classes == null) {
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
```
主要逻辑：
- 从缓存中获取
- 取不到则双重检查同步调用loadExtensionClasses方法加载

```java
private Map<String, Class<?>> loadExtensionClasses() {
        // 从SPI注解中缓存默认扩展实例名称
        cacheDefaultExtensionName();

        Map<String, Class<?>> extensionClasses = new HashMap<>();
        // 从不同的文件路径中读取配置文件
        for (LoadingStrategy strategy : strategies) {
      loadDirectory(extensionClasses, strategy.directory(), type.getName(), strategy.preferExtensionClassLoader(), strategy.excludedPackages());
            loadDirectory(extensionClasses, strategy.directory(), type.getName().replace("org.apache", "com.alibaba"), strategy.preferExtensionClassLoader(), strategy.excludedPackages());
        }

        return extensionClasses;
    }
```
**injectExtension:依赖注入**
```java
private T injectExtension(T instance) {

        if (objectFactory == null) {
            return instance;
        }

        try {
            // 反射获取该类中的所有方法
            for (Method method : instance.getClass().getMethods()) {
                if (!isSetter(method)) {
                    continue;
                }
                /**
                 * Check {@link DisableInject} to see if we need auto injection for this property
                 */
                if (method.getAnnotation(DisableInject.class) != null) {
                    continue;
                }
                Class<?> pt = method.getParameterTypes()[0];
                if (ReflectUtils.isPrimitives(pt)) {
                    continue;
                }

                try {
                    String property = getSetterProperty(method);
                    Object object = objectFactory.getExtension(pt, property);
                    if (object != null) {
                        method.invoke(instance, object);
                    }
                } catch (Exception e) {
                    logger.error("Failed to inject via method " + method.getName()
                            + " of interface " + type.getName() + ": " + e.getMessage(), e);
                }

            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```
主要逻辑：
- 通过反射获取类中的所有方法
- 找到set方法
- 找到需要依赖注入的属性，然后将对象注入进去。

## 4.扩展点自动包装（AOP）

### 使用示例
创建一个Bumblebee的Wapper类，实现目标接口，并提供构造函数供真实类注入。
```java
public class RobotWrapper implements Robot {

    Robot robot;

    public RobotWrapper(Robot robot) {
        this.robot = robot;
    }

    @Override
    public void sayHello() {
        System.out.println("before say hello");
        robot.sayHello();
        System.out.println("after say hello");
    }
}
```
在META-INF/services接口的全限定名称文件中将Wrapper配置
```
optimusPrime = org.apache.dubbo.spi.impl.OptimusPrime
bumblebee = org.apache.dubbo.spi.impl.Bumblebee
robotWrapper = org.apache.dubbo.spi.impl.RobotWrapper∂∂
```
主函数执行调用
```java
public class DubboSpiTest {
    @Test
    public void testSayHello() {
        ExtensionLoader<Robot> extensionLoader = ExtensionLoader.getExtensionLoader(Robot.class);
        Robot optimusPrime = extensionLoader.getExtension("optimusPrime");
        optimusPrime.sayHello();
        Robot bumblebee = extensionLoader.getExtension("bumblebee");
        bumblebee.sayHello();
    }
```
通过debug我们可以发现扩展加载器返回的最终是Wrapper实例
![image.png](https://upload-images.jianshu.io/upload_images/14607771-3fceda3601c1b867.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
输出结果达成了类似AOP的效果
```
before say hello
Hello, I am Optimus Prime.
after say hello
before say hello
Hello, I am Bumblebee.
after say hello
```
### 自动包装源码分析

**createExtension:根据扩展名创建扩展接口实现类对象**

```java
private T createExtension(String name) {
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            injectExtension(instance);
            // 缓存中查找是否加载包装类
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            // 不为空则通过构造方法注入实例
            if (CollectionUtils.isNotEmpty(wrapperClasses)) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            initExtension(instance);
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                    type + ") couldn't be instantiated: " + t.getMessage(), t);
        }
    }
```
主要步骤：
- 从文件加载的时候会根据有无构造方法判断是否是包装类，如果是则缓存包装类信息
- 创建扩展类结束前，如果缓存中包含扩展类信息，则通过反射创建，并通过构造函数注入


## 4.扩展点自动装配及自适应(IOC)

加载扩展点的时候，如果扩展点的类成员变量中为其他扩展点类型。`ExtensionLoader`会利用Setter方法自动注入扩展点

### 使用示例
我们实现两个类，分别是Car和Driver类。在Driver的实现类Trucker中需要注入司机实际开的车。代码如下。

```java
public class Trucker implements Driver {

    private Car car;

    public void setCar(Car car) {
        this.car = car;
    }

    @Override
    public void driveCar() {
        System.out.println("trucker drive ");
        car.getColor();
    }
}
```

在上面代码中，提供了Setter方法供Dubbo注入Car的扩展实现，但是问题来了，如果Car存在多个扩展实现，那么要选择注入哪个呢？

Dubbo针对这种情况引入了自适应机制。

Car接口定义
```java
@SPI
public interface Car {
    @Adaptive(value = "carType")
    void getColor(URL url);
}
```
上面代码中使用`@Adaptive`标识接口为自适应接口，定义了一个变量名为`carType`。当调用者通过`URL`对象指定了`carType`参数来决定调用的实例。

Driver接口定义
```
@SPI
public interface Driver {
    // 指定需要传递URL对象
    void driveCar(URL url);
}
```

Driver的实现类Trucker
```java
public class Trucker implements Driver {

    private Car car;

    public void setCar(Car car) {
        this.car = car;
    }

    @Override
    public void driveCar(URL url) {
        System.out.print("trucker drive ");
        // 传入URL调用Car
        car.getColor(url);
    }
}
```
测试方法
```java
public class DubboSpiTest {
    @Test
    public void testDriveCar() {
        ExtensionLoader<Driver> extensionLoader = ExtensionLoader.getExtensionLoader(Driver.class);
        Driver driver = extensionLoader.getExtension("trucker");
        
        // 构造URL总线，指定carType参数
        Map<String, String> map = new HashMap<>();
        map.put("carType", "redCar");
        URL url = new URL("", "", 0, map);
        
        // 接口调用
        driver.driveCar(url);
    }
}
```
输出结果
```
trucker drive red car
```
