---
title: Java 中使用 Zookeeper 客户端 Curator 详解
date: 2020-01-08 19:46:55
tags:
- Zookeeper
- 分布式锁
categories:
- 后端
- Java
- Zookeeper
---

# 简介

因为最近项目需要使用Zookeeper这个中间件，提前了解一下它的客户端Curator的使用。

Curator是Netflix公司开源的一套zookeeper客户端框架，解决了很多Zookeeper客户端非常底层的细节开发工作，包括连接重连、反复注册Watcher和NodeExistsException异常等等。Patrixck Hunt（Zookeeper）以一句“Guava is to Java that Curator to Zookeeper”给Curator予高度评价。

Curator无疑是Zookeeper客户端中的瑞士军刀，它译作”馆长”或者’’管理者’’，不知道是不是开发小组有意而为之，笔者猜测有可能这样命名的原因是说明Curator就是Zookeeper的馆长(脑洞有点大：Curator就是动物园的园长)。
Curator包含了几个包：

- curator-framework：对zookeeper的底层api的一些封装。
- curator-client：提供一些客户端的操作，例如重试策略等。
- curator-recipes：封装了一些高级特性，如：Cache事件监听、选举、分布式锁、分布式计数器、分布式Barrier等。

<!-- more -->

Maven依赖（注意zookeeper版本  这里对应的是3.4.6）

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>2.10.0</version>
</dependency>
```

# Curator的基本Api

## 创建会话

**方式一**

```java
@Test
public void test() {
    /*
    创建重试策略对象，参数含义如下：
    - baseSleepTimeMs: 基本睡眠时间。
    - maxRetries：最大重试次数。
    */
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    /*
    创建客户端对象，参数含义如下：
    - connectString：服务器列表，格式为 `host1:port,host2:port,...`。
    - sessionTimeoutMs：会话超时时间， 默认 60000 ms。
    - connectionTimeoutMs：连接超时时间，默认 60000 ms。
    - retryPolicy：重试策略
    */
    CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 5000, 5000, retryPolicy);
    // 连接 zookeeper 服务器
    client.start();
}
```

**方式二**

```java
@Test
public void test2() {
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    CuratorFramework client = CuratorFrameworkFactory.builder().connectString("127.0.0.1:2181")
        .sessionTimeoutMs(5000)
        .connectionTimeoutMs(5000)
        .retryPolicy(retryPolicy)
        .build();
    client.start();
}
```

**创建包含隔离命名空间的会话**

为了实现不同的Zookeeper业务之间的隔离，需要为每个业务分配一个独立的命名空间（NameSpace），即指定一个Zookeeper的根路径（官方术语：为Zookeeper添加“Chroot”特性）。例如（下面的例子）当客户端指定了独立命名空间为“/base”，那么该客户端对Zookeeper上的数据节点的操作都是基于该目录进行的。通过设置Chroot可以将客户端应用与Zookeeper服务端的一课子树相对应，在多个应用共用一个Zookeeper集群的场景下，这对于实现不同应用之间的相互隔离十分有意义。

```java
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
		CuratorFramework client =
		CuratorFrameworkFactory.builder()
				.connectString(connectionInfo)
				.sessionTimeoutMs(5000)
				.connectionTimeoutMs(5000)
				.retryPolicy(retryPolicy)
				.namespace("base")
				.build();
```

## 数据节点基本操作（增删改查）

**Zookeeper的节点创建模式：**

- PERSISTENT：持久化
- PERSISTENT_SEQUENTIAL：持久化并且带序列号
- EPHEMERAL：临时
- EPHEMERAL_SEQUENTIAL：临时并且带序列号

增删改查操作如下

```java
@Before
public void before() throws Exception {
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    CuratorFramework client = CuratorFrameworkFactory.builder().connectString("127.0.0.1:2181")
        .sessionTimeoutMs(5000)
        .connectionTimeoutMs(5000)
        .retryPolicy(retryPolicy)
        .build();
    client.start();
    // 删除一个节点，并且递归删除其所有的子节点
    client.delete().deletingChildrenIfNeeded().forPath("/study");
    LOGGER.info("清除上一次测试的数据成功！");
}

/**
 * 基本操作
 * @throws Exception
 */
@Test
public void test3() throws Exception {
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    CuratorFramework client = CuratorFrameworkFactory.builder().connectString("127.0.0.1:2181")
        .sessionTimeoutMs(5000)
        .connectionTimeoutMs(5000)
        .retryPolicy(retryPolicy)
        .namespace("study")
        .build();
    client.start();
    /**
     * 创建节点
     */
    // 创建一个节点，初始内容为空
    client.create().forPath("/name");
    // 创建一个节点，附带初始化内容
    client.create().forPath("/name2", "创建一个节点，附带初始化内容".getBytes(Charset.forName("utf-8")));
    // 创建一个节点，指定创建模式（临时节点），内容为空
    client.create().withMode(CreateMode.EPHEMERAL).forPath("/name3");
    // 创建一个节点，指定创建模式（临时节点），附带初始化内容
    client.create().withMode(CreateMode.EPHEMERAL).forPath("/name4", "创建一个节点，指定创建模式（临时节点），附带初始化内容".getBytes(Charset.forName("utf-8")));
    // 创建一个节点，指定创建模式（临时节点），附带初始化内容，并且自动递归创建父节点
    client.create().creatingParentContainersIfNeeded().withMode(CreateMode.EPHEMERAL).forPath("/parent/name5", "创建一个节点，指定创建模式（临时节点），附带初始化内容，并且自动递归创建父节点".getBytes(Charset.forName("utf-8")));

    /**
     * 更新数据节点数据
     */
    // 更新一个节点的数据内容
    client.setData().forPath("/name2", "更新一个节点的数据内容".getBytes(Charset.forName("utf-8")));
    // 更新一个节点的数据内容，强制指定版本进行更新
    client.setData().withVersion(0).forPath("/name", "更新一个节点的数据内容，强制指定版本进行更新".getBytes(Charset.forName("utf-8")));

    /**
     * 读取节点
     */
    // 读取一个节点的数据内容
    String s = new String(client.getData().forPath("/name2"), Charset.forName("utf-8"));
    LOGGER.info("读取一个节点的数据内容, s: [{}]", s);
    // 读取一个节点的数据内容，同时获取到该节点的stat
    Stat stat = new Stat();
    s = new String(client.getData().storingStatIn(stat).forPath("/name"), Charset.forName("utf-8"));
    LOGGER.info("读取一个节点的数据内容，同时获取到该节点的stat, s: [{}], stat: [{}]", s, stat);

    /**
     * 删除节点
     */
    // 删除一个节点
    client.delete().forPath("/name");
    // 删除一个节点，并且递归删除其所有的子节点
    client.delete().deletingChildrenIfNeeded().forPath("/parent");

    /**
     * 检查节点是否存在，不存在时对象为 null
     */
    stat = client.checkExists().forPath("/name5");
    LOGGER.info("检查节点是否存在, stat: [{}]", stat);
}
```

## 事务

CuratorFramework的实例包含inTransaction( )接口方法，调用此方法开启一个ZooKeeper事务. 可以复合create, setData, check, and/or delete 等操作然后调用commit()作为一个原子操作提交。一个例子如下：

```java
@Test
public void test4() throws Exception {
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    CuratorFramework client = CuratorFrameworkFactory.builder()
        .connectString("127.0.0.1:2181")
        .sessionTimeoutMs(5000)
        .connectionTimeoutMs(5000)
        .retryPolicy(retryPolicy)
        .namespace("study")
        .build();
    client.start();

    // 事务操作，保证原子性
    client.inTransaction()
        .create().withMode(CreateMode.EPHEMERAL).forPath("/name", "aaaaa".getBytes())
        .and().setData().forPath("/name", "bbb".getBytes())
        .and().commit();
}
```

## 异步接口

上面提到的创建、删除、更新、读取等方法都是同步的，Curator提供异步接口，引入了BackgroundCallback接口用于处理异步接口调用之后服务端返回的结果信息。BackgroundCallback接口中一个重要的回调值为CuratorEvent，里面包含事件类型、响应吗和节点的详细信息。

**CuratorEventType**

| 事件类型 | 对应CuratorFramework实例的方法 |
| :------: | :----------------------------: |
|  CREATE  |           #create()            |
|  DELETE  |           #delete()            |
|  EXISTS  |         #checkExists()         |
| GET_DATA |           #getData()           |
| SET_DATA |           #setData()           |
| CHILDREN |         #getChildren()         |
|   SYNC   |      #sync(String,Object)      |
| GET_ACL  |           #getACL()            |
| SET_ACL  |           #setACL()            |
| WATCHED  |       #Watcher(Watcher)        |
| CLOSING  |            #close()            |

**响应码(#getResultCode())**

| 响应码 |                   意义                   |
| :----: | :--------------------------------------: |
|   0    |              OK，即调用成功              |
|   -4   | ConnectionLoss，即客户端与服务端断开连接 |
|  -110  |        NodeExists，即节点已经存在        |
|  -112  |        SessionExpired，即会话过期        |

一个异步创建节点的例子如下：

```java
private static final Logger LOGGER = LoggerFactory.getLogger(Test04App.class);

@Before
public void before() throws Exception {
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    CuratorFramework client = CuratorFrameworkFactory.builder().connectString("127.0.0.1:2181")
        .sessionTimeoutMs(5000)
        .connectionTimeoutMs(5000)
        .retryPolicy(retryPolicy)
        .build();
    client.start();
    // 删除一个节点，并且递归删除其所有的子节点
    Stat stat = client.checkExists().forPath("/study");
    if (stat != null) {
        client.delete().deletingChildrenIfNeeded().forPath("/study");
    }
    LOGGER.info("清除上一次测试的数据成功！");
}

/**
 * 异步操作
 * @throws Exception
 */
@Test
public void test5() throws Exception {
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    CuratorFramework client = CuratorFrameworkFactory.builder()
        .connectString("127.0.0.1:2181")
        .sessionTimeoutMs(5000)
        .connectionTimeoutMs(5000)
        .retryPolicy(retryPolicy)
        .namespace("study")
        .build();
    client.start();

    Executor executor = Executors.newFixedThreadPool(2);
    client.create().creatingParentContainersIfNeeded().withMode(CreateMode.EPHEMERAL).inBackground(new BackgroundCallback() {
        @Override
        public void processResult(CuratorFramework client, CuratorEvent event) throws Exception {
            LOGGER.info("开始调用回调方法，WatchedEvent: [{}], ResultCode: [{}]", event.getType(), event.getResultCode());
        }
    }, executor).forPath("/name");

    // 不让程序结束，否则看不到回调方法的调用
    for (;;);
}
```

<b>注意：</b>如果#inBackground()方法不指定executor，那么会默认使用Curator的EventThread去进行异步处理。

# Curator 高级特性

<b>提醒：</b>强烈推荐使用ConnectionStateListener监控连接的状态，当连接状态为LOST，curator-recipes下的所有Api将会失效或者过期，尽管后面所有的例子都没有使用到ConnectionStateListener。

## 缓存

Zookeeper原生支持通过注册Watcher来进行事件监听，但是开发者需要反复注册(Watcher只能单次注册单次使用)。Cache是Curator中对事件监听的包装，可以看作是对事件监听的本地缓存视图，能够自动为开发者处理反复注册监听。Curator提供了三种Watcher(Cache)来监听结点的变化。

### Path Cache

Path Cache用来监控一个ZNode的子节点. 当一个子节点增加， 更新，删除时， Path Cache会改变它的状态， 会包含最新的子节点， 子节点的数据和状态，而状态的更变将通过PathChildrenCacheListener通知。

实际使用时会涉及到四个类：

- PathChildrenCache
- PathChildrenCacheEvent
- PathChildrenCacheListener
- ChildData

通过下面的构造函数创建Path Cache:

```java
public PathChildrenCache(CuratorFramework client, String path, boolean cacheData)
```

想使用cache，必须调用它的start方法，使用完后调用close方法。 可以设置StartMode来实现启动的模式。

StartMode有下面几种：

1. NORMAL：正常初始化。
2. BUILD_INITIAL_CACHE：在调用start()之前会调用rebuild()。
3. POST_INITIALIZED_EVENT： 当Cache初始化数据后发送一个PathChildrenCacheEvent.Type#INITIALIZED事件。

`public void addListener(PathChildrenCacheListener listener)`可以增加listener监听缓存的变化。

`getCurrentData()`方法返回一个List`<ChildData>`对象，可以遍历所有的子节点。

设置/更新、移除其实是使用client (CuratorFramework)来操作, 不通过PathChildrenCache操作，案例如下代码所示：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.cache.*;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.TimeUnit;

/**
 * @author ZGY
 * @date 2019/12/25 10:31
 * @description Test05App, Curator 高级特性案例
 */
public class Test05App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test05App.class);

    @Test
    public void test() throws Exception {
        // 创建客户端 CuratorFramework 对象
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .retryPolicy(new ExponentialBackoffRetry(1000, 3))
                .connectString("127.0.0.1:2181")
                .sessionTimeoutMs(5000)
                .connectionTimeoutMs(5000)
                .build();
        // 连接 zookeeper服务器
        client.start();
        // 创建一个 PathChildrenCache 对象来监听对应路径下的的子节点
        PathChildrenCache pathChildrenCache = new PathChildrenCache (client, "/example/cache", true);
        // 开始监听子节点变化
        pathChildrenCache.start();
        // 当子节点数据变化时需要处理的逻辑
        pathChildrenCache.getListenable().addListener((clientFramework, event) -> {
            LOGGER.info("事件类型为：{}", event.getType());
            if (null != event.getData()) {
                LOGGER.info("节点路径为：{}，节点数据为：{}", event.getData().getPath(), new String(event.getData().getData()));
            }
        });

        // 创建节点
        client.create().creatingParentsIfNeeded().forPath("/example/cache/test01", "01".getBytes());
        TimeUnit.MILLISECONDS.sleep(10);

        // 创建节点
        client.create().creatingParentsIfNeeded().forPath("/example/cache/test02", "02".getBytes());
        TimeUnit.MILLISECONDS.sleep(10);

        // 修改数据
        client.setData().forPath("/example/cache/test01", "01_V2".getBytes());
        TimeUnit.MILLISECONDS.sleep(10);

        // 遍历缓存中的数据
        for (ChildData childData : pathChildrenCache.getCurrentData()) {
            LOGGER.info("获取childData对象数据, Path: [{}], Data: [{}]", childData.getPath(), new String(childData.getData()));
        }

        // 删除数据
        client.delete().forPath("/example/cache/test01");
        TimeUnit.MILLISECONDS.sleep(10);

        // 删除数据
        client.delete().forPath("/example/cache/test02");
        TimeUnit.MILLISECONDS.sleep(10);

        // 关闭监听
        pathChildrenCache.close();

        // 删除测试用的数据，如果存在子节点，一并删除
        client.delete().deletingChildrenIfNeeded().forPath("/example");

        // 断开与 zookeeper 的连接
        client.close();
        LOGGER.info("程序执行完毕");
    }
}
```

<b>注意：</b>如果new PathChildrenCache(client, PATH, true)中的参数cacheData值设置为false，则示例中的event.getData().getData()、data.getData()将返回null，cache将不会缓存节点数据。

<b>注意：</b>示例中的TimeUnit.MILLISECONDS.sleep(10)可以注释掉，但是注释后事件监听的触发次数会不全，这可能与PathCache的实现原理有关，不能太过频繁的触发事件！

### Node Cache

Node Cache与Path Cache类似，Node Cache只是监听某一个特定的节点。它涉及到下面的三个类：

- NodeCache - Node Cache实现类
- NodeCacheListener - 节点监听器
- ChildData - 节点数据

<b>注意：</b>使用cache，依然要调用它的start()方法，使用完后调用close()方法。

getCurrentData()将得到节点当前的状态，通过它的状态可以得到当前的值。

示例代码如下：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.cache.ChildData;
import org.apache.curator.framework.recipes.cache.NodeCache;
import org.apache.curator.framework.recipes.cache.NodeCacheListener;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.TimeUnit;

/**
 * @author ZGY
 * @date 2019/12/25 11:19
 * @description Test06App, Curator 高级特性 Node Cache
 */
public class Test06App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test06App.class);

    @Test
    public void test() throws Exception {
        // 创建客户端 CuratorFramework 对象
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectionTimeoutMs(5000)
                .connectString("127.0.0.1:2181")
                .sessionTimeoutMs(5000)
                .retryPolicy(new ExponentialBackoffRetry(1000, 3))
                .build();

        // 连接 zookeeper服务器
        client.start();

        // 创建一个 NodeCache 对象来监听指定节点
        NodeCache nodeCache = new NodeCache(client, "/example/cache");
        // 当节点数据变化时需要处理的逻辑
        nodeCache.getListenable().addListener(() -> {
            ChildData currentData = nodeCache.getCurrentData();
            if (null != currentData) {
                LOGGER.info("节点数据：Path[{}], Data: [{}]",currentData.getPath() , new String(currentData.getData()));
            } else {
                LOGGER.info("节点被删除！");
            }
        });
        // 开始监听子节点变化
        nodeCache.start();

        // 创建节点
        client.create().creatingParentsIfNeeded().forPath("/example/cache", "test01".getBytes());
        TimeUnit.MILLISECONDS.sleep(100);

        // 修改数据
        client.setData().forPath("/example/cache", "test01_V1".getBytes());
        TimeUnit.MILLISECONDS.sleep(100);

        // 获取节点数据
        String s = new String(client.getData().forPath("/example/cache"));
        LOGGER.info("数据s：[{}]", s);

        // 删除节点
        client.delete().forPath("/example/cache");
        TimeUnit.MILLISECONDS.sleep(100);

        // 删除测试用的数据，如果存在子节点，一并删除
        client.delete().deletingChildrenIfNeeded().forPath("/example");

        // 关闭监听
        nodeCache.close();

        // 断开与 zookeeper 的连接
        client.close();

        LOGGER.info("程序执行完毕！");

        // 为了查看打印日志，不加这段代码看不到节点监听处理逻辑
        for (;;);
    }
}
```

<b>注意：</b>示例中的TimeUnit.MILLISECONDS.sleep(100)可以注释，但是注释后事件监听的触发次数会不全，这可能与NodeCache的实现原理有关，不能太过频繁的触发事件！

<b>注意：</b>NodeCache只能监听一个节点的状态变化。

### Tree Cache

Tree Cache可以监控整个树上的所有节点，类似于PathCache和NodeCache的组合，主要涉及到下面四个类：

- TreeCache - Tree Cache实现类
- TreeCacheListener - 监听器类
- TreeCacheEvent - 触发的事件类
- ChildData - 节点数据

示例代码如下：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.cache.TreeCache;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.concurrent.TimeUnit;

/**
 * @author ZGY
 * @date 2019/12/25 11:45
 * @description Test07App, Curator 高级特性 Tree Cache
 */
public class Test07App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test07App.class);

    @Test
    public void test() throws Exception {
        // 创建客户端 CuratorFramework 对象
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectionTimeoutMs(5000)
                .connectString("127.0.0.1:2181")
                .sessionTimeoutMs(5000)
                .retryPolicy(new ExponentialBackoffRetry(1000, 3))
                .build();

        // 连接 zookeeper服务器
        client.start();

        // 创建一个 NodeCache 对象来监听指定节点下的所有节点
        TreeCache treeCache = new TreeCache(client, "/example/cache");
        // 当指定节点下的某个节点数据变化时需要处理的逻辑
        treeCache.getListenable().addListener((curatorFramework, event) -> {
            LOGGER.info("事件类型：{}， 路径：{}，数据：{}",
                    event.getType(),
                    event.getData() == null? null:event.getData().getPath(),
                    event.getData() == null? null:new String(event.getData().getData()));
        });

        // 开始监听指定节点下的所有节点变化
        treeCache.start();

        // 创建节点
        client.create().creatingParentsIfNeeded().forPath("/example/cache", "test01".getBytes());
        TimeUnit.MILLISECONDS.sleep(100);

        // 修改数据
        client.setData().forPath("/example/cache", "test01_V2".getBytes());
        TimeUnit.MILLISECONDS.sleep(100);

        // 修改数据
        client.setData().forPath("/example/cache", "test01_V3".getBytes());
        TimeUnit.MILLISECONDS.sleep(100);

        // 删除节点
        client.delete().forPath("/example/cache");
        TimeUnit.MILLISECONDS.sleep(100);

        // 删除测试用的数据，如果存在子节点，一并删除
        client.delete().deletingChildrenIfNeeded().forPath("/example");

        // 关闭监听
        treeCache.close();

        // 断开与 zookeeper 的连接
        client.close();

        LOGGER.info("程序执行完毕！");

        // 为了查看打印日志，不加这段代码看不到节点监听处理逻辑
        for (;;);
    }
}
```

<b>注意：</b>在此示例中没有使用TimeUnit.MILLISECONDS.sleep(100)，但是事件触发次数也是正常的。

<b>注意：</b>TreeCache在初始化(调用start()方法)的时候会回调TreeCacheListener实例一个事TreeCacheEvent，而回调的TreeCacheEvent对象的Type为INITIALIZED，ChildData为null，此时event.getData().getPath()很有可能导致空指针异常，这里应该主动处理并避免这种情况。

## Leader选举

使用场景如下，当我们的某些功能要提供高可用时，比如，服务器突然崩溃了，导致功能不能访问，这时就可以把该功能部署到多台机器上，但是并不是让这些机器上的功能同时对外提供服务，而是选一台机器上的功能对外提供服务，其他机器上的功能用作备用，当正在对外提供服务的机器出现故障时，我们就执行 Leader 选举，再选择一台机器来对外提供服务，这样就保证了服务的高可用。

Curator 有两种 leader 选举的方式,分别是**LeaderSelector**和**LeaderLatch**。

### LeaderLatch

LeaderLatch有两个构造函数：

```java
public LeaderLatch(CuratorFramework client, String latchPath)
public LeaderLatch(CuratorFramework client, String latchPath,  String id)
```

LeaderLatch的启动：

```java
leaderLatch.start( );
```

一旦启动，LeaderLatch 会和其它使用相同`latchPath`的其它 LeaderLatch 交涉，然后其中一个最终会被选举为leader，可以通过`leaderLatch.hasLeadership()`方法查看LeaderLatch实例是否leader,`true`说明当前实例是leader。

<b>异常处理：</b> LeaderLatch实例可以增加ConnectionStateListener来监听网络连接问题。 当 SUSPENDED 或 LOST 时, leader不再认为自己还是leader。当LOST后连接重连后RECONNECTED,LeaderLatch会删除先前的ZNode然后重新创建一个。LeaderLatch用户必须考虑导致leadership丢失的连接问题。 强烈推荐你使用ConnectionStateListener。

示例代码如下：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.leader.LeaderLatch;
import org.apache.curator.framework.recipes.leader.LeaderLatchListener;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.curator.utils.CloseableUtils;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

/**
 * @author ZGY
 * @date 2019/12/25 15:11
 * @description Test08App, Curator 高级特性 leader 选举
 */
public class Test08App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test08App.class);

    /**
     * 使用 LeaderLatch
     */
    @Test
    public void test() throws Exception {
        List<CuratorFramework> clientList = new ArrayList<>();
        List<LeaderLatch> leaderLatchList = new ArrayList<>();
        ExponentialBackoffRetry retry = new ExponentialBackoffRetry(1000, 3);

        for (int i = 0; i < 5; i++) {
            CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 5000, 5000, retry);
            clientList.add(client);

            final LeaderLatch latch = new LeaderLatch(client, "/francis/leader", "#client" + i);
            latch.addListener(new LeaderLatchListener() {
                @Override
                public void isLeader() {
                    LOGGER.info("I am Leader, id: [{}]", latch.getId());
                }

                @Override
                public void notLeader() {
                    LOGGER.info("I am not Leader, id: [{}]", latch.getId());
                }
            });
            leaderLatchList.add(latch);

            client.start();
            latch.start();
        }

        LOGGER.info("程序停止10秒开始");
        TimeUnit.SECONDS.sleep(10);
        LOGGER.info("程序停止10秒结束");

        LeaderLatch currentLatch = null;
        for (LeaderLatch latch : leaderLatchList) {
            if (latch.hasLeadership()) {
                currentLatch = latch;
                break;
            }
        }
        LOGGER.info("current leader is {}", currentLatch.getId());
        currentLatch.close();
        LOGGER.info("release the leader {}", currentLatch.getId());

        TimeUnit.SECONDS.sleep(5);

        for (LeaderLatch latch : leaderLatchList) {
            if (latch.hasLeadership()) {
                currentLatch = latch;
                break;
            }
        }
        LOGGER.info("current leader is {}", currentLatch.getId());
        currentLatch.close();
        LOGGER.info("release the leader {}", currentLatch.getId());

        TimeUnit.SECONDS.sleep(10);

        for (LeaderLatch latch : leaderLatchList) {
            if (null != latch.getState() && !latch.getState().equals(LeaderLatch.State.CLOSED)) {
                CloseableUtils.closeQuietly(latch);
            }
        }
        for (CuratorFramework client : clientList) {
            CloseableUtils.closeQuietly(client);
        }
    }
}
```

首先我们创建了5个LeaderLatch，启动后它们中的一个会被选举为leader。 因为选举会花费一些时间，start后并不能马上就得到leader。
通过hasLeadership查看自己是否是leader， 如果是的话返回true。
可以通过.getLeader().getId()可以得到当前的leader的ID。
只能通过close释放当前的领导权。

### LeaderSelector

LeaderSelector使用的时候主要涉及下面几个类：

- LeaderSelector
- LeaderSelectorListener
- LeaderSelectorListenerAdapter
- CancelLeadershipException

核心类是LeaderSelector，它的构造函数如下：

```java
public LeaderSelector(CuratorFramework client, String mutexPath,LeaderSelectorListener listener)
public LeaderSelector(CuratorFramework client, String mutexPath, ThreadFactory threadFactory, Executor executor, LeaderSelectorListener listener)
```

类似LeaderLatch,LeaderSelector必须start: `leaderSelector.start();` 一旦启动，当实例取得领导权时你的listener的takeLeadership()方法被调用。当这个方法执行完后，该实例就会放弃 leader 的执行权。

<b>注意：</b>当你不再使用LeaderSelector实例时，应该调用它的close方法。

示例代码如下，推荐继承 LeaderSelectorListenerAdapter 类并实现 Closeable 接口：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.leader.LeaderSelector;
import org.apache.curator.framework.recipes.leader.LeaderSelectorListenerAdapter;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.curator.utils.CloseableUtils;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.io.Closeable;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.Random;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * @author ZGY
 * @date 2019/12/25 17:10
 * @description Test09App, Curator 高级特性 leader 选举
 */
public class Test09App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test09App.class);

    @Test
    public void test() throws Exception {
        List<CuratorFramework> clients = new ArrayList<>();
        List<LeaderSelectorAdapter> examples = new ArrayList<>();
        for (int i = 0; i < 5; i++) {
            CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181",
                    new ExponentialBackoffRetry(20000, 3));
            clients.add(client);
            LeaderSelectorAdapter selectorAdapter = new LeaderSelectorAdapter(client, "/francis/leader", "Client #" + i);
            examples.add(selectorAdapter);
            // 连接 zookeeper 服务器
            client.start();
            // 开始执行 leader 选举
            selectorAdapter.start();
        }

        // 测试完毕后，关闭选举和会话
        TimeUnit.SECONDS.sleep(30);
        LOGGER.info("开始回收数据了哦！");
        for (LeaderSelectorAdapter example : examples) {
            CloseableUtils.closeQuietly(example);
        }
        for (CuratorFramework client : clients) {
            CloseableUtils.closeQuietly(client);
        }
    }

    private class LeaderSelectorAdapter extends LeaderSelectorListenerAdapter implements Closeable {

        private final String name;
        private final LeaderSelector leaderSelector;
        private final AtomicInteger leaderCount = new AtomicInteger();

        public LeaderSelectorAdapter(CuratorFramework client, String path, String name) {
            this.name = name;
            this.leaderSelector = new LeaderSelector(client, path, this);
            // 希望一个 selector 放弃 leader 后还要重新参与leader选举
            this.leaderSelector.autoRequeue();
        }

        public void start() throws IOException {
            leaderSelector.start();
        }

        @Override
        public void close() throws IOException {
            leaderSelector.close();
        }

        /**
         * 当某个实例成为 leader 后就会执行这个方法，当这个方法执行完后，该实例就会放弃 leader 的执行权。
         * @param client
         * @throws Exception
         */
        @Override
        public void takeLeadership(CuratorFramework client) throws Exception {
            final int waitSeconds = new Random().nextInt(5);
            LOGGER.info("{} 现在是 leader，接下来我会一直当 leader {} 秒钟，除开这一次，我已经当过 {} 次 leader 了！", name, waitSeconds, leaderCount.getAndIncrement());
            TimeUnit.SECONDS.sleep(waitSeconds);
        }
    }
}
```

**为什么推荐继承 LeaderSelectorListenerAdapter 类来实现 leader 选举？**

查看 LeaderSelectorListenerAdapter 源代码如下：

```java
/**
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *   http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package org.apache.curator.framework.recipes.leader;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.state.ConnectionState;

/**
 * An implementation of {@link LeaderSelectorListener} that adds the recommended handling
 * for connection state problems
 */
public abstract class LeaderSelectorListenerAdapter implements LeaderSelectorListener
{
    @Override
    public void stateChanged(CuratorFramework client, ConnectionState newState)
    {
        if ( (newState == ConnectionState.SUSPENDED) || (newState == ConnectionState.LOST) )
        {
            throw new CancelLeadershipException();
        }
    }
}
```

这里的意思就是，如果当前实例发生SUSPENDED（暂停）或者LOST（丢失）连接问题，最好直接抛CancelLeadershipException，此时，leaderSelector实例会尝试中断并且取消正在执行takeLeadership（）方法的线程。

看 org.apache.curator.framework.recipes.leader.LeaderSelector 如下代码：

```java
@Override
public void stateChanged(CuratorFramework client, ConnectionState newState) {
    try
    {
        listener.stateChanged(client, newState);
    }
    catch ( CancelLeadershipException dummy )
    {
        leaderSelector.interruptLeadership();
    }
}
```

通过 debug 调试，当程序抛出 CancelLeadershipException 异常时会执行方法 `leaderSelector.interruptLeadership();`，该方法的作用就是让该实例放弃 leader 执行权。

## 分布式锁

分布式的锁全局同步， 这意味着任何一个时间点不会有两个客户端都拥有相同的锁。

### 可重入共享锁 Shared Reentrant Lock

**Shared意味着锁是全局可见的**， 客户端都可以请求锁。 Reentrant和JDK的ReentrantLock类似，即可重入， 意味着同一个客户端在拥有锁的同时，可以多次获取，不会被阻塞。 它是由类InterProcessMutex来实现。 它的构造函数为：

```java
public InterProcessMutex(CuratorFramework client, String path)
```

通过`acquire()`获得锁，并提供超时机制：

```java
public void acquire();
public boolean acquire(long time,TimeUnit unit);
```

通过`release()`方法释放锁。 InterProcessMutex 实例可以重用。

<b>特别提醒：</b>错误处理 还是强烈推荐你使用 ConnectionStateListener 处理连接状态的改变。 当连接LOST时你不再拥有锁。

示例代码如下：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.curator.utils.CloseableUtils;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.Random;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicBoolean;

/**
 * @author ZGY
 * @date 2019/12/26 14:04
 * @description Test10App
 */
public class Test10App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test10App.class);

    @Test
    public void test() throws InterruptedException {
        // 创建需要共享的资源对象
        final FakeLimitedResource resource = new FakeLimitedResource();
        // 创建线程池对象
        ExecutorService executorService = Executors.newFixedThreadPool(2);

        final ExponentialBackoffRetry retry = new ExponentialBackoffRetry(1000, 3);

        for (int i = 0; i < 2; i++) {
            final int index = i;
            Callable<Void> task = () -> {
                CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 5000, 5000, retry);
                try {
                    client.start();
                    InterProcessMutexDemo mutexDemo = new InterProcessMutexDemo(client, "/examples/locks", resource, "我是客户端: " + index);
                    for (int j = 0; j < 10; j++) {
                        mutexDemo.doWork(10, TimeUnit.SECONDS);
                    }
                } catch (Exception e) {
                    LOGGER.error("程序出现异常！", e);
                } finally {
                    // 关闭会话
                    CloseableUtils.closeQuietly(client);
                }
                return null;
            };

            // 交给线程池执行
            executorService.submit(task);
        }

        // 当线程池中的所有任务执行完后，关闭线程池
        executorService.shutdown();

        // 等待除主线程外其他线程都执行完毕
        executorService.awaitTermination(10, TimeUnit.MINUTES);
    }

    /**
     * 共享资源
     */
    private class FakeLimitedResource {

        private final AtomicBoolean inUse = new AtomicBoolean(false);

        public void use() throws InterruptedException {
            // 如果设置值失败
            if (!inUse.compareAndSet(false, true)) {
                throw new RuntimeException("该资源的原始值不是 false，所以设置值失败！");
            }

            try {
                // 模拟程序复杂业务
                int seconds = new Random().nextInt(3);
                LOGGER.info("模拟程序复杂业务，耗时 {} 秒", seconds);
                TimeUnit.SECONDS.sleep(seconds);
            } finally {
                // 强制将 inUse 设置为 false
                inUse.set(false);
            }
        }
    }

    /**
     * 锁
     */
    private class InterProcessMutexDemo {

        private InterProcessMutex lock;
        private FakeLimitedResource resource;
        private String clientName;

        public InterProcessMutexDemo(CuratorFramework client, String lockPath, FakeLimitedResource resource, String clientName) {
            this.lock = new InterProcessMutex(client, lockPath);
            this.resource = resource;
            this.clientName = clientName;
        }

        public void doWork(long time, TimeUnit unit) throws Exception {
            /*try {
                // 如果获取不到锁
                LOGGER.info("{}， 第一次加锁", this.clientName);
                if (!lock.acquire(time, unit)) {
                    throw new RuntimeException(this.clientName + "，第一次获取锁失败！");
                }

                // 第二次获取锁测试可重入性
                try {
                    LOGGER.info("{}， 第二次加锁", this.clientName);
                    if (!lock.acquire(time, unit)) {
                        throw new RuntimeException(this.clientName + "，第二次获取锁失败！");
                    }
                    LOGGER.info("{} 获取到了锁！", this.clientName);
                    resource.use();
                } finally {
                    LOGGER.info("{} 资源使用完毕，释第二次加的锁！", this.clientName);
                    lock.release();
                }

                LOGGER.info("{} 获取到了锁！", this.clientName);
                resource.use();
            } finally {
                LOGGER.info("{} 资源使用完毕，释第一次加的锁！", this.clientName);
                lock.release();
            }*/

            try {
                // 如果获取不到锁
                LOGGER.info("{}， 第一次加锁", this.clientName);
                if (!lock.acquire(time, unit)) {
                    throw new RuntimeException(this.clientName + "，第一次获取锁失败！");
                }

                LOGGER.info("{}， 第二次加锁", this.clientName);
                if (!lock.acquire(time, unit)) {
                    throw new RuntimeException(this.clientName + "，第二次获取锁失败！");
                }

                LOGGER.info("{} 获取到了锁！", this.clientName);
                resource.use();
            } finally {
                LOGGER.info("{} 资源使用完毕，释第一次加的锁！", this.clientName);
                lock.release();
                lock.release();
            }
        }
    }
}
```

<b>这里有个地方特别注意：</b>在上面的示例代码中，加多少次锁就要释放几次锁，不然在下次获取锁的时候就会高概率的抛异常。原因是 Curator 的分布式锁机制导致的，它创建的 zookeeper 节点的类型是**临时顺序**节点，而在获取节点的时候会先获取最先创建（序号最小）的节点。如果加 n 次锁过后没有释放 n 次锁，那么这个节点就依然还存活在 zookeeper 节点中，因此导致其他线程获取其他节点（序号不是最小的节点）时就获取不到锁，获取不了锁就会抛异常。之所以说是高概率而不是绝对，是因为在上面示例代码中是多线程环境，在代码中有这样一段代码如下：

```java
// 关闭会话
CloseableUtils.closeQuietly(client);
```

所以当某个线程的客户端被关闭后，也相当于 zookeeper 中对应的节点也没有了，自然另外的线程就能获取到锁了。

这个代码我研究了好久，各种猜测，最后 debug 才明白是怎么回事，所以特地记录在此！

### 不可重入共享锁 Shared Lock

这个锁和上面的 InterProcessMutex 相比，就是少了 Reentrant 的功能，也就意味着它不能在同一个线程中重入。这个类是 `InterProcessSemaphoreMutex` ,使用方法和 `InterProcessMutex` 类似。

示例代码如下：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.locks.InterProcessSemaphoreMutex;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.curator.utils.CloseableUtils;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.Random;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;

/**
 * @author ZGY
 * @date 2019/12/27 14:43
 * @description Test11App, Curator 高级特性——不可重入共享锁 示例代码
 */
public class Test11App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test11App.class);

    @Test
    public void test() throws InterruptedException {
        // 创建共享资源对象
        final FakeLimitedResource resource = new FakeLimitedResource();
        // 创建线程池
        ExecutorService executorService = Executors.newFixedThreadPool(5);
        // 重试策略对象
        final ExponentialBackoffRetry retry = new ExponentialBackoffRetry(1000, 3);

        for (int i = 0; i < 5; i++) {
            final int index = i;
            Runnable task = () -> {
                CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 5000, 5000, retry);
                try {
                    client.start();
                    InterProcessSemaphoreMutexDemo mutexDemo = new InterProcessSemaphoreMutexDemo(client, "/examples/locks", resource, "客户端" + index);
                    for (int j = 0; j < 5; j++) {
                        mutexDemo.doWork(5, TimeUnit.SECONDS);
                    }
                } finally {
                    CloseableUtils.closeQuietly(client);
                }
            };

            // 交给线程池执行
            executorService.submit(task);
        }

        // 当线程池中的所有任务执行完后，关闭线程池
        executorService.shutdown();
        // 等待除主线程外其他线程都执行完毕
        executorService.awaitTermination(10, TimeUnit.MINUTES);
    }

    private class InterProcessSemaphoreMutexDemo {

        /**
         * 不可重入贡献锁对象
         */
        private InterProcessSemaphoreMutex lock;
        /**
         * 共享资源对象
         */
        private FakeLimitedResource resource;
        /**
         * 客户端名称
         */
        private String clientName;

        /**
         * @param client 连接 zookeeper 的客户端对象
         * @param lockPath 锁路径
         * @param resource 共享资源对象
         * @param clientName 客户名称
         */
        public InterProcessSemaphoreMutexDemo(CuratorFramework client, String lockPath, FakeLimitedResource resource, String clientName) {
            this.lock = new InterProcessSemaphoreMutex(client, lockPath);
            this.resource = resource;
            this.clientName = clientName;
        }

        /**
         * 业务功能
         * @param time
         * @param unit
         */
        public void doWork(long time, TimeUnit unit) {
            try {
                LOGGER.info("{} 第一次获取锁", this.clientName);
                if (!lock.acquire(time, unit)) {
                    LOGGER.info("{} 第一次没有获取到锁！", this.clientName);
                    return;
                }

                // 这个示例代码是不可重入锁，所以这里会获取不了锁
                LOGGER.info("{} 第二次获取锁", this.clientName);
                if (!lock.acquire(time, TimeUnit.SECONDS)) {
                    LOGGER.info("{} 第二次没有获取到锁！", this.clientName);
                    return;
                }

                LOGGER.info("{} 获取到了锁，马上开始使用资源！", this.clientName);
                this.resource.use();
            } catch (Exception e) {
                LOGGER.error("业务功能发生了异常！", e);
                return;
            } finally {
                // 释放锁
                try {
                    LOGGER.info("{} 业务功能处理完了，马上开始释放锁！", this.clientName);
                    if (lock.isAcquiredInThisProcess()) {
                        lock.release();
                    }
                    if (lock.isAcquiredInThisProcess()) {
                        lock.release();
                    }
                } catch (Exception e) {
                    LOGGER.info("释放锁发生了异常！", e);
                }
            }
        }
    }

    /**
     * 共享资源
     */
    private class FakeLimitedResource {

        private final AtomicBoolean inUse = new AtomicBoolean(false);

        public void use() throws InterruptedException {
            // 如果设置值失败
            if (!inUse.compareAndSet(false, true)) {
                throw new RuntimeException("该资源的原始值不是 false，所以设置值失败！");
            }

            try {
                // 模拟程序复杂业务
                int seconds = new Random().nextInt(3);
                LOGGER.info("模拟程序复杂业务，耗时 {} 秒", seconds);
                TimeUnit.SECONDS.sleep(seconds);
            } finally {
                // 强制将 inUse 设置为 false
                inUse.set(false);
            }
        }
    }
}
```

运行后发现，有且只有一个 client 成功获取第一个锁(第一个 acquire() 方法返回 true )，然后它自己阻塞在第二个 acquire() 方法，获取第二个锁失败，方法返回 false。这样也就验证了 `InterProcessSemaphoreMutex` 实现的锁是不可重入的。

### 可重入读写锁 Shared Reentrant Read Write Lock

类似JDK的**ReentrantReadWriteLock**。一个读写锁管理一对相关的锁。一个负责读操作，另外一个负责写操作。读操作在写锁没被使用时可同时由多个进程使用，而写锁在使用时不允许读(阻塞)。

此锁是可重入的。**一个拥有写锁的线程可重入读锁，但是读锁却不能进入写锁**。这也意味着**写锁可以降级成读锁， 比如请求写锁 —>请求读锁—>释放读锁 —->释放写锁**。从读锁升级成写锁是不行的。

可重入读写锁主要由两个类实现：`InterProcessReadWriteLock`、`InterProcessMutex`。使用时首先创建一个`InterProcessReadWriteLock`实例，然后再根据你的需求得到读锁或者写锁，读写锁的类型是`InterProcessMutex`。

示例代码如下：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.framework.recipes.locks.InterProcessReadWriteLock;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.curator.utils.CloseableUtils;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Random;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;

/**
 * @author ZGY
 * @date 2019/12/30 13:59
 * @description Test12App, 可重入读写锁 Shared Reentrant Read Write Lock
 */
public class Test12App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test12App.class);

    @Test
    public void test() throws InterruptedException {
        ExponentialBackoffRetry retry = new ExponentialBackoffRetry(1000, 3);

        final FakeLimitedResource resource = new FakeLimitedResource();
        ExecutorService executorService = Executors.newFixedThreadPool(5);
        for (int i = 0; i < 5; i++) {
            final int index = i;
            Runnable task = () -> {
                final CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 5000, 5000, retry);
                try {
                    client.start();
                    ReentrantReadWriteLockDemo demo = new ReentrantReadWriteLockDemo(client, "/examples/locks", resource, "客户端" + index);
                    demo.doWork(5, TimeUnit.SECONDS);
                } finally {
                    CloseableUtils.closeQuietly(client);
                }
            };
            executorService.execute(task);
        }

        executorService.shutdown();

        executorService.awaitTermination(5, TimeUnit.MINUTES);
    }

    private class ReentrantReadWriteLockDemo {

        private final InterProcessReadWriteLock lock;
        private final InterProcessMutex readLock;
        private final InterProcessMutex writeLock;
        private final FakeLimitedResource resource;
        private final String clientName;

        public ReentrantReadWriteLockDemo(CuratorFramework client, String lockPath, FakeLimitedResource resource, String clientName) {
            this.lock = new InterProcessReadWriteLock(client, lockPath);
            this.readLock = this.lock.readLock();
            this.writeLock = this.lock.writeLock();
            this.resource = resource;
            this.clientName = clientName;
        }

        /**
         * 业务功能
         * @param time
         * @param unit
         */
        public void doWork(long time, TimeUnit unit) {
            try {
                if (!this.writeLock.acquire(time, unit)) {
                    LOGGER.info("{} 不能获取到写锁", this.clientName);
                }
                LOGGER.info("{} 已得到写锁", this.clientName);
                if (!this.readLock.acquire(time, unit)) {
                    LOGGER.info("{} 不能获取到读锁", this.clientName);
                }
                LOGGER.info("{} 已得到读锁", this.clientName);

                LOGGER.info("开始使用共享资源");
                resource.use();
                LOGGER.info("共享资源使用完毕");
            } catch (Exception e) {
                LOGGER.error("public void doWork(long time, TimeUnit unit) 方法发生异常！", e);
            } finally {
                try {
                    LOGGER.info("业务处理完毕，开始释放锁");
                    if (this.readLock.isAcquiredInThisProcess()) {
                        this.readLock.release();
                    }
                    if (this.writeLock.isAcquiredInThisProcess()) {
                        this.writeLock.release();
                    }
                    LOGGER.info("锁释放成功！");
                } catch (Exception e) {
                    LOGGER.error("public void doWork(long time, TimeUnit unit) 方法释放锁发生异常！", e);
                }
            }
        }
    }

    /**
     * 共享资源
     */
    private class FakeLimitedResource {

        private final AtomicBoolean inUse = new AtomicBoolean(false);

        public void use() throws InterruptedException {
            // 如果设置值失败
            if (!inUse.compareAndSet(false, true)) {
                throw new RuntimeException("该资源的原始值不是 false，所以设置值失败！");
            }

            try {
                // 模拟程序复杂业务
                int seconds = new Random().nextInt(3);
                LOGGER.info("模拟程序复杂业务，耗时 {} 秒", seconds);
                TimeUnit.SECONDS.sleep(seconds);
            } finally {
                // 强制将 inUse 设置为 false
                inUse.set(false);
            }
        }
    }
}
```

### 信号量 Shared Semaphore

一个计数的信号量类似JDK的Semaphore。 JDK中Semaphore维护的一组许可(**permits**)，而Curator中称之为租约(**Lease**)。 有两种方式可以决定semaphore的最大租约数。第一种方式是用户给定path并且指定最大LeaseSize。第二种方式用户给定path并且使用`SharedCountReader`类。**如果不使用SharedCountReader, 必须保证所有实例在多进程中使用相同的(最大)租约数量,否则有可能出现A进程中的实例持有最大租约数量为10，但是在B进程中持有的最大租约数量为20，此时租约的意义就失效了。**

这次调用`acquire()`会返回一个租约对象。 客户端必须在finally中close这些租约对象，否则这些租约会丢失掉。 但是， 但是，如果客户端session由于某种原因比如crash丢掉， 那么这些客户端持有的租约会自动close， 这样其它客户端可以继续使用这些租约。 租约还可以通过下面的方式返还：

```java
public void returnAll(Collection<Lease> leases)
public void returnLease(Lease lease)
```

注意你可以一次性请求多个租约，如果Semaphore当前的租约不够，则请求线程会被阻塞。 同时还提供了超时的重载方法。

```java
public Lease acquire()
public Collection<Lease> acquire(int qty)
public Lease acquire(long time, TimeUnit unit)
public Collection<Lease> acquire(int qty, long time, TimeUnit unit)
```

Shared Semaphore使用的主要类包括下面几个：

- InterProcessSemaphoreV2
- Lease
- SharedCountReader

示例代码如下：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.locks.InterProcessSemaphoreV2;
import org.apache.curator.framework.recipes.locks.Lease;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Collection;
import java.util.Random;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;

/**
 * @author ZGY
 * @date 2019/12/30 14:55
 * @description Test13App, 信号量—Shared Semaphore
 */
public class Test13App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test13App.class);

    @Test
    public void test() throws Exception {
        FakeLimitedResource resource = new FakeLimitedResource();
        ExponentialBackoffRetry retry = new ExponentialBackoffRetry(1000, 3);
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 10000, 5000, retry);
        client.start();

        // 定义信号量变量，最大租约为 10
        InterProcessSemaphoreV2 semaphore = new InterProcessSemaphoreV2(client, "/examples/locks", 10);
        // 获取 5 个租约
        Collection<Lease> leases = semaphore.acquire(5);
        LOGGER.info("leases 租约数量为：{}", leases.size());

        // 获取 1 个租约
        Lease lease = semaphore.acquire();
        LOGGER.info("lease: [{}]", lease);

        // 使用共享资源
        resource.use();

        // 在 6 秒内获取 5 个租约，如果获取不了就返回 null
        Collection<Lease> leases2 = semaphore.acquire(5, 6, TimeUnit.SECONDS);
        if (leases2 != null) {
            LOGGER.info("leases2 租约数量为：{}", leases2.size());
        } else {
            LOGGER.info("在 6 秒内没有获取到 5 个租约，所以 leases2 为 null");
        }

        // 把使用的租约还给 semaphore
        semaphore.returnLease(lease);
        semaphore.returnAll(leases);
        if (leases2 != null) {
            semaphore.returnAll(leases2);
        }
    }

    /**
     * 共享资源
     */
    private class FakeLimitedResource {

        private final AtomicBoolean inUse = new AtomicBoolean(false);

        public void use() throws InterruptedException {
            // 如果设置值失败
            if (!inUse.compareAndSet(false, true)) {
                throw new RuntimeException("该资源的原始值不是 false，所以设置值失败！");
            }

            try {
                // 模拟程序复杂业务
                int seconds = new Random().nextInt(3);
                LOGGER.info("模拟程序复杂业务，耗时 {} 秒", seconds);
                TimeUnit.SECONDS.sleep(seconds);
            } finally {
                // 强制将 inUse 设置为 false
                inUse.set(false);
            }
        }
    }
}
```

首先我们先获得了5个租约， 最后我们把它还给了semaphore。 接着请求了一个租约，因为semaphore还有5个租约，所以请求可以满足，返回一个租约，还剩4个租约。 然后再请求一个租约，因为租约不够，**阻塞到超时，还是没能满足，返回结果为null(租约不足会阻塞到超时，然后返回null，不会主动抛出异常；如果不设置超时时间，会一致阻塞)。**

### 多共享锁对象 Multi Shared Lock

Multi Shared Lock是一个锁的容器。 当调用`acquire()`， 所有的锁都会被`acquire()`，如果请求失败，所有的锁都会被release。 同样调用release时所有的锁都被release(**失败被忽略**)。 基本上，它就是组锁的代表，在它上面的请求释放操作都会传递给它包含的所有的锁。

主要涉及两个类：

- InterProcessMultiLock
- InterProcessLock

它的构造函数需要包含的锁的集合，或者一组ZooKeeper的path。

```java
public InterProcessMultiLock(List<InterProcessLock> locks)
public InterProcessMultiLock(CuratorFramework client, List<String> paths)
```

示例代码如下：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.locks.InterProcessLock;
import org.apache.curator.framework.recipes.locks.InterProcessMultiLock;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.framework.recipes.locks.InterProcessSemaphoreMutex;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.Arrays;
import java.util.Random;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;

/**
 * @author ZGY
 * @date 2019/12/30 16:03
 * @description Test14App, 多共享锁对象 —Multi Shared Lock
 */
public class Test14App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test14App.class);

    @Test
    public void test() throws Exception {
        FakeLimitedResource resource = new FakeLimitedResource();
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 10000, 5000, new ExponentialBackoffRetry(5000, 3));
        client.start();

        // 创建一个可重入锁对象
        InterProcessLock lock = new InterProcessMutex(client, "/examples/locks");
        // 创建一个不可重入锁对象
        InterProcessLock lock2 = new InterProcessSemaphoreMutex(client, "/examples/locks2");
        // 创建多共享锁对象
        InterProcessLock lock3 = new InterProcessMultiLock(Arrays.asList(lock, lock2));

        if (!lock3.acquire(2000, TimeUnit.SECONDS)) {
            LOGGER.info("获取所有的锁失败！");
            return;
        }

        LOGGER.info("获取所有的锁成功！lock 是否获取到了锁：{}, lock2 是否获取到了锁：{}", lock.isAcquiredInThisProcess(), lock2.isAcquiredInThisProcess());

        try {
            // 使用共享资源
            resource.use();
        } finally {
            // 释放所有锁
            lock3.release();
            LOGGER.info("释放所有的锁成功！lock 是否获取到了锁：{}, lock2 是否获取到了锁：{}", lock.isAcquiredInThisProcess(), lock2.isAcquiredInThisProcess());
        }
    }

    /**
     * 共享资源
     */
    private class FakeLimitedResource {

        private final AtomicBoolean inUse = new AtomicBoolean(false);

        public void use() throws InterruptedException {
            // 如果设置值失败
            if (!inUse.compareAndSet(false, true)) {
                throw new RuntimeException("该资源的原始值不是 false，所以设置值失败！");
            }

            try {
                // 模拟程序复杂业务
                int seconds = new Random().nextInt(3);
                LOGGER.info("模拟程序复杂业务，耗时 {} 秒", seconds);
                TimeUnit.SECONDS.sleep(seconds);
            } finally {
                // 强制将 inUse 设置为 false
                inUse.set(false);
            }
        }
    }
}
```

新建一个`InterProcessMultiLock`， 包含一个重入锁和一个非重入锁。 调用`acquire()`后可以看到线程同时拥有了这两个锁。 调用`release()`看到这两个锁都被释放了。

**最后再重申一次， 强烈推荐使用ConnectionStateListener监控连接的状态，当连接状态为LOST，锁将会丢失。**

## 分布式计数器

顾名思义，计数器是用来计数的, 利用ZooKeeper可以实现一个集群共享的计数器。 只要使用相同的path就可以得到最新的计数器值， 这是由ZooKeeper的一致性保证的。Curator有两个计数器， 一个是用int来计数(`SharedCount`)，一个用long来计数(`DistributedAtomicLong`)。

### 分布式int计数器(SharedCount)

这个类使用int类型来计数。 主要涉及三个类。

- SharedCount
- SharedCountReader
- SharedCountListener

`SharedCount`代表计数器， 可以为它增加一个`SharedCountListener`，当计数器改变时此Listener可以监听到改变的事件，而`SharedCountReader`可以读取到最新的值， 包括字面值和带版本信息的值VersionedValue。

示例代码如下：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.shared.SharedCount;
import org.apache.curator.framework.recipes.shared.SharedCountListener;
import org.apache.curator.framework.recipes.shared.SharedCountReader;
import org.apache.curator.framework.state.ConnectionState;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.ArrayList;
import java.util.List;
import java.util.Random;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

/**
 * @author ZGY
 * @date 2019/12/30 16:56
 * @description Test15App, 分布式int计数器—SharedCount
 */
public class Test15App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test15App.class);

    @Test
    public void test() throws Exception {
        final Random random = new Random();
        SharedCounterDemo demo = new SharedCounterDemo();
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 10000, 5000, new ExponentialBackoffRetry(2000, 3));
        client.start();
        // 创建计数器对象
        SharedCount sharedCount = new SharedCount(client, "/examples/counter", 0);
        // 给计数器对象添加监听器
        sharedCount.addListener(demo);
        // 启动计数器
        sharedCount.start();

        List<SharedCount> sharedCounts = new ArrayList<>();
        ExecutorService executorService = Executors.newFixedThreadPool(5);
        for (int i = 0; i < 5; i++) {
            final SharedCount SharedCount = new SharedCount(client, "/examples/counter", 0);
            sharedCounts.add(sharedCount);
            Callable<Void> task = () -> {
                SharedCount.start();
                TimeUnit.SECONDS.sleep(random.nextInt(10));
                boolean b = SharedCount.trySetCount(sharedCount.getVersionedValue(), random.nextInt(100));
                while (!b) {
                    b = SharedCount.trySetCount(sharedCount.getVersionedValue(), random.nextInt(100));
                }
                LOGGER.info("修改数据，b: [{}]", b);
                return null;
            };
            executorService.submit(task);
        }

        executorService.shutdown();
        executorService.awaitTermination(10, TimeUnit.MINUTES);

        // 关闭计数器
        for (SharedCount count : sharedCounts) {
            count.close();
        }

        sharedCount.close();
    }

    private class SharedCounterDemo implements SharedCountListener {
        /**
         * 数字更改时触发
         * @param sharedCount
         * @param newCount
         * @throws Exception
         */
        @Override
        public void countHasChanged(SharedCountReader sharedCount, int newCount) throws Exception {
            LOGGER.info("数据被修改为：{}", newCount);
        }

        /**
         * 状态更改时触发
         * @param client
         * @param newState
         */
        @Override
        public void stateChanged(CuratorFramework client, ConnectionState newState) {
            LOGGER.info("状态被修改为了：{}", newState);
        }
    }
}
```

在这个例子中，我们使用`baseCount`来监听计数值(`addListener`方法来添加SharedCountListener )。 任意的SharedCount， 只要使用相同的path，都可以得到这个计数值。 然后我们使用5个线程为计数值增加一个10以内的随机数。相同的path的SharedCount对计数值进行更改，将会回调给`baseCount`的SharedCountListener。

```java
boolean b = SharedCount.trySetCount(sharedCount.getVersionedValue(), random.nextInt(100));
while (!b) {
    b = SharedCount.trySetCount(sharedCount.getVersionedValue(), random.nextInt(100));
}
```

这里我们使用`trySetCount`去设置计数器。 **第一个参数提供当前的VersionedValue,如果期间其它client更新了此计数值， 你的更新可能不成功， 更新不成功会返回 false。所以失败了你可以尝试再更新一次。 而`setCount`是强制更新计数器的值**。

注意计数器必须`start`,使用完之后必须调用`close`关闭它。

强烈推荐使用`ConnectionStateListener`。 在本例中`SharedCountListener`扩展`ConnectionStateListener`。

### 分布式long计数器(DistributedAtomicLong)

再看一个Long类型的计数器。 除了计数的范围比`SharedCount`大了之外， 它首先尝试使用乐观锁的方式设置计数器， 如果不成功(比如期间计数器已经被其它client更新了)， 它使用`InterProcessMutex`方式来更新计数值。

可以从它的内部实现`DistributedAtomicValue.trySet()`中看出：

```java
AtomicValue < byte[] > trySet(MakeValue makeValue) throws Exception {
    MutableAtomicValue < byte[] > result = new MutableAtomicValue < byte[] > (null, null, false);
    tryOptimistic(result, makeValue);
    if(!result.succeeded() && (mutex != null)) {
        tryWithMutex(result, makeValue);
    }
    return result;
}
```

此计数器有一系列的操作：

- get(): 获取当前值
- increment()： 加一
- decrement(): 减一
- add()： 增加特定的值
- subtract(): 减去特定的值
- trySet(): 尝试设置计数值
- forceSet(): 强制设置计数值

你**必须**检查返回结果的`succeeded()`， 它代表此操作是否成功。 如果操作成功， `preValue()`代表操作前的值， `postValue()`代表操作后的值。

示例代码如下：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.atomic.AtomicValue;
import org.apache.curator.framework.recipes.atomic.DistributedAtomicLong;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.curator.retry.RetryNTimes;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

/**
 * @author ZGY
 * @date 2019/12/30 17:29
 * @description Test16App, 分布式long计数器—DistributedAtomicLong
 */
public class Test16App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test16App.class);

    @Test
    public void test() throws InterruptedException {
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 10000, 5000, new ExponentialBackoffRetry(2000, 3));
        client.start();

        ExecutorService executorService = Executors.newFixedThreadPool(5);
        for (int i = 0; i < 5; i++) {
            /**
             * 重试策略对象，参数解释如下：
             * 第一个参数：重试次数
             * 第二个参数：重试之间的睡眠时间
             */
            RetryNTimes retryNTimes = new RetryNTimes(2, 2);
            final DistributedAtomicLong atomicLong = new DistributedAtomicLong(client, "/examples/counter", retryNTimes);
            Callable<Void> task = () -> {
                try {
                    // 加 1
                    AtomicValue<Long> value = atomicLong.increment();
                    if (value.succeeded()) {
                        LOGGER.info("修改前的值为：{}， 修改后的值为：{}", value.preValue(), value.postValue());
                    } else {
                        LOGGER.info("修改失败");
                    }
                } catch (Exception e) {
                    LOGGER.error("程序出现异常！", e);
                }
                return null;
            };

            executorService.submit(task);
        }

        executorService.shutdown();
        executorService.awaitTermination(10, TimeUnit.MINUTES);
    }
}
```

## 分布式队列

使用Curator也可以简化Ephemeral Node (**临时节点**)的操作。Curator也提供ZK Recipe的分布式队列实现。 利用ZK的 PERSISTENTS_EQUENTIAL节点， 可以保证放入到队列中的项目是按照顺序排队的。 如果单一的消费者从队列中取数据， 那么它是先入先出的，这也是队列的特点。 如果你严格要求顺序，你就的使用单一的消费者，可以使用Leader选举只让Leader作为唯一的消费者。

但是， 根据Netflix的Curator作者所说， ZooKeeper真心不适合做Queue，或者说ZK没有实现一个好的Queue，详细内容可以看 [Tech Note 4](https://cwiki.apache.org/confluence/display/CURATOR/TN4)， 原因有五：

1. ZK有1MB 的传输限制。 实践中ZNode必须相对较小，而队列包含成千上万的消息，非常的大。
2. 如果有很多节点，ZK启动时相当的慢。 而使用queue会导致好多ZNode. 你需要显著增大 initLimit 和 syncLimit.
3. ZNode很大的时候很难清理。Netflix不得不创建了一个专门的程序做这事。
4. 当很大量的包含成千上万的子节点的ZNode时， ZK的性能变得不好
5. ZK的数据库完全放在内存中。 大量的Queue意味着会占用很多的内存空间。

尽管如此， Curator还是创建了各种Queue的实现。 如果Queue的数据量不太多，数据量不太大的情况下，酌情考虑，还是可以使用的。

### 分布式队列(DistributedQueue)

DistributedQueue是最普通的一种队列。 它设计以下四个类：

- QueueBuilder - 创建队列使用QueueBuilder,它也是其它队列的创建类
- QueueConsumer - 队列中的消息消费者接口
- QueueSerializer - 队列消息序列化和反序列化接口，提供了对队列中的对象的序列化和反序列化
- DistributedQueue - 队列实现类

QueueConsumer是消费者，它可以接收队列的数据。处理队列中的数据的代码逻辑可以放在QueueConsumer.consumeMessage()中。

**正常情况下先将消息从队列中移除，再交给消费者消费。但这是两个步骤，不是原子的。**可以调用Builder的lockPath()消费者加锁，当消费者消费数据时持有锁，这样其它消费者不能消费此消息。如果消费失败或者进程死掉，消息可以交给其它进程。这会带来一点性能的损失。最好还是单消费者模式使用队列。

示例代码如下：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.queue.DistributedQueue;
import org.apache.curator.framework.recipes.queue.QueueBuilder;
import org.apache.curator.framework.recipes.queue.QueueConsumer;
import org.apache.curator.framework.recipes.queue.QueueSerializer;
import org.apache.curator.framework.state.ConnectionState;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.concurrent.TimeUnit;

/**
 * @author ZGY
 * @date 2020/1/3 14:43
 * @description Test17App, 分布式队列—DistributedQueue
 */
public class Test17App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test17App.class);

    @Test
    public void test() throws Exception {
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 10000, 5000, new ExponentialBackoffRetry(1000, 3));
        CuratorFramework client2 = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 10000, 5000, new ExponentialBackoffRetry(1000, 3));

        // 客户端启动
        client.start();
        // 客户端2启动
        client2.start();

        /**
         * 是线程不安全的
         */
        // DistributedQueue<String> queue = QueueBuilder.builder(client, createQueueConsumer("消费者A"), creQueueSerializer(), "/example/queue").buildQueue();
        // DistributedQueue<String> queue2 = QueueBuilder.builder(client, createQueueConsumer("消费者B"), creQueueSerializer(), "/example/queue").buildQueue();

        /**
         * 正常情况下先将消息从队列中移除，再交给消费者消费。但这是两个步骤，不是原子的。
         * 可以调用Builder的lockPath()消费者加锁，当消费者消费数据时持有锁，这样其它消费者不能消费此消息。
         * 如果消费失败或者进程死掉，消息可以交给其它进程。这会带来一点性能的损失。最好还是单消费者模式使用队列。
         */
        DistributedQueue<String> queue = QueueBuilder.builder(client, createQueueConsumer("消费者A"), creQueueSerializer(), "/example/queue").lockPath("/example/lock").buildQueue();
        DistributedQueue<String> queue2 = QueueBuilder.builder(client, createQueueConsumer("消费者B"), creQueueSerializer(), "/example/queue").lockPath("/example/lock").buildQueue();

        // 消息队列启动
        queue.start();
        // 消息队列2启动
        queue2.start();

        for (int i = 0; i < 100; i++) {
            // 往消息队列中放数据
            queue.put("Test-A-" + i);
            TimeUnit.MILLISECONDS.sleep(10);
            // 往消息队列2中放数据
             queue2.put("Test-B-" + i);
        }

        TimeUnit.SECONDS.sleep(20);

        // 关闭消息队列
        queue.close();
        // 关闭消息队列2
        queue2.close();

        // 客户端关闭
        client.close();
        // 客户端2关闭
        client2.close();
        LOGGER.info("程序执行结束！");
    }

    /**
     * 定义队列消费者
     * @return
     */
    private QueueConsumer<String> createQueueConsumer(final String name) {
        return new QueueConsumer<String>() {
            /**
             * 消费消息时调用的方法
             * @param message
             * @throws Exception
             */
            @Override
            public void consumeMessage(String message) throws Exception {
                LOGGER.info("{} 消费消息：{}", name, message);
            }

            /**
             * 状态改变时调用的方法
             * @param client
             * @param newState
             */
            @Override
            public void stateChanged(CuratorFramework client, ConnectionState newState) {
                LOGGER.info("连接状态为：{}", newState.name());
            }
        };
    }

    /**
     * 队列消息序列化实现类
     * @return
     */
    private QueueSerializer<String> creQueueSerializer() {
        return new QueueSerializer<String>() {
            /**
             * 序列化
             * @param item
             * @return
             */
            @Override
            public byte[] serialize(String item) {
                return item.getBytes();
            }

            /**
             * 反序列化
             * @param bytes
             * @return
             */
            @Override
            public String deserialize(byte[] bytes) {
                return new String(bytes);
            }
        };
    }
}
```

例子中定义了两个分布式队列和两个消费者，因为PATH是相同的，通常会存在消费者抢占消费消息的情况，但是我使用了 `lockPath(/example/lock)` 方法，所以不会，如果将该代码注释掉，然后方案前面注释的代码，就会出现消费者抢占消费消息的情况（但是我测试了 N 次，也没有出现这个这种情况）。

### 带Id的分布式队列(DistributedIdQueue)

DistributedIdQueue和上面的队列类似，**但是可以为队列中的每一个元素设置一个ID**。 可以通过ID把队列中任意的元素移除。 它涉及几个类：

- QueueBuilder
- QueueConsumer
- QueueSerializer
- DistributedQueue

通过下面方法创建：

```java
builder.buildIdQueue()
```

放入元素时：

```java
queue.put(aMessage, messageId);
```

移除元素时：

```java
int numberRemoved = queue.remove(messageId);
```

示例代码如：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.queue.*;
import org.apache.curator.framework.state.ConnectionState;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.curator.utils.CloseableUtils;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.Random;
import java.util.concurrent.TimeUnit;

/**
 * @author ZGY
 * @date 2020/1/8 13:55
 * @description Test18App，带Id的分布式队列—DistributedIdQueue
 */
public class Test18App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test18App.class);

    /**
     * 测试方法
     */
    @Test
    public void test() throws Exception {
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 10000, 5000, new ExponentialBackoffRetry(5000, 3));
        client.getCuratorListenable().addListener((client1, event) -> {
            LOGGER.info("监听客户端连接事件，事件名为：{}", event.getType().name());
        });
        client.start();

        // 创建队列对象
        DistributedIdQueue<String> queue = QueueBuilder.builder(client, createQueueConsumer(), createQueueSerializer(), "/example/queue").buildIdQueue();
        // 启动队列
        queue.start();

        Random random = new Random();
        for (int i = 0; i < 10; i++) {
            queue.put("Test-" + i, "Id-" + i);
            /**
             * 往队列中放入一个数据后睡眠任意秒，该线程放弃 CPU 执行权。队列开始消费，如果该线程在睡眠任意秒后重新获得 CPU 执行权，但是队列的数据没有被消费，
             * 此时调用 queue.remove("Id-" + i) 方法把队列中的指定数据删除了，该消费就不会被消费了。
             */
            TimeUnit.SECONDS.sleep(random.nextInt(2));
            queue.remove("Id-" + i);
        }

        TimeUnit.SECONDS.sleep(2);

        LOGGER.info("程序执行完毕！准备回收资源");

        CloseableUtils.closeQuietly(queue);
        CloseableUtils.closeQuietly(client);

        LOGGER.info("资源回收完毕！");
    }

    /**
     * 创建序列化和反序列化对象
     * @return
     */
    private QueueSerializer<String> createQueueSerializer() {
        return new QueueSerializer<String>() {
            @Override
            public byte[] serialize(String item) {
                return item.getBytes();
            }

            @Override
            public String deserialize(byte[] bytes) {
                return new String(bytes);
            }
        };
    }

    /**
     * 创建消费者对象
     * @return
     */
    private QueueConsumer<String> createQueueConsumer() {
        return new QueueConsumer<String>() {
            @Override
            public void consumeMessage(String message) throws Exception {
                LOGGER.info("消费的消息内容为：{}", message);
            }

            @Override
            public void stateChanged(CuratorFramework client, ConnectionState newState) {
                LOGGER.info("当前连接的状态改变了，新状态为：{}", newState);
            }
        };
    }
}
```

在这个例子中， 有些元素还没有被消费者消费前就移除了，这样消费者不会收到删除的消息。

### 优先级分布式队列(DistributedPriorityQueue)

优先级队列对队列中的元素按照优先级进行排序。 **Priority越小， 元素越靠前， 越先被消费掉**。 它涉及下面几个类：

- QueueBuilder
- QueueConsumer
- QueueSerializer
- DistributedPriorityQueue

通过builder.buildPriorityQueue(minItemsBeforeRefresh)方法创建。 当优先级队列得到元素增删消息时，它会暂停处理当前的元素队列，然后刷新队列。minItemsBeforeRefresh指定刷新前当前活动的队列的最小数量。 主要设置你的程序可以容忍的不排序的最小值。

放入队列时需要指定优先级：

```java
queue.put(aMessage, priority);
```

示例代码如下：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.queue.DistributedPriorityQueue;
import org.apache.curator.framework.recipes.queue.QueueBuilder;
import org.apache.curator.framework.recipes.queue.QueueConsumer;
import org.apache.curator.framework.recipes.queue.QueueSerializer;
import org.apache.curator.framework.state.ConnectionState;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.curator.utils.CloseableUtils;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.Random;
import java.util.concurrent.TimeUnit;

/**
 * @author ZGY
 * @date 2020/1/8 15:32
 * @description Test19App, 优先级分布式队列—DistributedPriorityQueue
 */
public class Test19App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test19App.class);

    /**
     * 测试方法
     */
    @Test
    public void test() throws Exception {
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 10000, 5000, new ExponentialBackoffRetry(5000, 3));
        client.getCuratorListenable().addListener((client1, event) -> {
            LOGGER.info("监听客户端连接事件，事件名为：{}", event.getType().name());
        });
        client.start();

        // 创建优先级队列对象
        DistributedPriorityQueue<String> queue = QueueBuilder.builder(client, createQueueConsumer(), createQueueSerializer(), "/example/queue").buildPriorityQueue(0);
        queue.start();

        Random random = new Random();
        for (int i = 0; i < 10; i++) {
            int priority = random.nextInt(100);
            LOGGER.info("Test-" + i + " priority: " + priority);
            queue.put("Test" + i, priority);
            TimeUnit.SECONDS.sleep(random.nextInt(2));
        }

        LOGGER.info("程序执行完毕！开始回收资源");

        CloseableUtils.closeQuietly(queue);
        CloseableUtils.closeQuietly(client);
    }

    /**
     * 创建序列化和反序列化对象
     * @return
     */
    private QueueSerializer<String> createQueueSerializer() {
        return new QueueSerializer<String>() {
            @Override
            public byte[] serialize(String item) {
                return item.getBytes();
            }

            @Override
            public String deserialize(byte[] bytes) {
                return new String(bytes);
            }
        };
    }

    /**
     * 创建消费者对象
     * @return
     */
    private QueueConsumer<String> createQueueConsumer() {
        return new QueueConsumer<String>() {
            @Override
            public void consumeMessage(String message) throws Exception {
                LOGGER.info("消费的消息内容为：{}", message);
            }

            @Override
            public void stateChanged(CuratorFramework client, ConnectionState newState) {
                LOGGER.info("当前连接的状态改变了，新状态为：{}", newState);
            }
        };
    }
}
```

有时候你可能会有错觉，优先级设置并没有起效。那是因为优先级是对于队列积压的元素而言，如果消费速度过快有可能出现在后一个元素入队操作之前前一个元素已经被消费，这种情况下DistributedPriorityQueue会退化为DistributedQueue。

### 分布式延迟队列(DistributedDelayQueue)

JDK中也有DelayQueue，不知道你是否熟悉。 DistributedDelayQueue也提供了类似的功能， 元素有个delay值， 消费者隔一段时间才能收到元素。 涉及到下面四个类。

- QueueBuilder
- QueueConsumer
- QueueSerializer
- DistributedDelayQueue

通过下面的语句创建：

```java
QueueBuilder<MessageType>    builder = QueueBuilder.builder(client, consumer, serializer, path);
... more builder method calls as needed ...
DistributedDelayQueue<MessageType> queue = builder.buildDelayQueue();
```

放入元素时可以指定`delayUntilEpoch`：

```java
queue.put(aMessage, delayUntilEpoch);
```

注意`delayUntilEpoch`不是离现在的一个时间间隔， 比如20毫秒，而是未来的一个时间戳，如 System.currentTimeMillis() + 10秒。 如果delayUntilEpoch的时间已经过去，消息会立刻被消费者接收。

示例代码如下：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.queue.DistributedDelayQueue;
import org.apache.curator.framework.recipes.queue.QueueBuilder;
import org.apache.curator.framework.recipes.queue.QueueConsumer;
import org.apache.curator.framework.recipes.queue.QueueSerializer;
import org.apache.curator.framework.state.ConnectionState;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.curator.utils.CloseableUtils;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.concurrent.TimeUnit;

/**
 * @author ZGY
 * @date 2020/1/8 15:53
 * @description Test20App, 分布式延迟队列—DistributedDelayQueue
 */
public class Test20App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test20App.class);

    @Test
    public void test() throws Exception {
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 10000, 5000, new ExponentialBackoffRetry(5000, 3));
        client.getCuratorListenable().addListener((client1, event) -> {
            LOGGER.info("监听客户端连接事件，事件名为：{}", event.getType().name());
        });
        client.start();

        // 创建延时队列对象
        DistributedDelayQueue<String> queue = QueueBuilder.builder(client, createQueueConsumer(), createQueueSerializer(), "/example/queue").buildDelayQueue();
        queue.start();

        for (int i = 0; i < 10; i++) {
            queue.put("Test-" + i, System.currentTimeMillis() + 1000);
        }

        LOGGER.info("所有的数据都已经放入了延时队列中！");

        // 等待消费者消费完队列数据再释放资源
        TimeUnit.SECONDS.sleep(10);

        // 释放资源
        CloseableUtils.closeQuietly(queue);
        CloseableUtils.closeQuietly(client);
    }

    /**
     * 创建序列化和反序列化对象
     * @return
     */
    private QueueSerializer<String> createQueueSerializer() {
        return new QueueSerializer<String>() {
            @Override
            public byte[] serialize(String item) {
                return item.getBytes();
            }

            @Override
            public String deserialize(byte[] bytes) {
                return new String(bytes);
            }
        };
    }

    /**
     * 创建消费者对象
     * @return
     */
    private QueueConsumer<String> createQueueConsumer() {
        return new QueueConsumer<String>() {
            @Override
            public void consumeMessage(String message) throws Exception {
                LOGGER.info("消费的消息内容为：{}", message);
            }

            @Override
            public void stateChanged(CuratorFramework client, ConnectionState newState) {
                LOGGER.info("当前连接的状态改变了，新状态为：{}", newState);
            }
        };
    }
}
```

### 基于JDK的分布式队列(SimpleDistributedQueue)

前面虽然实现了各种队列，但是你注意到没有，这些队列并没有实现类似JDK一样的接口。 `SimpleDistributedQueue`提供了和JDK基本一致的接口(但是没有实现Queue接口)。 创建很简单：

```java
public SimpleDistributedQueue(CuratorFramework client,String path)
```

增加元素：

```java
public boolean offer(byte[] data) throws Exception
```

删除元素：

```java
public byte[] take() throws Exception
```

另外还提供了其它方法：

```java
public byte[] peek() throws Exception
public byte[] poll(long timeout, TimeUnit unit) throws Exception
public byte[] poll() throws Exception
public byte[] remove() throws Exception
public byte[] element() throws Exception
```

没有`add`方法， 多了`take`方法。`take`方法在成功返回之前会被阻塞。 而`poll`方法在队列为空时直接返回null。

示例代码如下：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.queue.SimpleDistributedQueue;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.curator.utils.CloseableUtils;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.TimeUnit;

/**
 * @author ZGY
 * @date 2020/1/8 16:22
 * @description Test21App，SimpleDistributedQueue
 */
public class Test21App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test21App.class);

    @Test
    public void test() throws InterruptedException {
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 20000, 5000, new ExponentialBackoffRetry(5000, 3));
        client.getCuratorListenable().addListener((client1, event) -> {
            LOGGER.info("监听客户端连接事件，事件名为：{}", event.getType().name());
        });
        client.start();

        // 创建队列对象
        SimpleDistributedQueue queue = new SimpleDistributedQueue(client, "/example/queue");

        // 创建生产者和消费者对象
        Producer producer = new Producer(queue);
        Consumer consumer = new Consumer(queue);

        // 启动生产者和消费者线程
        new Thread(producer).start();
        new Thread(consumer).start();

        // 等待队列中的数据处理完毕
        TimeUnit.SECONDS.sleep(10);

        // 释放资源
        CloseableUtils.closeQuietly(client);

        LOGGER.info("程序执行完毕！");
    }

    /**
     * 消费者
     */
    private class Consumer implements Runnable {

        private SimpleDistributedQueue queue;

        public Consumer(SimpleDistributedQueue queue) {
            this.queue = queue;
        }

        @Override
        public void run() {
            try {
                while (true) {
                    byte[] bytes = this.queue.take();
                    if (bytes == null) {
                        break;
                    }
                    LOGGER.info("消费一条消息成功：{}", new String(bytes));
                }
            } catch (Exception e) {
                LOGGER.error("程序出现异常!", e);
                return;
            }
        }
    }

    /**
     * 生产者
     */
    private class Producer implements Runnable {

        private SimpleDistributedQueue queue;

        public Producer(SimpleDistributedQueue queue) {
            this.queue = queue;
        }

        @Override
        public void run() {
            for (int i = 0; i < 5; i++) {
                try {
                    boolean flag = this.queue.offer(("test-" + i).getBytes());
                    if (flag) {
                        LOGGER.info("发送消息成功：{}", "test-" + i);
                    } else {
                        LOGGER.info("发送消息失败：{}", "test-" + i);
                    }
                } catch (Exception e) {
                    LOGGER.error("程序发生异常！", e);
                    continue;
                }
            }
        }
    }
}
```

## 分布式栅栏(屏障)——Barrier

分布式Barrier是这样一个类： 它会阻塞所有节点上的等待进程，直到某一个被满足， 然后所有的节点继续进行。

比如赛马比赛中， 等赛马陆续来到起跑线前。 一声令下，所有的赛马都飞奔而出。

### DistributedBarrier

`DistributedBarrier`类实现了栅栏的功能。 它的构造函数如下：

```java
public DistributedBarrier(CuratorFramework client, String barrierPath)
```

首先你需要设置栅栏，它将阻塞在它上面等待的线程:

```java
setBarrier();
```

然后需要阻塞的线程调用方法告诉栅栏等待放行:

```java
public void waitOnBarrier()
```

移除栅栏，所有等待的线程将继续执行：

```java
removeBarrier();
```

**异常处理** DistributedBarrier 会监控连接状态，当连接断掉时`waitOnBarrier()`方法会抛出异常。

示例代码如下：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.barriers.DistributedBarrier;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Random;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

/**
 * @author ZGY
 * @date 2020/1/8 17:30
 * @description Test22App, DistributedBarrier
 */
public class Test22App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test22App.class);

    @Test
    public void test() throws Exception {
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 20000, 5000, new ExponentialBackoffRetry(5000, 3));
        client.start();

        // 创建线程池
        ExecutorService executorService = Executors.newFixedThreadPool(5);

        // 创建栅栏对象
        final DistributedBarrier barrier = new DistributedBarrier(client, "/examples/barrier");
        // 设置栅栏，在该栅栏上的所有线程都将阻塞
        barrier.setBarrier();

        Random random = new Random();

        for (int i = 0; i < 5; i++) {
            final int index = i;
            Callable<Void> task = () -> {
                TimeUnit.MILLISECONDS.sleep(random.nextInt(3));
                LOGGER.info("客户端 #" + index + "准备数据");

                // 告诉栅栏对象，数据准备完毕
                try {
                    barrier.waitOnBarrier();
                } catch (Exception e) {
                    LOGGER.error("程序出现异常！", e);
                }
                // 栅栏对象放行后，执行
                LOGGER.info("开始处理数据！");
                return null;
            };
            executorService.submit(task);
        }

        TimeUnit.SECONDS.sleep(5);

        LOGGER.info("当所有线程数据准备完毕后，开始放行！");

        // 放行栅栏
        barrier.removeBarrier();

        // 关闭线程池
        executorService.shutdown();

        // 和 executorService.shutdown() 组合使用，监控线程池是否已经关闭，如果线程池内之前提交的任务还没有完成，会一直监控到任务处理完成后。
        executorService.awaitTermination(10, TimeUnit.MINUTES);
    }
}
```

这个例子创建了`controlBarrier`来设置栅栏和移除栅栏。 我们创建了5个线程，在此Barrier上等待。 最后移除栅栏后所有的线程才继续执行。

### 双栅栏(DistributedDoubleBarrier)

双栅栏允许客户端在计算的开始和结束时同步。当足够的进程加入到双栅栏时，进程开始计算， 当计算完成时，离开栅栏。 双栅栏类是`DistributedDoubleBarrier`。 构造函数为:

```java
public DistributedDoubleBarrier(CuratorFramework client,
                                String barrierPath,
                                int memberQty)
```

`memberQty`是成员数量，当`enter()`方法被调用时，成员被阻塞，直到所有的成员都调用了`enter()`。 当`leave()`方法被调用时，它也阻塞调用线程，直到所有的成员都调用了`leave()`。 就像百米赛跑比赛， 发令枪响， 所有的运动员开始跑，等所有的运动员跑过终点线，比赛才结束。

DistributedDoubleBarrier会监控连接状态，当连接断掉时`enter()`和`leave()`方法会抛出异常。

示例代码如下：

```java
package com.zgy.test;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.barriers.DistributedDoubleBarrier;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Random;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

/**
 * @author ZGY
 * @date 2020/1/8 18:07
 * @description Test23App, 双栅栏—DistributedDoubleBarrier
 */
public class Test23App {

    private static final Logger LOGGER = LoggerFactory.getLogger(Test23App.class);

    @Test
    public void test() throws InterruptedException {
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 20000, 5000, new ExponentialBackoffRetry(5000, 3));
        client.start();

        // 创建线程池
        ExecutorService executorService = Executors.newFixedThreadPool(5);


        Random random = new Random();

        for (int i = 0; i < 5; i++) {
            // 创建栅栏对象
            final DistributedDoubleBarrier doubleBarrier = new DistributedDoubleBarrier(client, "/examples/barrier", 5);
            final int index = i;
            Callable<Void> task = () -> {
                TimeUnit.MILLISECONDS.sleep(random.nextInt(3));
                LOGGER.info("客户端 #" + index + "进入栅栏");
                doubleBarrier.enter();

                // 当所有线程都进入栅栏后，执行下面的代码
                LOGGER.info("执行代码");
                TimeUnit.SECONDS.sleep(random.nextInt(3));

                doubleBarrier.leave();
                LOGGER.info("客户端 #" + index + "离开栅栏");

                return null;
            };
            executorService.submit(task);
        }

        // 关闭线程池
        executorService.shutdown();

        // 和 executorService.shutdown() 组合使用，监控线程池是否已经关闭，如果线程池内之前提交的任务还没有完成，会一直监控到任务处理完成后。
        executorService.awaitTermination(10, TimeUnit.MINUTES);
    }
}
```

# 结束

源代码：https://github.com/ZGYSYY/spring-boot-seckill/blob/master/src/test/java/com/zgy/test

> 本文参考自：http://throwable.coding.me/2018/12/16/zookeeper-curator-usage/