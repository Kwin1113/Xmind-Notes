### ExtensionLoader获取DubboProtocol

ExtensionLoader的经典使用方法

```java
ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(DubboProtocol.NAME);
```

通过静态方法获取（已缓存的）对应扩展类（实际上为接口）的ExtensionLoader扩展点加载器，通过扩展点加载器再进行指定的扩展点进行加载。

以上代码等价于如下：

```java
// 1. 获取Protocol扩展点的ExtensionLoader
ExtensionLoader protocolLoader = ExtensionLoader.getExtensionLoader(Protocol.class);
// 2. 创建给定类型的protocol
protocolLoader.getExtension(DubboProtocol.NAME);
```

#### 1. 获取ExtensionLoader

将两步操作分开来看可能会更加清楚，首先第一步是根据Dubbo SPI机制，获取到扩展点的入口类ExtensionLoader，而ExtensionLoader是基于某个扩展点（接口）的，因此入参是对应的扩展类class。

```java
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        /*
        扩展点合法性判断
        1. 扩展点不能为空（否则扩展啥）
        2. 扩展点应该是接口；SPI的机制
        3. 必须通过SPI注解来确定扩展点真正加载的具体类
         */
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

        // 很自然的使用类为key，加载器为value做缓存，避免重复创建加载器
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            // 通过构造器创建
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
```

大概看一眼构造器里面做了什么初始化操作。

```java
    private ExtensionLoader(Class<?> type) {
        this.type = type;
        // 依赖注入使用的对象工厂
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
```

首先通过type指定该ExtensionLoader加载的扩展点类型，再是指定一个ExtensionFactory实例（用于依赖注入）。

我们可以看到这一行的操作，如果是加载ExtensionFactory本身这个扩展点，则objectFactory属性为null，否则通过ExtensionLoader机制获取自适应的扩展类。

其中getAdaptiveExtension()方法用于加载自适应扩展点，这个后面再提，其主要的功能就是加载一个默认/自适应的扩展类。

#### 2. 通过ExtensionLoader获取扩展点

成功获取到对应扩展点的ExtensionLoader后，调用其 **getExtension(String)** 方法获取到具体的扩展点。

回忆一下Dubbo SPI的机制，我们需要在 **META-INF/dubbo/接口全限定名** 文件中通过 **配置名=扩展实现类全限定名** 的方式配置我们的扩展点实现，因此获取具体扩展点实现的方法入参为 *String配置名*。

```java
    public T getExtension(String name) {
        if (StringUtils.isEmpty(name)) {
            throw new IllegalArgumentException("Extension name == null");
        }
        if ("true".equals(name)) {
            return getDefaultExtension();
        }
        // 双检锁从缓存中获取扩展点实现实例对象
        // 1. 从缓存中查询（ConcurrentMap<String, Holder<Object>> cachedInstances）
        final Holder<Object> holder = getOrCreateHolder(name);
        Object instance = holder.get();
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    // 2. 创建扩展点实现实例对象并放到缓存里
                    instance = createExtension(name);
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }
```

我们先看一下 **createExtension(name)**，里头涉及到一些比较关键的点。首先获取到配置名对应的扩展点实现类，尝试从缓存中获取（Dubbo中用了大量的全局缓存来优化创建查询等操作）；接下来实现 **扩展点自动装配** 的特性，类似于Spring IoC，但Dubbo只允许通过setter方法进行注入；再接下来实现 **扩展点自动包装** 的特性，代理模式类似于Spring AOP，不过是通过构造器传入被代理对象的方式实现；最后是一些dubbo组件的初始化操作。代码及注释如下：

```java
private T createExtension(String name) {
        // 1. 获取name配置名对应的扩展点实现类
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            // 2. 从已创建的实例缓存中查询
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            // 3. 依赖注入，如果扩展点实现依赖其他扩展点，则需要通过setter方法注入
            injectExtension(instance);
            // 4. wrapper类代理
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (CollectionUtils.isNotEmpty(wrapperClasses)) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            // 初始化一些dubbo组件
            initExtension(instance);
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                    type + ") couldn't be instantiated: " + t.getMessage(), t);
        }
    }
```

在创建扩展类实例的过程中，有一步很重要，就是第一步获取对应的扩展点实现类 **getExtensionClassed()**，这个方法在整个dubbo流程中到处都被使用，因为这一步涉及到加载扩展点的配置文件。

同样是全局缓存，通过双检锁进行 **loadExtensionClasses()** 操作。

```java
    private Map<String, Class<?>> getExtensionClasses() {
        // 从配置文件中加载的所有扩展点类缓存中查询
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    // 加载配置文件
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
```

loadExtensionClasses()方法分两步。第一步加载默认扩展点实现，通过 **@SPI** 注解的参数指定；第二步从配置文件中加载，以配置名为key，扩展点实现为value的Map返回。

```java
    private Map<String, Class<?>> loadExtensionClasses() {
        // 1. 加载并缓存默认扩展点实现
        cacheDefaultExtensionName();

        // 2. 从配置文件中加载
        Map<String, Class<?>> extensionClasses = new HashMap<>();

        for (LoadingStrategy strategy : strategies) {
            loadDirectory(extensionClasses, strategy.directory(), type.getName(), strategy.preferExtensionClassLoader(), strategy.excludedPackages());
            loadDirectory(extensionClasses, strategy.directory(), type.getName().replace("org.apache", "com.alibaba"), strategy.preferExtensionClassLoader(), strategy.excludedPackages());
        }

        return extensionClasses;
    }
```

先看加载默认扩展点实现，从扩展点接口上获取SPI注解，该注解有一个value属性标识该扩展点的默认实现方式，当然也可以为空，具体 的可以看@SPI接口的注释。

```java
    private void cacheDefaultExtensionName() {
        // 获取扩展点接口上的SPI注解，SPI注解只能用在接口上
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        if (defaultAnnotation == null) {
            return;
        }

        // 获取SPI接口上的默认扩展点实现方式
        String value = defaultAnnotation.value();
        if ((value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            if (names.length > 1) {
                throw new IllegalStateException("More than 1 default extension name on extension " + type.getName()
                        + ": " + Arrays.toString(names));
            }
            if (names.length == 1) {
                cachedDefaultName = names[0];
            }
        }
    }
```

第二步是加载用户配置文件。其中strategies是指定的三个配置文件路径：

1. META-INF/dubbo/
2. META-INF/dubbo/internal/
3. META-INF/services/

从 **loadDirectory** 方法往下翻，里面比较重要的就是 **loadClass** 方法，

```java
    private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
        if (!type.isAssignableFrom(clazz)) {
            throw new IllegalStateException("Error occurred when loading extension class (interface: " +
                    type + ", class line: " + clazz.getName() + "), class "
                    + clazz.getName() + " is not subtype of interface.");
        }
        // 缓存标注自适应实现类
        if (clazz.isAnnotationPresent(Adaptive.class)) {
            cacheAdaptiveClass(clazz);
        }
        // 缓存Wrapper代理类
        else if (isWrapperClass(clazz)) {
            cacheWrapperClass(clazz);
        }
        // 
        else {
            clazz.getConstructor();
            if (StringUtils.isEmpty(name)) {
                name = findAnnotationName(clazz);
                if (name.length() == 0) {
                    throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
                }
            }

            String[] names = NAME_SEPARATOR.split(name);
            if (ArrayUtils.isNotEmpty(names)) {
                cacheActivateClass(clazz, names[0]);
                for (String n : names) {
                    cacheName(clazz, n);
                    saveInExtensionClass(extensionClasses, clazz, n);
                }
            }
        }
    }
```



