### Dubbo 优雅停机

#### 一、Dubbo 优雅停机待解决的问题

为了实现优雅停机，Dubbo 需要解决一些问题：

1. 新的请求不能再发往正在停机的 Dubbo 服务提供者。
2. 若关闭服务提供者，已经接收到服务请求，需要处理完毕才能下线服务。
3. 若关闭服务消费者，已经发出的服务请求，需要等待响应返回。

解决以上三个问题，才能使停机对业务影响降低到最低，做到优雅停机。

## 二、2.5.X

Dubbo 优雅停机在 2.5.X 版本实现比较完整，这个版本的实现相对简单，比较容易理解。所以我们先以 Dubbo 2.5.X 版本源码为基础，先来看一下 Dubbo 如何实现优雅停机。

优雅停机入口类位于 `AbstractConfig` 静态代码中，源码如下：

```java
static {
    Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
        public void run() {
            if (logger.isInfoEnabled()) {
                logger.info("Run shutdown hook now.");
            }
            ProtocolConfig.destroyAll();
        }
    }, "DubboShutdownHook"));
}
```

这里将会注册一个 `ShutdownHook` ，一旦应用停机将会触发调用  `ProtocolConfig.destroyAll()`。`ProtocolConfig.destroyAll() `源码如下：

```java
public static void destroyAll() {
    //注销注册中心
    AbstractRegistryFactory.destroyAll();
    ExtensionLoader<Protocol> loader = ExtensionLoader.getExtensionLoader(Protocol.class);
    for (String protocolName : loader.getLoadedExtensions()) {
        try {
            //注销Protocol
            Protocol protocol = loader.getLoadedExtension(protocolName);
            if (protocol != null) {
                protocol.destroy();
            }
        } catch (Throwable t) {
            logger.warn(t.getMessage(), t);
        }
    }
}
```

从上面可以看到，Dubbo 优雅停机主要分为两步：

1. 注销注册中心。
2. 注销所有 Protocol。

注销注册中心源码如下：

```java
public static void destroyAll() {
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("Close all registries " + getRegistries());
    }
    // 锁定注册中心关闭过程
    LOCK.lock();
    try {
        for (Registry registry : getRegistries()) {
            try {
                //删除注册信息、取消订阅
                registry.destroy();
            } catch (Throwable e) {
                LOGGER.error(e.getMessage(), e);
            }
        }
        REGISTRIES.clear();
    } finally {
        // 释放锁
        LOCK.unlock();
    }
}
```

以 ZK 为例，Dubbo 将会删除其对应服务节点，然后取消订阅。由于 ZK 节点信息变更，ZK 服务端将会通知 dubbo 消费者下线该服务节点，最后再关闭服务与 ZK 连接。

通过注册中心，Dubbo 可以及时通知消费者下线服务，新的请求也不再发往下线的节点，也就解决上面提到的第一个问题：**新的请求不能再发往正在停机的 Dubbo 服务提供者**。

但是这里还是存在一些弊端，由于网络的隔离，ZK 服务端与 Dubbo 连接可能存在一定延迟，ZK 通知可能不能在第一时间通知消费端。考虑到这种情况，在注销注册中心之后，加入等待进制，代码如下：

```java
try {
    Thread.sleep(ConfigUtils.getServerShutdownTimeout());
} catch (InterruptedException e) {
    logger.warn("Interrupted unexpectedly when waiting for registry notification during shutdown process!");
}
```

默认等待时间为 **10000ms**，可以通过设置 `dubbo.service.shutdown.wait` 覆盖默认参数。10s 只是一个经验值，可以根据实际情设置。不过这个等待时间设置比较讲究，不能设置成太短，太短将会导致消费端还未收到 ZK 通知，提供者就停机了。也不能设置太长，太长又会导致关停应用时间边长，影响发布体验。

#### 注销Protocol:

```java
for (String protocolName : loader.getLoadedExtensions()) {
    try {
        Protocol protocol = loader.getLoadedExtension(protocolName);
        if (protocol != null) {
            protocol.destroy();
        }
    } catch (Throwable t) {
        logger.warn(t.getMessage(), t);
    }
}
```

`loader#getLoadedExtensions` 将会返回两种 `Protocol` 子类，分别为 `DubboProtocol` 与 `InjvmProtocol`。

`DubboProtocol` 用与服务端请求交互，而 `InjvmProtocol` 用于内部请求交互。如果应用调用自己提供 Dubbo 服务，不会再执行网络调用，直接执行内部方法。

这里我们主要来分析一下 `DubboProtocol` 内部逻辑。

```java
public void destroy() {
    for (String key : new ArrayList<String>(serverMap.keySet())) {
        ExchangeServer server = serverMap.remove(key);
        if (server != null) {
            try {
                if (logger.isInfoEnabled()) {
                    logger.info("Close dubbo server: " + server.getLocalAddress());
                }
                //关闭server
                server.close(getServerShutdownTimeout());
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
        }
    }
    ////关闭client
    for (String key : new ArrayList<String>(referenceClientMap.keySet())) {
        ExchangeClient client = referenceClientMap.remove(key);
        if (client != null) {
            try {
                if (logger.isInfoEnabled()) {
                    logger.info("Close dubbo connect: " + client.getLocalAddress() + "-->" + client.getRemoteAddress());
                }
                
                client.close();
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
        }
    }
    
    for (String key : new ArrayList<String>(ghostClientMap.keySet())) {
        ExchangeClient client = ghostClientMap.remove(key);
        if (client != null) {
            try {
                if (logger.isInfoEnabled()) {
                    logger.info("Close dubbo connect: " + client.getLocalAddress() + "-->" + client.getRemoteAddress());
                }
                client.close();
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
        }
    }
    stubServiceMethodsMap.clear();
    super.destroy();
}
```

Dubbo 默认使用 Netty 作为其底层的通讯框架，分为 `Server` 与 `Client`。`Server` 用于接收其他消费者 `Client` 发出的请求。

上面源码中首先关闭 `Server` ，停止接收新的请求，然后再关闭 `Client`。这样做就降低服务被消费者调用的可能性。

### 关闭 Server

```java
public void close(final int timeout) {
    if (timeout > 0) {
        final long max = (long) timeout;
        final long start = System.currentTimeMillis();
        if (getUrl().getParameter(Constants.CHANNEL_SEND_READONLYEVENT_KEY, false)){
            //发送read_only事件
            sendChannelReadOnlyEvent();
        }
        while (HeaderExchangeServer.this.isRunning() 
                && System.currentTimeMillis() - start < max) {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                logger.warn(e.getMessage(), e);
            }
        }
    }
    //关掉定时心跳
    doClose();
    server.close(timeout);
}
```

这里将会向服务消费者发送 `READ_ONLY` 事件。消费者接受之后，主动排除这个节点，将请求发往其他正常节点。这样又进一步降低了注册中心通知延迟带来的影响。

接下来将会关闭心跳检测，关闭底层通讯框架 NettyServer。

```java
//AbstractServer
public void close(int timeout) {
    //关闭线程池 这个过程将会尽可能将线程池中的任务执行完毕，再关闭线程池
    ExecutorUtil.gracefulShutdown(executor ,timeout);
    //关闭netty服务
    close();
}
```

```java
//关闭netty服务 
try {
    if (channel != null) {
        // unbind.
        channel.close();
    }
} catch (Throwable e) {
    logger.warn(e.getMessage(), e);
}
```

关闭 Server，优雅等待线程池关闭，解决了上面提到的第二个问题：若关闭服务提供者，已经接收到服务请求，需要处理完毕才能下线服务。

#### 关闭 Client

Client 关闭方式大致同 Server，这里主要介绍一下处理已经发出请求逻辑，代码位于`HeaderExchangeChannel#close`

```java
//HeaderExchangeChannel
public void close(int timeout) {
    if (closed) {
        return;
    }
    closed = true;
    if (timeout > 0) {
        long start = System.currentTimeMillis();
        //等待发送的请求响应信息
        while (DefaultFuture.hasFuture(HeaderExchangeChannel.this) 
                && System.currentTimeMillis() - start < timeout) {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                logger.warn(e.getMessage(), e);
            }
        }
    }
    close();
}
```

关闭 Client 的时候，如果还存在未收到响应的信息请求，将会等待一定时间，直到确认所有请求都收到响应，或者等待时间超过超时时间。

通过这一点我们就解决了第三个问题：若关闭服务消费者，已经发出的服务请求，需要等待响应返回。