---
layout: post
title: SpringBoot集成Websocket
categories: SpringBoot
tags: [Websocket, 即时通讯]
---

#### Java实现websocket的两种方式

##### 1.Tomcat实现
1. 添加配置类

   ```java
   @Configuration
   public class WebSocketConfig {
   	/**
   	 * 注入ServerEndpointExporter，
   	 * 这个bean会自动注册使用了@ServerEndpoint注解声明的Websocket endpoint
   	 */
   	@Bean
   	public ServerEndpointExporter serverEndpointExporter() {
   		return new ServerEndpointExporter();
   	}
   }
   
   ```

2. 消息处理类

   ```java
   @Component
   @Slf4j
   @ServerEndpoint("/ws/{controlPanelId}") //此注解相当于设置访问URL
   public class WebSocket {
   
   	private Session session;
   
   	private static CopyOnWriteArraySet<WebSocket> webSockets = new CopyOnWriteArraySet<>();
   	private static Map<String, Session> sessionPool = new HashMap<String, Session>();
   
   	@OnOpen
   	public void onOpen(Session session, @PathParam(value = "controlPanelId") String controlPanelId) {
   		try {
   			this.session = session;
   			webSockets.add(this);
   			sessionPool.put(controlPanelId, session);
   			log.info("【websocket消息】有新的连接，ip:" + WebSocketUtil.getRemoteAddress(session) + ",id:" + controlPanelId + ",总数为:" + webSockets.size());
   		} catch (Exception e) {
   		}
   	}
   
   	@OnClose
   	public void onClose() {
   		try {
   			webSockets.remove(this);
   			log.info("【websocket消息】连接断开，ip为:" + WebSocketUtil.getRemoteAddress(session) + "，总数为:" + webSockets.size());
   		} catch (Exception e) {
   		}
   	}
   
   	@OnMessage
   	public void onMessage(String message) {
   		log.info("【websocket消息】收到客户端消息:" + message);
   		JSONObject obj = new JSONObject();
   		obj.put(WebsocketConst.MSG_CMD, WebsocketConst.CMD_CHECK);//业务类型
   		obj.put(WebsocketConst.MSG_TXT, "心跳响应");//消息内容
   		session.getAsyncRemote().sendText(obj.toJSONString());
   	}
   
   	// 此为广播消息
   	public void sendAllMessage(String message) {
   		log.info("【websocket消息】广播消息:" + message);
   		for (WebSocket webSocket : webSockets) {
   			try {
   				if (webSocket.session.isOpen()) {
   					webSocket.session.getAsyncRemote().sendText(message);
   				}
   			} catch (Exception e) {
   				e.printStackTrace();
   			}
   		}
   	}
   
   	// 此为单点消息
   	public void sendOneMessage(String controlPanelId, String message) {
   		Session session = sessionPool.get(controlPanelId);
   		if (session != null && session.isOpen()) {
   			try {
   				log.info("【websocket消息】 单点消息:" + message);
   				session.getAsyncRemote().sendText(message);
   			} catch (Exception e) {
   				e.printStackTrace();
   			}
   		}
   	}
   
   	// 此为单点消息(多人)
   	public void sendMoreMessage(String[] controlPanelIds, String message) {
   		for (String controlPanelId : controlPanelIds) {
   			Session session = sessionPool.get(controlPanelId);
   			if (session != null && session.isOpen()) {
   				try {
   					log.info("【websocket消息】 单点消息:" + message);
   					session.getAsyncRemote().sendText(message);
   				} catch (Exception e) {
   					e.printStackTrace();
   				}
   			}
   		}
   	}
   }
   
   ```

   

##### 2.使用Spring集成的websocket实现

1. 添加消息处理类

   ```java
   @Component
   @Slf4j
   public class CpWebSocketHandler extends TextWebSocketHandler {
   
   	//静态变量，用来记录当前在线连接数。应该把它设计成线程安全的。
   	private static AtomicInteger onlineNum = new AtomicInteger();
   	//用来存放每个客户端对应的WebSocketServer对象。
   	private static ConcurrentHashMap<String, WebSocketSession> sessionPools = new ConcurrentHashMap<>();
   
   	/**
   	 * 收到消息
   	 *
   	 * @param session
   	 * @param message
   	 * @throws IOException
   	 */
   	@Override
   	public void handleTextMessage(WebSocketSession session, TextMessage message) throws IOException {
   		String controlPanelId = session.getAttributes().get("controlPanelId").toString();
   		log.info("【WebSocket消息】 收到客户端[{}]发来的消息:{}", controlPanelId, message.getPayload());
   		JSONObject heartBeat = new JSONObject();
   		heartBeat.put(WebsocketConst.MSG_CMD, WebsocketConst.CMD_CHECK);//业务类型
   		heartBeat.put(WebsocketConst.MSG_TXT, "心跳响应");//消息内容
   		session.sendMessage(new TextMessage(heartBeat.toJSONString()));
   	}
   
   	/**
   	 * 客户端连接成功
   	 *
   	 * @param session
   	 * @throws Exception
   	 */
   	@Override
   	public void afterConnectionEstablished(WebSocketSession session) throws Exception {
   
   		//System.out.println("获取到拦截器中用户ID : " + session.getAttributes().get("controlPanelId"));
   		String ControlPanelId = session.getAttributes().get("controlPanelId").toString();
   		//TODO: 重复链接没有进行处理
   		sessionPools.put(ControlPanelId, session);
   		addOnlineCount();
   		log.info("【WebSocket消息】: 客户端ID:[{}],IP[{}],连接成功,当前连接数:[{}]", ControlPanelId, session.getRemoteAddress(), onlineNum);
   		session.sendMessage(new TextMessage("连接服务器成功，当前连接数" + onlineNum));
   	}
   
   
   	/**
   	 * 连接关闭
   	 *
   	 * @param session
   	 * @param status
   	 * @throws Exception
   	 */
   	@Override
   	public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
   		String ControlPanelId = session.getAttributes().get("controlPanelId").toString();
   		sessionPools.remove(ControlPanelId);
   		subOnlineCount();
   		log.info("【WebSocket消息】: 客户端ID:[{}],IP[{}],断开连接,当前连接数:[{}]", ControlPanelId, session.getRemoteAddress(), onlineNum);
   	}
   
   	/**
   	 * 添加连接人数
   	 */
   	public static void addOnlineCount() {
   		onlineNum.incrementAndGet();
   	}
   
   	/**
   	 * 移除连接人数
   	 */
   	public static void subOnlineCount() {
   		onlineNum.decrementAndGet();
   	}
   
   	/**
   	 * 给指定用户发送消息
   	 */
   	public Result sendOneMessage(String controlPanelId, String msgContent) {
   		WebSocketSession socketSession = sessionPools.get(controlPanelId);
   		if (socketSession != null) {
   			try {
   				socketSession.sendMessage(new TextMessage(msgContent));
   			} catch (IOException e) {
   				e.printStackTrace();
   			}
   		}
   		return Result.error("发送失败,[" + controlPanelId + "]" + "已离线");
   		// TODO: 2021/2/3  是否需要对消息进行缓存处理
   	}
   
   	/**
   	 * 给所有控制面板发送消息(广播消息)
   	 *
   	 * @param msgContent
   	 * @return
   	 */
   	public Result sendAllMessage(String msgContent) {
   		sessionPools.entrySet().stream().forEach(sessionPool -> {
   
   			WebSocketSession webSocketSession = sessionPool.getValue();
   			try {
   				webSocketSession.sendMessage(new TextMessage(msgContent));
   			} catch (IOException e) {
   				e.printStackTrace();
   			}
   		});
   		return Result.OK();
   
   	}
   
   	/**
   	 * 给多个控制面板发送消息
   	 *
   	 * @return
   	 */
   	public Result sendManyMessage(String[] controlPanelIds, String msgContent) {
   		for (String controlPanelId : controlPanelIds) {
   			WebSocketSession webSocketSession = sessionPools.get(controlPanelId);
   			try {
   				webSocketSession.sendMessage(new TextMessage(msgContent));
   			} catch (IOException e) {
   				e.printStackTrace();
   			}
   		}
   		return Result.OK();
   	}
   }
   
   ```

2. 配置拦截器

   ```java
   @Slf4j
   @Component
   public class CpWebSocketInterceptor extends HttpSessionHandshakeInterceptor {
   
   
   	/**
   	 * 握手前 这里需要对session进行转换 以及校验token的合法性
   	 *
   	 * @return
   	 * @throws Exception
   	 */
   	@Override
   	public boolean beforeHandshake(ServerHttpRequest serverHttpRequest, ServerHttpResponse serverHttpResponse, WebSocketHandler webSocketHandler, Map<String, Object> map) throws Exception {
   		log.info("before handshaking");
   		return super.beforeHandshake(serverHttpRequest, serverHttpResponse, webSocketHandler, map);
   	}
   
   	@Override
   	public void afterHandshake(ServerHttpRequest serverHttpRequest, ServerHttpResponse serverHttpResponse, WebSocketHandler webSocketHandler, Exception e) {
   		log.info("after handshaking");
   	}
   }
   
   ```

3. 添加配置类

   ```java
   @Configuration
   @EnableWebSocket
   public class CpWebSocketConfig implements WebSocketConfigurer {
   	@Autowired
   	CpWebSocketInterceptor cpWebSocketInterceptor;
   	@Autowired
   	CpWebSocketHandler cpWebSocketHandler;
   
   	@Override
   	public void registerWebSocketHandlers(WebSocketHandlerRegistry webSocketHandlerRegistry) {
   		webSocketHandlerRegistry
   				//添加myHandler消息处理对象，和websocket访问地址
   				.addHandler(cpWebSocketHandler, "/cpSocket")
   				//设置允许跨域访问
   				.setAllowedOrigins("*")
   				//添加拦截器可实现用户链接前进行权限校验等操作
   				.addInterceptors(cpWebSocketInterceptor);
   	}
   }
   
   ```

   

