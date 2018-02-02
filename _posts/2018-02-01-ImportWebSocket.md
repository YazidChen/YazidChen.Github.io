---
layout: post
title:  "WebSocket应用"
categories: WebSocket
description: WebSocket应用实践。
keywords: WebSocket, SockJS, STOMP
---

# 一、为何引入WebSocket #

## 1.1 遇到问题 ##
当前有一客服管理系统，有以下业务需求：

#### 1、呼出业务 ####

![](https://i.imgur.com/KQgXefy.png)

客服管理系统向第三方云呼系统发出呼叫请求，云呼系统异步返回呼出回调，后台服务接收到回调后，需要通知WEB端提示用户呼出的结果，此时有后台服务主动通知WEB的需求。

#### 2、呼入业务 ####

![](https://i.imgur.com/kDPC13M.png)

当有电话呼入时，云呼系统发起呼入通知给客服管理系统，客服管理系统后台服务接收到呼入通知后，需要及时让WEB发起界面提醒，此时也有后台服务主动通知WEB的需求。

## 1.2 解决方案 ##

1. WEB通过HTTP轮询后台服务（不满足及时性，消耗资源大）;
2. HTTP长轮询(基本满足及时性，消耗资源较大）
3. WebSocket(满足及时性，消耗资源小，提供双向通信)。

初步判断，WebSocket可以作为解决方案。下面开始深入研究WebSocket。

# 二、WebSocket是什么 #

**WebSocket协议**由HTML5定义，能更好的**节省**服务器资源和带宽，并且能够更实时地进行**双向**通讯。它是一种持久化的协议。
我们拿WebSocket与HTTP进行对比。

![](https://i.imgur.com/LLATtBY.png)


首先是HTTP轮询，浏览器通过ajax每隔几秒就向服务器发起一次请求，询问服务器是否有新的消息。每次发起的请求，即Request，它都包含header头部，多次轮询相当于多次发送了header头部信息，这便会造成带宽资源的浪费。每次返回的Response亦是如此。

其次是HTTP长轮询，与前者相比，长轮询采用的是阻塞模式，即客户端发起请求，会一直等待服务端有消息后才返回，而后再次发起请求。如果无消息，就会一直等，直至超时。

以上两种方式，都是客户端主动向服务端发起请求，等待服务端处理。而服务端不能主动联系客户端。

下面开始WebSocket协议内容：
首先，客户端会向服务端发起一次Handshake握手的请求，服务端收到请求并确认无误后，返回确认通知。这两步操作还是通过HTTP协议进行通信的。确认通知返回成功后，通信协议由HTTP协议升级为WebSocket协议。之后，服务端就能主动推送消息给客户端，当然，客户端也能主动推消息给服务端。真真正正实现了双向通讯。如果要关闭通信链接，客户端可以给服务端发送断开链接的请求，当然，服务端也可以主动断开链接。这样，整个通信链接的生命周期也就结束了。
由此看来，WebSocket只需要一次HTTP请求进行建立链接，之后所有的通信都不需要有复杂的header头部信息，也无需多次鉴权了，从而省下了很多资源。

![](https://i.imgur.com/jjDjhHU.png)

继续深入学习WebSocket，我们发现他同HTTP协议一样，也是属于OSI模型的应用层，也是建立在TCP协议之上的。WebSocket协议的**统一资源标识符**是`ws`，如果加密，则是`wss`,这和HTTP如出一辙，都是由TSL进行加密的。

![](https://i.imgur.com/lhwKS1T.png)

服务器的网址，即为URL：`ws://example.com:80/test`

# 三、SpringMVC如何引入WebSocket #

这里有三种API，分别是`WebSocket API`，`SockJS`以及`STOMP`。`WebSocket API`是发送和接收消息的底层API，`SockJS`在`WebSocket API`之上，而`STOMP`是基于`SockJS`的高级API。

## 3.1 WebSocket API ##

### 3.1.1 服务端配置 ###

1、创建Handler

```java
import org.springframework.stereotype.Service;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.AbstractWebSocketHandler;
import java.io.IOException

/**
 * @author YazidChen
 * @date 2017/12/21 0021 11:10
 **/
@Service//此处注解扫描
public class WsHandler extends AbstractWebSocketHandler {

    @Override
    public void handleTextMessage(WebSocketSession session, TextMessage message) throws IOException {
        System.out.println("into handleText");
        session.sendMessage(new TextMessage("来自服务器的消息！"));
    }
}
```

2、创建握手拦截器：

```java
import com.jz.service.jzcf_ums.api.form.UserSession;
import com.jzcf.ums.sso.utils.ShiroUtils;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.stereotype.Service;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.server.HandshakeInterceptor;

import java.util.Map;

/**
 * @author YazidChen
 * @date 2017/12/21 0021 11:21
 **/
@Service//此处注解扫描
public class WsHandshakeInterceptor implements HandshakeInterceptor {

    @Override
    public boolean beforeHandshake(ServerHttpRequest serverHttpRequest, ServerHttpResponse serverHttpResponse, WebSocketHandler webSocketHandler, Map<String, Object> map) throws Exception {
	//在引入Shiro的情况下，此处可获取用户Session
        UserSession userSession = ShiroUtils.getUserSession();
        map.put("userSession", userSession);
        return true;
    }

    @Override
    public void afterHandshake(ServerHttpRequest serverHttpRequest, ServerHttpResponse serverHttpResponse, WebSocketHandler webSocketHandler, Exception e) {
    }
}

```

3、将WsHandler绑定到特定的URL上

```java
import com.jzcf.csms.core.websocket.handler.WsHandler;
import com.jzcf.csms.core.websocket.interceptor.WsHandshakeInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;

/**
 * @author YazidChen
 * @date 2018/01/31 0031 10:26
 **/
@Configuration
@EnableWebSocket
public class WsConfig implements WebSocketConfigurer {
    @Autowired
    private WsHandler wsHandler;
    @Autowired
    private WsHandshakeInterceptor wsHandshakeInterceptor;

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(wsHandler, "/chat").setAllowedOrigins("https://csms-dev.jzcaifu.com").addInterceptors(wsHandshakeInterceptor);
    }
}

```

- `addHandler()`将`WsHandler`绑定在了`/chat`路径上；
- `setAllowedOrigins()`允许来源为`https://csms-dev.jzcaifu.com`的请求，设置为`*`允许所有来源；
- `addInterceptors()`添加拦截器。


### 3.3.2 客户端配置 ###

客户端发起WebSocket链接请求：

```js
let ws = new WebSocket("wss://csms-dev.jzcaifu.com/chat");//发起连接
            ws.onopen = function () {
                console.log("WebSocket open now!")
                ws.send("message test!");//客户端向服务器发送消息
            };

            ws.onmessage = function (e) {
                console.log("WebSocket message now!");
                console.log(e);
            };

            ws.onclose = function () {
                console.log("WebSocket close now!");
            };
```

### 3.3.3 运行结果 ###

![](https://i.imgur.com/mTgAbTb.png)

![](https://i.imgur.com/KwNihlQ.png)

- `Connection`为Upgrade，表示客户端希望连接升级；
- `Upgrade`为Websocket，表示希望升级到Websocket协议；
- `Sec-WebSocket-Key`是随机的字符串，服务器端会用这些数据来构造出一个SHA-1的信息摘要。把`Sec-WebSocket-Key`加上一个特殊字符串，然后计算`SHA-1`摘要，之后进行`BASE-64`编码，将结果作为`Sec-WebSocket-Accept`头的值，返回给客户端。如此操作，可以尽量避免普通`HTTP`请求被误认为`Websocket`协议；
- `Sec-WebSocket-Version`表示支持的Websocket版本；
- Websocket通过`HTTP/1.1`协议的`101`状态码进行握手，此时握手成功。


## 3.2 SockJS ##

在`WebSocket API`的基础上，稍加修改，就变为`SockJS`形式。

### 3.2.1 服务端配置 ###

```java
import com.jzcf.csms.core.websocket.handler.WsHandler;
import com.jzcf.csms.core.websocket.interceptor.WsHandshakeInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;

/**
 * @author YazidChen
 * @date 2018/01/31 0031 10:26
 **/
@Configuration
@EnableWebSocket
public class WsConfig implements WebSocketConfigurer {
    @Autowired
    private WsHandler wsHandler;
    @Autowired
    private WsHandshakeInterceptor wsHandshakeInterceptor;

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(wsHandler, "/chat").setAllowedOrigins("https://csms-dev.jzcaifu.com").addInterceptors(wsHandshakeInterceptor).withSockJS();//此处使用SockJS
    }
}
```

方法`withSockJS()`用SockJS改造了原始的API

### 3.2.2 客户端配置 ###

```js
<!--引入sockjs-->
<script type="text/javascript" src="/statics/libs/sockjs.min.js"></script>
<!--此处为https协议-->
            let sock = new SockJS("https://csms-dev.jzcaifu.com/chat");
            sock.onopen = function () {
                console.log("WebSocket open now!")
                sock.send("message test!");//客户端向服务器发送消息
            };

            sock.onmessage = function (e) {
                console.log("WebSocket message now!");
                console.log(e);
            };

            sock.onclose = function () {
                console.log("WebSocket close now!");
            };
```

**SockJS**将`http`或`https`协议处理成为`ws`或`wss`协议。

### 3.2.3 运行结果 ###

![](https://i.imgur.com/d91tSia.png)

![](https://i.imgur.com/9bOXUOI.png)

## 3.3 STOMP ##

此处为XML配置形式示例。

### 3.3.1 服务端配置 ###

1、创建拦截器：


```java
import com.jz.service.jzcf_ums.api.form.UserSession;
import com.jzcf.ums.sso.utils.ShiroUtils;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.server.HandshakeInterceptor;

import java.util.Map;

/**
 * @author YazidChen
 * @date 2017/12/21 0021 11:21
 **/
public class WsHandshakeInterceptor implements HandshakeInterceptor {

    @Override
    public boolean beforeHandshake(ServerHttpRequest serverHttpRequest, ServerHttpResponse serverHttpResponse, WebSocketHandler webSocketHandler, Map<String, Object> map) throws Exception {
        UserSession userSession = ShiroUtils.getUserSession();
        map.put("userSession", userSession);
        return true;
    }

    @Override
    public void afterHandshake(ServerHttpRequest serverHttpRequest, ServerHttpResponse serverHttpResponse, WebSocketHandler webSocketHandler, Exception e) {
    }
}

```

2、XML配置：


```xml
    <!--利用fastjson改变消息体传输方式-->
    <bean id="fastjsonSockJsMessageCodec" class="com.alibaba.fastjson.support.spring.FastjsonSockJsMessageCodec">
    </bean>

    <!--开启websocket-->
    <websocket:message-broker application-destination-prefix="/ws" user-destination-prefix="/user">
        <websocket:stomp-endpoint path="/portfolio" allowed-origins="*">
            <websocket:handshake-interceptors>
                <bean id="wsHandshakeInterceptor"
                      class="com.jzcf.csms.core.websocket.interceptor.WsHandshakeInterceptor"/>
            </websocket:handshake-interceptors>
            <websocket:sockjs message-codec="fastjsonSockJsMessageCodec"/>
        </websocket:stomp-endpoint>
        <websocket:simple-broker prefix="/topic, /queue"/>
    </websocket:message-broker>
```

- `stomp-endpoint`中，`path`为`/portfolio`是客户端需要连接到WebSocket握手的端点的HTTP URL；
- `stomp-endpoint`中，`allowed-origins`为`*`允许所有来源；
- `message-broker`中，`application-destination-prefix`的`/ws`将STOMP消息路由到类中的@MessageMapping注解的方法；
- `message-broker`中，`user-destination-prefix="/user"`表示开启用户目的地，服务器可以发送针对特定用户的消息，并且Spring的STOMP可以识别"/user/"以此为前缀的目标；
- `simple-broker`中，`prefix="/topic, /queue"`表示代理目的地前缀，即订阅时的前缀。

3、对于用户目的地，编写一个用户管理中心：


```java
import com.jzcf.csms.core.callcenter.form.WsMessage;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Service;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * @author YazidChen
 * @date 2018/01/23 0023 12:45
 **/
@Service
public class WsUsers {
    public static final Map<Long, List<String>> users;

    @Autowired
    private SimpMessagingTemplate template;

    static {
        users = new HashMap<>();
    }

    /**
     * 发送消息给订阅者
     *
     * @param subscribePath 订阅路径
     * @param userId        客服ID
     * @param wsMessage     消息体
     * @author YazidChen
     * @createDate 2018/01/23 0023 14:59
     */
    public void sendToUser(String subscribePath, Long userId, WsMessage wsMessage) {
        List<String> list = users.get(userId);
        if (list == null || list.isEmpty()) {
            return;
        }
        System.out.println(template);
        list.forEach(username -> template.convertAndSendToUser(username, subscribePath, wsMessage));
    }
}

```

4、创建监听器：

```java
import com.jz.service.jzcf_ums.api.form.UserSession;
import com.jzcf.csms.core.user.service.UserService;
import com.jzcf.csms.core.user.util.StatusEnum;
import com.jzcf.csms.core.websocket.service.WsUsers;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationListener;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageHeaders;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.messaging.SessionConnectedEvent;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

/**
 * @author YazidChen
 * @date 2018/01/23 0023 11:41
 **/
@Component//注解扫描
public class WsConnectedListen implements ApplicationListener<SessionConnectedEvent> {

    @Autowired
    private UserService userService;

    @Override
    public void onApplicationEvent(SessionConnectedEvent sessionConnectedEvent) {
        MessageHeaders messageHeader = sessionConnectedEvent.getMessage().getHeaders();
        Map simpSessionAttributes = (Map) ((Message) messageHeader.get("simpConnectMessage")).getHeaders().get("simpSessionAttributes");

        UserSession userSession = (UserSession) simpSessionAttributes.get("userSession");
        String simpUser = messageHeader.get("simpUser").toString();
        //内存不存在当前用户ID
        if (WsUsers.users.get(userSession.getUserId()) == null) {
            List<String> list = new ArrayList<>();
            list.add(simpUser);
            WsUsers.users.put(userSession.getUserId(), list);
        } else {//内存存在当前用户ID
            List<String> list = WsUsers.users.get(userSession.getUserId());
            //校验是否不包含当前session
            if (!list.contains(simpUser)) {
                list.add(simpUser);
            }
        }
        //用户上线
        userService.onOffLine(userSession, StatusEnum.ON_OFF_LINE.ON_LINE.type);
    }
}

```

`SessionConnectedEvent`事件在STOMP连接完全建立后发布，此处用于将用户一至多个浏览器窗口的sessionId存入内存中。（考虑到客服系统用户量小，选择存入内存。）


```java
import com.jz.service.jzcf_ums.api.form.UserSession;
import com.jzcf.csms.core.user.service.UserService;
import com.jzcf.csms.core.user.util.StatusEnum;
import com.jzcf.csms.core.websocket.service.WsUsers;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationListener;
import org.springframework.messaging.MessageHeaders;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.messaging.SessionDisconnectEvent;
import java.util.List;
import java.util.Map;

/**
 * @author YazidChen
 * @date 2018/01/23 0023 14:15
 **/
@Component//注解扫描
public class WsDisconnectListen implements ApplicationListener<SessionDisconnectEvent> {

    @Autowired
    private UserService userService;

    @Override
    public void onApplicationEvent(SessionDisconnectEvent sessionDisconnectEvent) {
        MessageHeaders messageHeader = sessionDisconnectEvent.getMessage().getHeaders();
        UserSession userSession = (UserSession) ((Map) messageHeader.get("simpSessionAttributes")).get("userSession");
        String simpUser = messageHeader.get("simpUser").toString();
        //如果内存中存在当前用户ID
        if (WsUsers.users.get(userSession.getUserId()) != null) {
            List list = WsUsers.users.get(userSession.getUserId());
            if (list.contains(simpUser)) {
                //删除用户当前断开的session
                list.remove(simpUser);
            }
            if (list.size() == 0) {
                WsUsers.users.remove(userSession.getUserId());
            }
        }
        //用户下线
        userService.onOffLine(userSession, StatusEnum.ON_OFF_LINE.OFF_LINE.type);
    }
}

```

`SessionDisconnectEvent`事件再STOMP连接完全关闭时发布，此处将用户sessionId移出内存。

`web.xml`中配置监听器包路径：

```xml
    <listener>
        <listener-class>...websocket.listen.WsConnectedListen</listener-class>
    </listener>

    <listener>
        <listener-class>...listen.WsDisconnectListen</listener-class>
    </listener>
```


5、编写测试Controller:


```java
import com.jz.service.jzcf_ums.api.form.UserSession;
import com.jzcf.csms.common.utils.R;
import com.jzcf.csms.core.callcenter.form.WsMessage;
import com.jzcf.csms.core.websocket.service.WsUsers;
import com.jzcf.csms.shiro.controller.AbstractController;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.messaging.simp.annotation.SendToUser;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.ResponseBody;
import java.util.HashMap;
import java.util.Map;

/**
 * @author YazidChen
 * @date 2017/12/22 0022 14:28
 **/
@Controller
public class WsController extends AbstractController {
    @Autowired
    private SimpMessagingTemplate template;
    @Autowired
    private WsUsers wsUsers;

    @MessageMapping("trade")
    @SendToUser("/queue/sub")
    public R wsTest(SimpMessageHeaderAccessor accessor, @RequestParam Map<String, Object> params) {
        System.out.println("接收到客户端消息！");
        UserSession userSession = (UserSession) accessor.getSessionAttributes().get("userSession");
        return R.ok().put("mess", "服务端返回消息！");
    }

    /**
     * 发送用户消息测试接口
     *
     * @param userId        用户ID
     * @param subscribePath 订阅地址
     * @return
     * @author YazidChen
     * @createDate 2018/01/31 0031 16:05
     */
    @GetMapping("sendMsToUser")
    @ResponseBody
    public R sendMsToUser(long userId, String subscribePath) {
        Map<String, Object> map = new HashMap<>();
        map.put("data", "The Data!");
        map.put("message", "The message!");

        WsMessage wsMessage = new WsMessage();
        wsMessage.setBusinessType("TEST");
        wsMessage.setData(map);

        wsUsers.sendToUser(subscribePath, userId, wsMessage);
        return R.ok();
    }

    /**
     * 发送全局消息测试接口
     *
     * @param subscribePath 订阅地址
     * @return
     * @author YazidChen
     * @createDate 2018/01/31 0031 16:05
     */
    @GetMapping("sendMsAll")
    @ResponseBody
    public R sendMsAll(String subscribePath) {
        Map<String, Object> map = new HashMap<>();
        map.put("data", "The All Data!");
        map.put("message", "The All message!");

        WsMessage wsMessage = new WsMessage();
        wsMessage.setBusinessType("TEST All");
        wsMessage.setData(map);

        template.convertAndSend(subscribePath, wsMessage);
        return R.ok();
    }
}

```

### 3.3.2 客户端配置 ###

需要引入stomp.js:


```js
<script type="text/javascript" src="/statics/libs/stomp.min.js"></script>

let socket = new SockJS('https://csms-dev.jzcaifu.com/portfolio');
            let stompClient = Stomp.over(socket);
            /*            stompClient.debug = function (str) {
                            //不输出日志到控制台
                        };*/
            stompClient.connect({}, function (frame) {
                let messageModel = {};
                messageModel.type = 1;
                messageModel.content = "jhbjhjashuj";
                //向服务端发送消息
                stompClient.send("/ws/trade", {}, JSON.stringify(messageModel));
                //用户订阅
                stompClient.subscribe('/user/queue/sub', function (message) {
                    let wsMessage = JSON.parse(message.body);
                    let data = wsMessage.data;
                    console.log(data);
                });
                //全局订阅
                stompClient.subscribe('/topic/all', function (message) {
                    let wsMessage = JSON.parse(message.body);
                    let data = wsMessage.data;
                    console.log(data);
                });
            });
```

### 3.3.3 运行结果 ###

1、连接结果：

![](https://i.imgur.com/rJCqnx9.png)

2、发送全局消息：

![](https://i.imgur.com/SMssDZ0.png)

3、用户目的地消息发送：

![](https://i.imgur.com/2hxYU27.png)


# 参考 #


本文参考以下文章，在此对原作者表示感谢！

[Spring.io WebSocket](https://docs.spring.io/spring/docs/5.0.3.RELEASE/spring-framework-reference/web.html#websocket)

[WebSocket 教程](http://www.ruanyifeng.com/blog/2017/05/websocket.html)

[WebSocket维基百科](https://zh.wikipedia.org/wiki/WebSocket)

[WebSocket 是什么原理？为什么可以实现持久连接？](https://www.zhihu.com/question/20215561)