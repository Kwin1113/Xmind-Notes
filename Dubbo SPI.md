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

大概看一眼构造器里面干了什么初始化操作。

```java
    private ExtensionLoader(Class<?> type) {
        this.type = type;
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
```

