---
title: Spring Boot 整合使用WebSocket
date: 2020-01-15 17:35:58
tags:
- SpringBoot
- WebSocket
categories:
- 后端
- Java
---

使用 websocket 有两种方式：

1. 是使用 sockjs。

2. 是使用 h5 的标准。

使用 Html5 标准自然更方便简单，所以记录的是配合 h5 的使用方法。

# 1、pom

核心是 `@ServerEndpoint` 这个注解。这个注解是 Javaee 标准里的注解，tomcat7 以上已经对其进行了实现，如果是用传统方法使用 tomcat 发布项目，只要在 pom 文件中引入 javaee 标准即可使用。

```xml
<dependency>
      <groupId>javax</groupId>
      <artifactId>javaee-api</artifactId>
      <version>7.0</version>
      <scope>provided</scope>
</dependency>
```

但使用 springboot 的内置 tomcat 时，就不需要引入javaee-api了，spring-boot 已经包含了。使用 springboot的websocket 功能首先引入 springboot 组件。

```xml
<dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-websocket</artifactId>
     <version>1.3.5.RELEASE</version>
</dependency>
```

顺便说一句，springboot 的高级组件会自动引用基础的组件，像 spring-boot-starter-websocket 就引入了 spring-boot-starter-web 和 spring-boot-starter，所以不要重复引入。

<!-- more -->

# 2、使用@ServerEndpoint创立websocket endpoint

首先要注入 ServerEndpointExporter ，这个 bean 会自动注册使用了 `@ServerEndpoint` 注解声明的 Websocket endpoint 。要注意，如果使用独立的servlet容器，而不是直接使用springboot的内置容器，就不要注入  ServerEndpointExporter ，因为它将由容器自己提供和管理。

```java
@Configuration
public class WebSocketConfig {
    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
}
```

接下来就是写 websocket 的具体实现类，很简单，直接上代码：

```java
@ServerEndpoint(value = "/websocket")
@Component
public class MyWebSocket {
    //静态变量，用来记录当前在线连接数。应该把它设计成线程安全的。
    private static int onlineCount = 0;

    //concurrent包的线程安全Set，用来存放每个客户端对应的MyWebSocket对象。
    private static CopyOnWriteArraySet<MyWebSocket> webSocketSet = new CopyOnWriteArraySet<MyWebSocket>();

    //与某个客户端的连接会话，需要通过它来给客户端发送数据
    private Session session;

    /**
     * 连接建立成功调用的方法
     */
    @OnOpen
    public void onOpen(Session session) {
        this.session = session;
        webSocketSet.add(this);     //加入set中
        addOnlineCount();           //在线数加1
        System.out.println("有新连接加入！当前在线人数为" + getOnlineCount());
        try {
            sendMessage(CommonConstant.CURRENT_WANGING_NUMBER.toString());
        } catch (IOException e) {
            System.out.println("IO异常");
        }
    }

    /**
     * 连接关闭调用的方法
     */
    @OnClose
    public void onClose() {
        webSocketSet.remove(this);  //从set中删除
        subOnlineCount();           //在线数减1
        System.out.println("有一连接关闭！当前在线人数为" + getOnlineCount());
    }

    /**
     * 收到客户端消息后调用的方法
     *
     * @param message 客户端发送过来的消息
     */
    @OnMessage
    public void onMessage(String message, Session session) {
        System.out.println("来自客户端的消息:" + message);

        //群发消息
        for (MyWebSocket item : webSocketSet) {
            try {
                item.sendMessage(message);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * 发生错误时调用
     *
     */
    @OnError
    public void onError(Session session, Throwable error) {
        System.out.println("发生错误");
        error.printStackTrace();
    }


    public void sendMessage(String message) throws IOException {
        this.session.getBasicRemote().sendText(message);
        //this.session.getAsyncRemote().sendText(message);
    }


    /**
     * 群发自定义消息
     * 
     */
    public static void sendInfo(String message) throws IOException {
        for (MyWebSocket item : webSocketSet) {
            try {
                item.sendMessage(message);
            } catch (IOException e) {
                continue;
            }
        }
    }

    public static synchronized int getOnlineCount() {
        return onlineCount;
    }

    public static synchronized void addOnlineCount() {
        MyWebSocket.onlineCount++;
    }

    public static synchronized void subOnlineCount() {
        MyWebSocket.onlineCount--;
    }
}
```

使用 springboot 的唯一区别是要 `@Component` 声明下，而使用独立容器是由容器自己管理 websocket 的，但在 springboot 中连容器都是spring管理的。

虽然 `@Component` 默认是单例模式的，但 springboot 还是会为每个 websocket 连接初始化一个 bean，所以可以用一个静态 set 保存起来。

# 3、前端代码　

```html
<!DOCTYPE HTML>
<html>
<head>
    <title>My WebSocket</title>
</head>

<body>
Welcome<br/>
<input id="text" type="text" /><button onclick="send()">Send</button>    <button onclick="closeWebSocket()">Close</button>
<div id="message">
</div>
</body>

<script type="text/javascript">
    var websocket = null;

    //判断当前浏览器是否支持WebSocket
    if('WebSocket' in window){
        websocket = new WebSocket("ws://localhost:8084/websocket");
    }
    else{
        alert('Not support websocket')
    }

    //连接发生错误的回调方法
    websocket.onerror = function(){
        setMessageInnerHTML("error");
    };

    //连接成功建立的回调方法
    websocket.onopen = function(event){
        setMessageInnerHTML("open");
    }

    //接收到消息的回调方法
    websocket.onmessage = function(event){
        setMessageInnerHTML(event.data);
    }

    //连接关闭的回调方法
    websocket.onclose = function(){
        setMessageInnerHTML("close");
    }

    //监听窗口关闭事件，当窗口关闭时，主动去关闭websocket连接，防止连接还没断开就关闭窗口，server端会抛异常。
    window.onbeforeunload = function(){
        websocket.close();
    }

    //将消息显示在网页上
    function setMessageInnerHTML(innerHTML){
        document.getElementById('message').innerHTML += innerHTML + '<br/>';
    }

    //关闭连接
    function closeWebSocket(){
        websocket.close();
    }

    //发送消息
    function send(){
        var message = document.getElementById('text').value;
        websocket.send(message);
    }
</script>
</html>
```

# 4、总结

springboot 已经做了深度的集成和优化，要注意是否添加了不需要的依赖、配置或声明。由于很多讲解组件使用的文章是和 spring 集成的，会有一些配置，在使用 springboot 时，由于 springboot 已经有了自己的配置，再这些配置有可能导致各种各样的异常。