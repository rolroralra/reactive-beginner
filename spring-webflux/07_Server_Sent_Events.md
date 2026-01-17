# Server-Sent Events (SSE)

## 1. 개요

Server-Sent Events(SSE)는 **서버에서 클라이언트로 실시간 데이터를 푸시**하는 기술입니다. WebSocket과 달리 단방향 통신이며, HTTP 기반으로 동작합니다.

### 1.1 SSE vs WebSocket

| 특성 | SSE | WebSocket |
|------|-----|-----------|
| **방향** | 단방향 (서버→클라이언트) | 양방향 |
| **프로토콜** | HTTP | WebSocket (WS) |
| **자동 재연결** | O | X (직접 구현) |
| **브라우저 지원** | 대부분 지원 | 대부분 지원 |
| **데이터 형식** | 텍스트만 | 텍스트/바이너리 |
| **복잡도** | 낮음 | 높음 |

### 1.2 사용 사례

- 실시간 알림
- 뉴스 피드
- 주식 시세 업데이트
- 소셜 미디어 타임라인
- 진행률 표시
- 로그 스트리밍

## 2. 기본 구현

### 2.1 간단한 SSE 엔드포인트

```java
package com.example.controller;

import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;

import java.time.Duration;
import java.time.LocalDateTime;

@RestController
public class SseController {

    @GetMapping(value = "/sse/time", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamTime() {
        return Flux.interval(Duration.ofSeconds(1))
            .map(seq -> "Current time: " + LocalDateTime.now());
    }
}
```

### 2.2 ServerSentEvent 사용

```java
@GetMapping(value = "/sse/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> streamEvents() {
    return Flux.interval(Duration.ofSeconds(1))
        .map(seq -> ServerSentEvent.<String>builder()
            .id(String.valueOf(seq))
            .event("message")
            .data("Event " + seq + " at " + LocalDateTime.now())
            .retry(Duration.ofSeconds(5))
            .comment("This is a comment")
            .build());
}
```

### 2.3 SSE 메시지 형식

```
id: 1
event: message
retry: 5000
data: Event 1 at 2024-01-15T10:30:00

id: 2
event: message
retry: 5000
data: Event 2 at 2024-01-15T10:30:01
```

## 3. 실용 예제

### 3.1 실시간 알림 시스템

```java
package com.example.service;

import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Sinks;

@Service
public class NotificationService {

    private final Sinks.Many<Notification> notificationSink =
        Sinks.many().multicast().onBackpressureBuffer();

    public void sendNotification(Notification notification) {
        notificationSink.tryEmitNext(notification);
    }

    public Flux<Notification> getNotificationStream() {
        return notificationSink.asFlux();
    }

    public Flux<Notification> getNotificationStreamForUser(String userId) {
        return notificationSink.asFlux()
            .filter(n -> n.getUserId().equals(userId));
    }
}

record Notification(String id, String userId, String message, String type, long timestamp) {}
```

```java
package com.example.controller;

import com.example.service.Notification;
import com.example.service.NotificationService;
import org.springframework.http.MediaType;
import org.springframework.http.codec.ServerSentEvent;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/notifications")
public class NotificationController {

    private final NotificationService notificationService;

    public NotificationController(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    // SSE 스트림 구독
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<Notification>> streamNotifications() {
        return notificationService.getNotificationStream()
            .map(notification -> ServerSentEvent.<Notification>builder()
                .id(notification.id())
                .event(notification.type())
                .data(notification)
                .build());
    }

    // 특정 사용자의 알림 스트림
    @GetMapping(value = "/stream/{userId}", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<Notification>> streamUserNotifications(
            @PathVariable String userId) {
        return notificationService.getNotificationStreamForUser(userId)
            .map(notification -> ServerSentEvent.<Notification>builder()
                .id(notification.id())
                .event(notification.type())
                .data(notification)
                .build());
    }

    // 알림 발송
    @PostMapping
    public Mono<Void> sendNotification(@RequestBody Notification notification) {
        notificationService.sendNotification(notification);
        return Mono.empty();
    }
}
```

### 3.2 주식 시세 스트리밍

```java
package com.example.service;

import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;

import java.time.Duration;
import java.util.Map;
import java.util.Random;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class StockService {

    private final Map<String, Double> stockPrices = new ConcurrentHashMap<>();
    private final Random random = new Random();

    public StockService() {
        // 초기 가격 설정
        stockPrices.put("AAPL", 150.0);
        stockPrices.put("GOOGL", 2800.0);
        stockPrices.put("MSFT", 300.0);
        stockPrices.put("AMZN", 3300.0);
    }

    public Flux<StockPrice> getStockPriceStream(String symbol) {
        return Flux.interval(Duration.ofMillis(500))
            .map(i -> {
                double currentPrice = stockPrices.get(symbol);
                double change = (random.nextDouble() - 0.5) * 2;  // -1 ~ +1
                double newPrice = Math.round((currentPrice + change) * 100) / 100.0;
                stockPrices.put(symbol, newPrice);

                return new StockPrice(
                    symbol,
                    newPrice,
                    change > 0 ? "UP" : "DOWN",
                    System.currentTimeMillis()
                );
            });
    }

    public Flux<StockPrice> getAllStockPricesStream() {
        return Flux.interval(Duration.ofSeconds(1))
            .flatMap(i -> Flux.fromIterable(stockPrices.keySet())
                .map(symbol -> {
                    double currentPrice = stockPrices.get(symbol);
                    double change = (random.nextDouble() - 0.5) * 2;
                    double newPrice = Math.round((currentPrice + change) * 100) / 100.0;
                    stockPrices.put(symbol, newPrice);

                    return new StockPrice(
                        symbol,
                        newPrice,
                        change > 0 ? "UP" : "DOWN",
                        System.currentTimeMillis()
                    );
                }));
    }
}

record StockPrice(String symbol, double price, String direction, long timestamp) {}
```

```java
@RestController
@RequestMapping("/stocks")
public class StockController {

    private final StockService stockService;

    public StockController(StockService stockService) {
        this.stockService = stockService;
    }

    @GetMapping(value = "/stream/{symbol}", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<StockPrice>> streamStock(@PathVariable String symbol) {
        return stockService.getStockPriceStream(symbol)
            .map(price -> ServerSentEvent.<StockPrice>builder()
                .event("stock-update")
                .data(price)
                .build());
    }

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<StockPrice>> streamAllStocks() {
        return stockService.getAllStockPricesStream()
            .map(price -> ServerSentEvent.<StockPrice>builder()
                .event("stock-update")
                .data(price)
                .build());
    }
}
```

### 3.3 작업 진행률 표시

```java
package com.example.service;

import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Sinks;

import java.time.Duration;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class TaskService {

    private final Map<String, Sinks.Many<TaskProgress>> taskSinks = new ConcurrentHashMap<>();

    public String startTask(String taskName) {
        String taskId = UUID.randomUUID().toString();
        Sinks.Many<TaskProgress> sink = Sinks.many().unicast().onBackpressureBuffer();
        taskSinks.put(taskId, sink);

        // 비동기 작업 시뮬레이션
        simulateTask(taskId, taskName, sink);

        return taskId;
    }

    public Flux<TaskProgress> getTaskProgress(String taskId) {
        Sinks.Many<TaskProgress> sink = taskSinks.get(taskId);
        if (sink == null) {
            return Flux.error(new IllegalArgumentException("Task not found: " + taskId));
        }
        return sink.asFlux();
    }

    private void simulateTask(String taskId, String taskName, Sinks.Many<TaskProgress> sink) {
        Flux.interval(Duration.ofMillis(500))
            .take(20)
            .doOnNext(i -> {
                int progress = (int) ((i + 1) * 5);
                String status = progress < 100 ? "IN_PROGRESS" : "COMPLETED";
                sink.tryEmitNext(new TaskProgress(taskId, taskName, progress, status));
            })
            .doOnComplete(() -> {
                sink.tryEmitComplete();
                taskSinks.remove(taskId);
            })
            .subscribe();
    }
}

record TaskProgress(String taskId, String taskName, int progress, String status) {}
```

```java
@RestController
@RequestMapping("/tasks")
public class TaskController {

    private final TaskService taskService;

    public TaskController(TaskService taskService) {
        this.taskService = taskService;
    }

    @PostMapping("/start")
    public Mono<Map<String, String>> startTask(@RequestParam String name) {
        String taskId = taskService.startTask(name);
        return Mono.just(Map.of("taskId", taskId));
    }

    @GetMapping(value = "/{taskId}/progress", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<TaskProgress>> getProgress(@PathVariable String taskId) {
        return taskService.getTaskProgress(taskId)
            .map(progress -> ServerSentEvent.<TaskProgress>builder()
                .id(String.valueOf(progress.progress()))
                .event("progress")
                .data(progress)
                .build());
    }
}
```

## 4. Heartbeat 구현

연결 유지를 위한 heartbeat 전송:

```java
@GetMapping(value = "/sse/with-heartbeat", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> streamWithHeartbeat() {
    // 실제 데이터 스트림
    Flux<ServerSentEvent<String>> dataStream = notificationService.getNotificationStream()
        .map(n -> ServerSentEvent.<String>builder()
            .event("notification")
            .data(n.message())
            .build());

    // 15초마다 heartbeat
    Flux<ServerSentEvent<String>> heartbeat = Flux.interval(Duration.ofSeconds(15))
        .map(i -> ServerSentEvent.<String>builder()
            .event("heartbeat")
            .data("")
            .build());

    return Flux.merge(dataStream, heartbeat);
}
```

## 5. 에러 처리

```java
@GetMapping(value = "/sse/with-error-handling", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> streamWithErrorHandling() {
    return someService.getDataStream()
        .map(data -> ServerSentEvent.<String>builder()
            .event("data")
            .data(data)
            .build())
        .onErrorResume(error -> {
            log.error("SSE Error: {}", error.getMessage());
            return Flux.just(ServerSentEvent.<String>builder()
                .event("error")
                .data("Error: " + error.getMessage())
                .build());
        })
        .doOnCancel(() -> log.info("Client disconnected"));
}
```

## 6. 클라이언트 구현

### 6.1 JavaScript 클라이언트

```html
<!DOCTYPE html>
<html>
<head>
    <title>SSE Demo</title>
</head>
<body>
    <h1>실시간 알림</h1>
    <div id="notifications"></div>

    <script>
        const eventSource = new EventSource('/notifications/stream');

        // 기본 message 이벤트
        eventSource.onmessage = (event) => {
            console.log('Message:', event.data);
        };

        // 특정 이벤트 타입 수신
        eventSource.addEventListener('notification', (event) => {
            const notification = JSON.parse(event.data);
            const div = document.getElementById('notifications');
            div.innerHTML += `
                <div class="notification ${notification.type}">
                    <strong>${notification.type}</strong>: ${notification.message}
                </div>
            `;
        });

        // 에러 이벤트
        eventSource.addEventListener('error', (event) => {
            console.error('SSE Error:', event.data);
        });

        // heartbeat 이벤트
        eventSource.addEventListener('heartbeat', () => {
            console.log('Heartbeat received');
        });

        // 연결 에러
        eventSource.onerror = (error) => {
            console.error('Connection error:', error);
            if (eventSource.readyState === EventSource.CLOSED) {
                console.log('Connection was closed');
            }
        };

        // 페이지 종료 시 연결 닫기
        window.addEventListener('beforeunload', () => {
            eventSource.close();
        });
    </script>
</body>
</html>
```

### 6.2 주식 시세 UI

```html
<!DOCTYPE html>
<html>
<head>
    <title>Stock Prices</title>
    <style>
        .up { color: green; }
        .down { color: red; }
        table { border-collapse: collapse; width: 100%; }
        td, th { border: 1px solid #ddd; padding: 8px; }
    </style>
</head>
<body>
    <h1>실시간 주식 시세</h1>
    <table>
        <thead>
            <tr>
                <th>Symbol</th>
                <th>Price</th>
                <th>Direction</th>
            </tr>
        </thead>
        <tbody id="stocks"></tbody>
    </table>

    <script>
        const stocks = {};
        const eventSource = new EventSource('/stocks/stream');

        eventSource.addEventListener('stock-update', (event) => {
            const stock = JSON.parse(event.data);
            stocks[stock.symbol] = stock;
            renderStocks();
        });

        function renderStocks() {
            const tbody = document.getElementById('stocks');
            tbody.innerHTML = Object.values(stocks)
                .map(stock => `
                    <tr>
                        <td>${stock.symbol}</td>
                        <td class="${stock.direction.toLowerCase()}">
                            $${stock.price.toFixed(2)}
                        </td>
                        <td class="${stock.direction.toLowerCase()}">
                            ${stock.direction === 'UP' ? '▲' : '▼'}
                        </td>
                    </tr>
                `)
                .join('');
        }
    </script>
</body>
</html>
```

### 6.3 WebClient로 SSE 수신

```java
@Service
public class SseClientService {

    private final WebClient webClient;

    public SseClientService(WebClient.Builder webClientBuilder) {
        this.webClient = webClientBuilder.baseUrl("http://localhost:8080").build();
    }

    public Flux<String> subscribeToEvents() {
        return webClient.get()
            .uri("/sse/events")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .retrieve()
            .bodyToFlux(String.class)
            .doOnNext(event -> log.info("Received: {}", event))
            .doOnError(error -> log.error("Error: {}", error.getMessage()));
    }

    public Flux<ServerSentEvent<String>> subscribeToTypedEvents() {
        ParameterizedTypeReference<ServerSentEvent<String>> type =
            new ParameterizedTypeReference<>() {};

        return webClient.get()
            .uri("/sse/events")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .retrieve()
            .bodyToFlux(type)
            .doOnNext(event -> log.info("Event: id={}, type={}, data={}",
                event.id(), event.event(), event.data()));
    }
}
```

## 7. 연결 관리

### 7.1 타임아웃 설정

```java
@GetMapping(value = "/sse/with-timeout", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> streamWithTimeout() {
    return dataStream
        .timeout(Duration.ofMinutes(30))
        .onErrorResume(TimeoutException.class, e ->
            Flux.just(ServerSentEvent.<String>builder()
                .event("timeout")
                .data("Connection timed out")
                .build()));
}
```

### 7.2 최대 이벤트 수 제한

```java
@GetMapping(value = "/sse/limited", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> streamLimited() {
    return dataStream
        .take(100)  // 최대 100개 이벤트
        .concatWith(Flux.just(ServerSentEvent.<String>builder()
            .event("complete")
            .data("Stream completed")
            .build()));
}
```

## 8. 다음 단계

- [예외 처리](08_예외_처리.md): 에러 핸들링
- [WebSocket](06_WebSocket.md): 양방향 실시간 통신
- [Testing](10_Testing.md): SSE 테스트
