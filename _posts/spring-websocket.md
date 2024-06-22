---
title: 'SpringBoot WebSocket技术'
date: 2020-10-15
lastmod: 2020-10-15
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["spring"]
categories: ["技术"]
lightgallery: true
---

最近看了Spring in Action，了解了一下WebSocket和Stomp协议相关技术，并搭建了一个项目。网上的例子不完整或者描述不清，所以自己记录一下以作备忘。

## 一.配置

Spring Boot项目搭建完成后，基于Spring Boot一切皆配置的概念，添加WebSocket支持十分简单。

首先是maven依赖：
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```
如果是使用的Spring Mvc的话，可能需要添加另外的2个依赖。

然后是添加配置类：WebSocketConfig
```java
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.AbstractWebSocketMessageBrokerConfigurer;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig extends AbstractWebSocketMessageBrokerConfigurer {
    @Override
    public void registerStompEndpoints(StompEndpointRegistry stompEndpointRegistry) {
        stompEndpointRegistry.addEndpoint("/endpointSang").withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/happy");
    }
}
```
其中的两个路径：
1.addEndpoint添加的第一个路径，是监听WebSocket连接的Stomp代理的端点，页面请求WebSocket连接时，连接到注册的该端点上Stomp代理，之后的消息会交给Stomp代理处理。

2.该配置启用了一个简单的消息代理，用来处理前缀为/happy的消息，也就是说，只有路径为/happy请求时，消息才会由消息代理处理


### 二.后端配置控制器Controller

Controller十分相似，部分注解略有不同:
```java
import com.example.demo.bean.TestMessage;
import com.example.demo.bean.TestResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.Date;

@Controller
public class WsController {
    @Autowired
    private SimpMessagingTemplate messagingTemplate;
    @MessageMapping("/welcome")//接收路径
    @SendTo("/happy/getNewResponse")//消息返回到的路径
    public TestResponse say(TestMessage message) {
        System.out.println(message.getName());
        say1();//调用另外的方式返回（由服务器主动发起的返回）
        return new TestResponse("welcome," + message.getName() + " !");//这次同步通信的返回
    }

    public void say1() {
        Date date =new Date(System.currentTimeMillis());
        DateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        System.out.println(date);
        messagingTemplate.convertAndSend("/happy/getHappyResponse", df.format(new Date()));//设置路径以及内容，返回当前服务器时间
    }
}
```

1. TestMessage与TestResponse为普通JavaBean，消息转换机制等与普通Controller基本一致

2. SimpMessagingTemplate该类提供为主动向页面正在监听WebSocket的程序发送消息的功能

3. @MessageMapping注解，与@RequestMapping注解类似，配置后台接收消息的路径以及处理函数
   
4. @SendTo注解，一般与@MessageMapping注解一起使用，该注解配置的控制器，返回的数据将发送到监听该配置路径的监听函数


### 三.前端页面
```html
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8"/>
    <title>广播式WebSocket</title>
    <script src="../static/jquery.min.js"></script>
    <script src="../static/stomp.js"></script>
    <script src="../static/sockjs.js"></script>
</head>
<body onload="disconnect()">
<div>
    <div>
        <button id="connect" onclick="connect();">连接</button>
        <button id="disconnect" disabled="disabled" onclick="disconnect();">断开连接</button>
    </div>

    <div id="conversationDiv">
        <label>输入你的名字</label><input type="text" id="name"/>
        <button id="sendName" onclick="sendName();">发送</button>
        <p id="response"></p>
    </div>
</div>
<script type="text/javascript">
    var stompClient = null;
    function setConnected(connected) {
        document.getElementById("connect").disabled = connected;
        document.getElementById("disconnect").disabled = !connected;
        document.getElementById("conversationDiv").style.visibility = connected ? 'visible' : 'hidden';
        $("#response").html();
    }
    function connect() {
        var socket = new SockJS('/endpointSang');//通过先前配置的端点建立连接
        stompClient = Stomp.over(socket);
        stompClient.connect({}, function (frame) {
            setConnected(true);
            console.log('Connected:' + frame);
            //开启监听，监听服务器推送到路径/happy/getNewResponse
            stompClient.subscribe('/happy/getNewResponse', function (response) {
                alert(JSON.parse(response.body).responseMessage);
            });
            //开启监听，监听服务器推送到路径/happy/getHappyResponse
            stompClient.subscribe('/happy/getHappyResponse', function (response) {
                console.log(response.body);
            })
        });
    }
    //关闭连接
    function disconnect() {
        if (stompClient != null) {
            stompClient.disconnect();
        }
        setConnected(false);
        console.log('Disconnected');
    }
    //主动发送信息
    function sendName() {
        var name = $('#name').val();
        console.log('name:' + name);
        //发送信息到后台监听/welcome路径的controller
        stompClient.send("/welcome", {}, JSON.stringify({'name': name}));
    }
    function showResponse(message) {
        $("#response").html(message);
    }
</script>
</body>
</html>
```
运行结果控制台日志：

注：
广播模式，只要所有程序监听同一个后台广播路径就可以了
点对点通信模式，可以在Js端使用随机数或者根据TokenId开启监听路径，后台根据用户的TokenId派发到不同端点就可以了
