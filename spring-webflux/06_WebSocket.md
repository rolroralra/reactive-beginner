# WebSocket

## 1. 개요

WebSocket은 **양방향 실시간 통신**을 위한 프로토콜입니다. HTTP와 달리 연결이 유지되며, 서버와 클라이언트 모두 데이터를 전송할 수 있습니다.

### 1.1 HTTP vs WebSocket

```
HTTP (단방향):
Client ──Request──→ Server
Client ←─Response── Server

WebSocket (양방향):
Client ←───────────→ Server
       (실시간 양방향)
```

### 1.2 사용 사례

- 실시간 채팅
- 실시간 알림
- 게임
- 주식/가상화폐 실시간 시세
- 협업 도구 (Google Docs 같은)

## 2. 의존성

```groovy
implementation 'org.springframework.boot:spring-boot-starter-webflux'
// WebSocket은 webflux에 포함되어 있음
```

## 3. 서버 구현

### 3.1 WebSocketHandler 구현

```java
package com.example.websocket;

import org.springframework.stereotype.Component;
import org.springframework.web.reactive.socket.WebSocketHandler;
import org.springframework.web.reactive.socket.WebSocketMessage;
import org.springframework.web.reactive.socket.WebSocketSession;
import reactor.core.publisher.Mono;

@Component
public class EchoWebSocketHandler implements WebSocketHandler {

    @Override
    public Mono<Void> handle(WebSocketSession session) {
        // 받은 메시지를 그대로 에코
        return session.send(
            session.receive()
                .map(WebSocketMessage::getPayloadAsText)
                .map(text -> "Echo: " + text)
                .map(session::textMessage)
        );
    }
}
```

### 3.2 WebSocket 설정

```java
package com.example.config;

import com.example.websocket.EchoWebSocketHandler;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.HandlerMapping;
import org.springframework.web.reactive.handler.SimpleUrlHandlerMapping;
import org.springframework.web.reactive.socket.WebSocketHandler;
import org.springframework.web.reactive.socket.server.support.WebSocketHandlerAdapter;

import java.util.Map;

@Configuration
public class WebSocketConfig {

    @Bean
    public HandlerMapping webSocketMapping(EchoWebSocketHandler echoHandler) {
        Map<String, WebSocketHandler> map = Map.of(
            "/ws/echo", echoHandler
        );

        SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
        mapping.setUrlMap(map);
        mapping.setOrder(-1);  // 높은 우선순위
        return mapping;
    }

    @Bean
    public WebSocketHandlerAdapter handlerAdapter() {
        return new WebSocketHandlerAdapter();
    }
}
```

## 4. 고급 핸들러 구현

### 4.1 채팅 서버

```java
package com.example.websocket;

import org.springframework.stereotype.Component;
import org.springframework.web.reactive.socket.WebSocketHandler;
import org.springframework.web.reactive.socket.WebSocketMessage;
import org.springframework.web.reactive.socket.WebSocketSession;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import reactor.core.publisher.Sinks;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class ChatWebSocketHandler implements WebSocketHandler {

    // 모든 세션에 브로드캐스트하기 위한 Sink
    private final Sinks.Many<String> chatSink = Sinks.many().multicast().onBackpressureBuffer();
    private final Flux<String> chatFlux = chatSink.asFlux();

    // 연결된 세션 관리
    private final Map<String, WebSocketSession> sessions = new ConcurrentHashMap<>();

    @Override
    public Mono<Void> handle(WebSocketSession session) {
        String sessionId = session.getId();
        sessions.put(sessionId, session);

        // 입장 메시지
        chatSink.tryEmitNext("User " + sessionId + " joined");

        // 메시지 수신 및 브로드캐스트
        Mono<Void> receive = session.receive()
            .map(WebSocketMessage::getPayloadAsText)
            .doOnNext(message -> chatSink.tryEmitNext(sessionId + ": " + message))
            .doOnTerminate(() -> {
                sessions.remove(sessionId);
                chatSink.tryEmitNext("User " + sessionId + " left");
            })
            .then();

        // 메시지 전송
        Mono<Void> send = session.send(
            chatFlux.map(session::textMessage)
        );

        return Mono.zip(receive, send).then();
    }
}
```

### 4.2 JSON 메시지 처리

```java
package com.example.websocket;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.socket.WebSocketHandler;
import org.springframework.web.reactive.socket.WebSocketMessage;
import org.springframework.web.reactive.socket.WebSocketSession;
import reactor.core.publisher.Mono;

@Component
public class JsonWebSocketHandler implements WebSocketHandler {

    private final ObjectMapper objectMapper;

    public JsonWebSocketHandler(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public Mono<Void> handle(WebSocketSession session) {
        return session.send(
            session.receive()
                .map(WebSocketMessage::getPayloadAsText)
                .flatMap(this::processMessage)
                .map(session::textMessage)
        );
    }

    private Mono<String> processMessage(String json) {
        return Mono.fromCallable(() -> {
            ChatMessage message = objectMapper.readValue(json, ChatMessage.class);

            ChatResponse response = new ChatResponse(
                "received",
                message.getContent(),
                System.currentTimeMillis()
            );

            return objectMapper.writeValueAsString(response);
        });
    }
}

// DTO 클래스
record ChatMessage(String type, String content) {}
record ChatResponse(String status, String content, long timestamp) {}
```

### 4.3 실시간 데이터 스트리밍

```java
package com.example.websocket;

import org.springframework.stereotype.Component;
import org.springframework.web.reactive.socket.WebSocketHandler;
import org.springframework.web.reactive.socket.WebSocketSession;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.time.Duration;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

@Component
public class StockPriceWebSocketHandler implements WebSocketHandler {

    @Override
    public Mono<Void> handle(WebSocketSession session) {
        // 1초마다 주가 데이터 전송
        Flux<String> stockPrices = Flux.interval(Duration.ofSeconds(1))
            .map(i -> generateStockPrice())
            .map(this::toJson);

        return session.send(stockPrices.map(session::textMessage));
    }

    private StockPrice generateStockPrice() {
        double price = 100 + Math.random() * 10;
        return new StockPrice(
            "AAPL",
            Math.round(price * 100) / 100.0,
            LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME)
        );
    }

    private String toJson(StockPrice stockPrice) {
        return String.format(
            "{\"symbol\":\"%s\",\"price\":%.2f,\"timestamp\":\"%s\"}",
            stockPrice.symbol(),
            stockPrice.price(),
            stockPrice.timestamp()
        );
    }
}

record StockPrice(String symbol, double price, String timestamp) {}
```

## 5. WebSocket 클라이언트

### 5.1 WebSocketClient 사용

```java
package com.example.client;

import org.springframework.web.reactive.socket.WebSocketMessage;
import org.springframework.web.reactive.socket.client.ReactorNettyWebSocketClient;
import org.springframework.web.reactive.socket.client.WebSocketClient;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.net.URI;
import java.time.Duration;

public class WebSocketClientExample {

    public static void main(String[] args) {
        WebSocketClient client = new ReactorNettyWebSocketClient();

        URI uri = URI.create("ws://localhost:8080/ws/echo");

        client.execute(uri, session -> {
            // 메시지 전송
            Flux<WebSocketMessage> output = Flux.interval(Duration.ofSeconds(1))
                .map(i -> "Message " + i)
                .map(session::textMessage)
                .take(5);

            // 메시지 수신
            Mono<Void> receive = session.receive()
                .map(WebSocketMessage::getPayloadAsText)
                .doOnNext(msg -> System.out.println("Received: " + msg))
                .then();

            // 전송 및 수신
            return session.send(output).thenMany(receive).then();
        }).block(Duration.ofSeconds(10));
    }
}
```

### 5.2 서비스에서 WebSocket 클라이언트 사용

```java
package com.example.service;

import org.springframework.stereotype.Service;
import org.springframework.web.reactive.socket.WebSocketMessage;
import org.springframework.web.reactive.socket.client.ReactorNettyWebSocketClient;
import org.springframework.web.reactive.socket.client.WebSocketClient;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import reactor.core.publisher.Sinks;

import java.net.URI;

@Service
public class WebSocketClientService {

    private final WebSocketClient client = new ReactorNettyWebSocketClient();
    private final Sinks.Many<String> messageSink = Sinks.many().unicast().onBackpressureBuffer();

    public Flux<String> connect(String url) {
        Mono<Void> connection = client.execute(
            URI.create(url),
            session -> session.receive()
                .map(WebSocketMessage::getPayloadAsText)
                .doOnNext(messageSink::tryEmitNext)
                .then()
        );

        return messageSink.asFlux()
            .doOnSubscribe(s -> connection.subscribe());
    }
}
```

## 6. 세션 관리

### 6.1 세션 속성

```java
@Override
public Mono<Void> handle(WebSocketSession session) {
    // 세션 ID
    String sessionId = session.getId();

    // 핸드셰이크 정보
    HandshakeInfo handshakeInfo = session.getHandshakeInfo();
    URI uri = handshakeInfo.getUri();
    HttpHeaders headers = handshakeInfo.getHeaders();
    InetSocketAddress remoteAddress = handshakeInfo.getRemoteAddress();

    // 쿼리 파라미터 추출
    String query = uri.getQuery();  // e.g., "userId=123&room=general"

    return session.send(/* ... */);
}
```

### 6.2 인증 정보 접근

```java
@Override
public Mono<Void> handle(WebSocketSession session) {
    return session.getHandshakeInfo().getPrincipal()
        .map(principal -> principal.getName())
        .defaultIfEmpty("anonymous")
        .flatMap(username -> {
            // 인증된 사용자로 처리
            return handleAuthenticatedSession(session, username);
        });
}

private Mono<Void> handleAuthenticatedSession(WebSocketSession session, String username) {
    return session.send(
        session.receive()
            .map(WebSocketMessage::getPayloadAsText)
            .map(msg -> username + ": " + msg)
            .map(session::textMessage)
    );
}
```

## 7. 에러 처리

```java
@Override
public Mono<Void> handle(WebSocketSession session) {
    return session.receive()
        .map(WebSocketMessage::getPayloadAsText)
        .flatMap(this::processMessage)
        .map(session::textMessage)
        .as(session::send)
        .doOnError(error -> log.error("WebSocket error: {}", error.getMessage()))
        .onErrorResume(error -> {
            String errorMessage = "{\"error\":\"" + error.getMessage() + "\"}";
            return session.send(Mono.just(session.textMessage(errorMessage)))
                .then(session.close());
        });
}

private Mono<String> processMessage(String message) {
    if (message.isEmpty()) {
        return Mono.error(new IllegalArgumentException("Empty message"));
    }
    return Mono.just("Processed: " + message);
}
```

## 8. Heartbeat (핑-퐁)

```java
@Override
public Mono<Void> handle(WebSocketSession session) {
    // 30초마다 핑 전송
    Flux<WebSocketMessage> pingFlux = Flux.interval(Duration.ofSeconds(30))
        .map(i -> session.pingMessage(factory ->
            factory.wrap("ping".getBytes())));

    Flux<WebSocketMessage> messageFlux = session.receive()
        .filter(msg -> msg.getType() != WebSocketMessage.Type.PONG)
        .map(msg -> session.textMessage("Echo: " + msg.getPayloadAsText()));

    return session.send(Flux.merge(pingFlux, messageFlux));
}
```

## 9. CORS 설정

```java
@Configuration
public class WebSocketConfig {

    @Bean
    public WebSocketHandlerAdapter handlerAdapter() {
        return new WebSocketHandlerAdapter(webSocketService());
    }

    @Bean
    public WebSocketService webSocketService() {
        return new HandshakeWebSocketService(new ReactorNettyRequestUpgradeStrategy() {
            @Override
            public Mono<Void> upgrade(ServerWebExchange exchange,
                                       WebSocketHandler handler,
                                       String subProtocol,
                                       Supplier<HandshakeInfo> handshakeInfoFactory) {
                // CORS 헤더 설정
                exchange.getResponse().getHeaders()
                    .add("Access-Control-Allow-Origin", "*");
                return super.upgrade(exchange, handler, subProtocol, handshakeInfoFactory);
            }
        });
    }
}
```

## 10. JavaScript 클라이언트

```html
<!DOCTYPE html>
<html>
<head>
    <title>WebSocket Chat</title>
</head>
<body>
    <div id="messages"></div>
    <input type="text" id="input" placeholder="메시지 입력">
    <button onclick="sendMessage()">전송</button>

    <script>
        const ws = new WebSocket('ws://localhost:8080/ws/chat');

        ws.onopen = () => {
            console.log('Connected');
        };

        ws.onmessage = (event) => {
            const messagesDiv = document.getElementById('messages');
            messagesDiv.innerHTML += `<p>${event.data}</p>`;
        };

        ws.onclose = () => {
            console.log('Disconnected');
        };

        ws.onerror = (error) => {
            console.error('WebSocket Error:', error);
        };

        function sendMessage() {
            const input = document.getElementById('input');
            ws.send(input.value);
            input.value = '';
        }
    </script>
</body>
</html>
```

## 11. 다음 단계

- [Server-Sent Events](07_Server_Sent_Events.md): 서버 → 클라이언트 단방향 푸시
- [예외 처리](08_예외_처리.md): 에러 핸들링
- [Testing](10_Testing.md): WebSocket 테스트
