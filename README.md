# lost-exception-lab

옛 이름은 ExceptionRadar다. 아래 본문의 ExceptionRadar는 그 이름 그대로 둔 설계안이다.

## 상태와 다음 할 일

- 질문: 멀티스레드 · 멀티프로세스 경계에서 예외는 어디로 사라지나. 다른 스레드(`ExecutorService`, `CompletableFuture`, 가상 스레드)나 자식 프로세스에서 난 예외를 호출한 쪽이 알 수 있는가.
- 상태: **예정**. 있는 것은 이 README(설계 · 로드맵), `build.gradle` · `settings.gradle`, 설정 파일(`application.yml` · `application.properties` · `logback.xml`)뿐이다. Java 소스와 테스트는 없고, 아래 로드맵 체크리스트도 전부 미체크다. 실행 결과 기록도 없다.
- 이 README의 코드 예시 · 설정 · 프로젝트 구조 · 실행 명령은 설계안이다. 그 API(`ExceptionRadar.builder()` 등)와 `Main.java`는 아직 없다.
- 다음 할 일:
  1. 질문을 랩 형태로 다시 세운다. 예외가 사라지는 경우를 먼저 재현하는 베이스라인을 만들고, 그다음 해법을 비교한다. 이 일은 다음 명세에서 한다.
  2. 그때 아래 8단계 로드맵 중 무엇을 남길지 정하고, 단계 문서는 `docs/adr/` ADR 형식으로 쓴다.

> Java 애플리케이션에서 프로세스 및 스레드 경계를 넘어 예외를 추적하고 감시하는 시스템

## 개요

ExceptionRadar는 Java 멀티스레드/멀티프로세스 환경에서 "잃어버린 예외"를 추적합니다. async/await 패턴, ExecutorService, Virtual Thread 등에서 발생하는 예외를 감지하고 중앙에서 모니터링할 수 있는 메커니즘을 제공합니다.

### 핵심 문제 정의

```java
// 이 예외는 어디로 갔을까?
try {
    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
        throw new RuntimeException("This exception is lost!");
    });
    // main 스레드는 이 예외를 잡을 수 없음
} catch (Exception e) {
    // 절대 실행되지 않음
}
```

**ExceptionRadar는 이러한 "잃어버린 예외"들을 추적합니다.**

## 프로젝트 목표

1. **크로스 스레드 예외 추적**: 독립된 스레드에서 발생한 예외를 메인 스레드에서 인지
2. **크로스 프로세스 예외 추적**: 자식 프로세스에서 발생한 예외를 부모 프로세스에서 감지
3. **JVM/OS 레벨 모니터링**:
    - Java Agent를 통한 기존 코드 자동 계측
    - JVM 시그널 핸들링 (SIGSEGV, SIGABRT 등)
    - Native Crash 감지
4. **실시간 예외 리포팅**: 예외 발생 즉시 중앙 모니터링 시스템으로 전달
5. **예외 컨텍스트 보존**: 예외가 발생한 전체 컨텍스트와 스택 트레이스 유지

## 아키텍처

### 시스템 구성도

```
┌─────────────────────────────────────────────────────────────────┐
│                      ExceptionRadar Core                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐    │
│  │    Thread      │  │    Process     │  │   Async        │    │
│  │    Monitor     │  │    Monitor     │  │   Tracker      │    │
│  └───────┬────────┘  └───────┬────────┘  └───────┬────────┘    │
│          │                   │                   │              │
│  ┌───────▼───────────────────▼───────────────────▼───────┐     │
│  │                  Exception Collector                   │     │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │     │
│  │  │  Context    │ │ StackTrace  │ │  Causation  │     │     │
│  │  │  Builder    │ │  Analyzer   │ │  Chain      │     │     │
│  │  └─────────────┘ └─────────────┘ └─────────────┘     │     │
│  └───────────────────────────┬───────────────────────────┘     │
│                              │                                  │
│  ┌───────────────────────────▼───────────────────────────┐     │
│  │                    Event Bus                           │     │
│  │            (Pub/Sub + Ring Buffer)                     │     │
│  └──┬──────────────┬──────────────┬──────────────┬───────┘     │
│     │              │              │              │              │
│  ┌──▼────┐    ┌────▼────┐   ┌────▼────┐   ┌────▼────┐        │
│  │Report │    │ Analyze │   │ Storage │   │ Alert   │        │
│  │  er   │    │    r    │   │         │   │ System  │        │
│  └───────┘    └─────────┘   └─────────┘   └─────────┘        │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                      Dashboard (Web UI)                          │
├─────────────────────────────────────────────────────────────────┤
│                      Java Agent (OS Level)                       │
└─────────────────────────────────────────────────────────────────┘
```

## 구현 로드맵

### Phase 1: Foundation (기반)
> 핵심 모델과 이벤트 시스템 구축

- [ ] Core 모델 (ExceptionContext, ExceptionEvent)
- [ ] 설정 시스템 (ExceptionRadarConfig, YAML 로더)
- [ ] Event Bus (Ring Buffer 기반)
- [ ] Handler 인터페이스 정의

**학습 포인트**: Record, Builder 패턴, Lock-free 자료구조

### Phase 2: Thread Tracking (스레드 추적)
> 스레드 레벨 예외 추적 구현

- [ ] UncaughtExceptionHandler 글로벌 설정
- [ ] ThreadPool Wrapper (ExecutorService 래핑)
- [ ] CompletableFuture Tracker
- [ ] Virtual Thread Monitor (Java 21)
- [ ] Context Propagation (ThreadLocal 기반)

**학습 포인트**: Java Concurrency 심화, ThreadLocal, Virtual Thread

### Phase 3: Reporting (리포팅)
> 예외 리포팅 시스템 구현

- [ ] ConsoleReporter
- [ ] FileReporter (JSON/Text 포맷)
- [ ] MetricsReporter (Micrometer 연동)

**학습 포인트**: SLF4J 통합, Micrometer 메트릭

### Phase 4: Process Tracking (프로세스 추적)
> 프로세스 간 예외 전파 시스템 구현

- [ ] ManagedProcess (ProcessBuilder 래핑)
- [ ] Unix Domain Socket IPC
- [ ] 통신 프로토콜 설계 (JSON 기반)
- [ ] ProcessRegistry (활성 프로세스 관리)
- [ ] Child Process Agent (자식용 경량 모듈)

**학습 포인트**: IPC, Unix Domain Socket, 프로토콜 설계

### Phase 5: Analysis & Resilience (분석/복원)
> 예외 분석 및 자동 복구 메커니즘 구현

- [ ] Pattern Detector (예외 패턴 인식)
- [ ] Historical Analyzer (시계열 분석)
- [ ] Retry Mechanism (자동 재시도)
- [ ] Circuit Breaker (서킷 브레이커)

**학습 포인트**: 시계열 분석, 상태 머신, 디자인 패턴

### Phase 6: Dashboard & Alert (대시보드/알림)
> 실시간 모니터링 대시보드 및 알림 시스템 구현

- [ ] 내장 HTTP 서버 (com.sun.net.httpserver)
- [ ] WebSocket 핸들러
- [ ] REST API
- [ ] 웹 UI (정적 파일)
- [ ] Alert Manager
- [ ] Alert Channels (Slack, Email, Webhook)

**학습 포인트**: HTTP 서버, WebSocket, 비동기 I/O

### Phase 7: OS Level Monitoring (OS 레벨)
> Java Agent 및 네이티브 레벨 모니터링 구현

- [ ] Java Agent 기초 (Instrumentation API)
- [ ] 바이트코드 조작 (ByteBuddy)
- [ ] 기존 스레드풀 자동 가로채기
- [ ] JVM Signal Handler (SIGSEGV, SIGABRT 등)
- [ ] Native Crash 감지 (JNI)

**학습 포인트**: Java Agent, 바이트코드 조작, JNI, 시스템 프로그래밍

### Phase 8: Distributed Tracing (분산 추적)
> 분산 환경에서의 예외 추적 구현

- [ ] Trace Context 모델
- [ ] OpenTelemetry 연동
- [ ] 크로스 서비스 예외 전파

**학습 포인트**: 분산 시스템, OpenTelemetry, 컨텍스트 전파

## 기술 스택

### Core (Phase 1-6)
- **Java 21**: Virtual Threads, Pattern Matching, Record Classes
- **Concurrency**: CompletableFuture, ExecutorService, ForkJoinPool
- **IPC**: Unix Domain Sockets (Java 16+)
- **Serialization**: Jackson (JSON)
- **Monitoring**: Micrometer
- **Logging**: SLF4J + Logback

### Advanced (Phase 7-8)
- **Agent**: ByteBuddy (바이트코드 조작)
- **Native**: JNI (네이티브 크래시 감지)
- **Tracing**: OpenTelemetry

## 주요 기능

> 설계안이다. 아래 API는 아직 없다.

### 1. Thread Exception Tracking

```java
// ExceptionRadar 초기화
ExceptionRadar radar = ExceptionRadar.builder()
    .enableThreadTracking()
    .build();

// 스레드풀 감시
radar.trackThreadPool(executorService);

// 이제 모든 스레드의 예외가 추적됨
executorService.submit(() -> {
    throw new RuntimeException("This will be caught by radar!");
});
```

### 2. Process Exception Tracking

```java
// 자식 프로세스 실행 및 감시
ProcessMonitor monitor = radar.createProcessMonitor();
Process child = monitor.execute("java", "-jar", "child-app.jar");

// 자식 프로세스의 예외를 부모에서 수신
monitor.onException(exception -> {
    log.error("Child process exception: ", exception);
});

// 활성 프로세스 조회
ProcessRegistry registry = monitor.getRegistry();
registry.getActiveProcesses().forEach(p -> 
    log.info("Process {} is running", p.pid())
);
```

### 3. Async/Await Exception Handling

```java
// CompletableFuture 체인에서 예외 추적
radar.trackAsync(() -> 
    CompletableFuture.supplyAsync(() -> processData())
        .thenApplyAsync(data -> transformData(data))
        .thenAcceptAsync(result -> saveResult(result))
);

// 체인의 어느 단계에서든 발생한 예외가 추적됨
```

### 4. Real-time Exception Dashboard

```java
// 실시간 대시보드 시작
ExceptionDashboard dashboard = radar.startDashboard(8080);

// 웹 브라우저에서 http://localhost:8080 접속
// 실시간 예외 발생 현황, 통계, 분석 결과 확인
```

### 5. Exception Pattern Analysis

```java
// 예외 패턴 분석기 설정
ExceptionAnalyzer analyzer = radar.getAnalyzer();

// 패턴 감지 시 콜백
analyzer.onPatternDetected(pattern -> {
    log.warn("Recurring exception pattern detected: {}", pattern);
    if (pattern.frequency() > 10) {
        alertManager.sendAlert(pattern);
    }
});
```

### 6. Auto Retry & Circuit Breaker

```java
// 재시도 메커니즘 설정
RetryConfig retryConfig = RetryConfig.builder()
    .maxAttempts(3)
    .backoff(Duration.ofSeconds(1))
    .retryOn(TransientException.class)
    .build();

// Circuit Breaker 설정
CircuitBreakerConfig cbConfig = CircuitBreakerConfig.builder()
    .failureThreshold(5)
    .resetTimeout(Duration.ofMinutes(1))
    .build();

radar.withResilience(retryConfig, cbConfig);
```

### 7. Java Agent (자동 계측)

```bash
# Agent를 사용하여 기존 애플리케이션 계측
java -javaagent:exception-radar-agent.jar -jar your-app.jar
```

```java
// Agent 설정 (agent.yml)
exception-radar-agent:
  intercept:
    - java.util.concurrent.ExecutorService
    - java.util.concurrent.ForkJoinPool
  exclude:
    - com.example.internal.*
```

### 8. Distributed Tracing Integration

```java
// OpenTelemetry 연동
radar.enableDistributedTracing(openTelemetry);

// 분산 환경에서 예외 추적
// Trace ID, Span ID가 예외 컨텍스트에 포함됨
```

## 설정

> 설계안이다. `src/main/resources/application.yml`은 있지만, 이 설정을 읽는 코드는 아직 없다.

### application.yml

```yaml
exception-radar:
  enabled: true

  # Phase 2: Thread Tracking
  thread:
    track-uncaught: true
    track-virtual-threads: true
    include-patterns:
      - "pool-.*"
      - "ForkJoinPool.*"
    exclude-patterns:
      - ".*-monitor-.*"

  # Phase 4: Process Tracking  
  process:
    enabled: true
    health-check-interval: 5s
    shutdown-timeout: 30s
    ipc-type: unix-socket

  # Phase 2: Async Tracking
  async:
    track-completable-future: true
    propagate-context: true

  # Phase 1: Event Bus
  event-bus:
    async: true
    buffer-size: 1024
    overflow-strategy: drop

  # Phase 1: Collector
  collector:
    max-stack-depth: 50
    max-causation-depth: 10
    collect-metadata: true

  # Phase 3: Reporters
  reporter:
    console:
      enabled: true
    file:
      enabled: true
      path: logs/exceptions.log
      format: json

  # Phase 5: Analyzer
  analyzer:
    enabled: true
    pattern-detection: true
    window-size: 5m
    min-frequency: 3

  # Phase 5: Resilience
  resilience:
    retry:
      enabled: true
      max-attempts: 3
      backoff: 1s
    circuit-breaker:
      enabled: true
      failure-threshold: 5
      reset-timeout: 1m

  # Phase 6: Alert
  alert:
    enabled: true
    channels:
      slack:
        enabled: false
        webhook-url: ${SLACK_WEBHOOK_URL:}
      email:
        enabled: false
        smtp-host: ${SMTP_HOST:localhost}

  # Phase 6: Dashboard
  dashboard:
    enabled: true
    port: 8080
    websocket: true

  # Phase 8: Distributed Tracing
  tracing:
    enabled: false
    exporter: otlp

  # Phase 3: Metrics
  metrics:
    enabled: true
    prefix: exception.radar

  # Filter
  filter:
    ignore-exceptions:
      - java.lang.InterruptedException
      - java.util.concurrent.CancellationException
    ignore-patterns:
      - ".*Connection reset.*"

  debug:
    enabled: false
    log-internal-events: false
```

## 프로젝트 구조

> 설계안이다. 지금 있는 것은 `build.gradle` · `settings.gradle` · `src/main/resources/`의 설정 파일뿐이다. `src/main/java/` 아래 클래스(`Main.java` 포함), `src/test/`, `dashboard/` 정적 파일은 아직 없다.

```
ExceptionRadar/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/exceptionradar/
│   │   │       │
│   │   │       │  # ===== Phase 1: Foundation =====
│   │   │       ├── ExceptionRadar.java
│   │   │       ├── ExceptionRadarConfig.java
│   │   │       ├── core/
│   │   │       │   ├── ExceptionContext.java
│   │   │       │   ├── ExceptionEvent.java
│   │   │       │   └── CausationChain.java
│   │   │       ├── event/
│   │   │       │   ├── EventBus.java
│   │   │       │   ├── EventDispatcher.java
│   │   │       │   ├── EventSubscriber.java
│   │   │       │   └── RingBuffer.java
│   │   │       ├── handler/
│   │   │       │   └── ExceptionHandler.java
│   │   │       ├── config/
│   │   │       │   └── ConfigLoader.java
│   │   │       │
│   │   │       │  # ===== Phase 2: Thread Tracking =====
│   │   │       ├── thread/
│   │   │       │   ├── ThreadMonitor.java
│   │   │       │   ├── ThreadPoolWrapper.java
│   │   │       │   ├── VirtualThreadMonitor.java
│   │   │       │   └── UncaughtExceptionInterceptor.java
│   │   │       ├── async/
│   │   │       │   ├── AsyncTracker.java
│   │   │       │   ├── CompletableFutureTracker.java
│   │   │       │   └── AsyncContextPropagator.java
│   │   │       ├── context/
│   │   │       │   ├── ContextHolder.java
│   │   │       │   └── ContextPropagator.java
│   │   │       │
│   │   │       │  # ===== Phase 3: Reporting =====
│   │   │       ├── reporter/
│   │   │       │   ├── Reporter.java
│   │   │       │   ├── ConsoleReporter.java
│   │   │       │   ├── FileReporter.java
│   │   │       │   └── MetricsReporter.java
│   │   │       │
│   │   │       │  # ===== Phase 4: Process Tracking =====
│   │   │       ├── process/
│   │   │       │   ├── ProcessMonitor.java
│   │   │       │   ├── ProcessWatcher.java
│   │   │       │   ├── ProcessCommunicator.java
│   │   │       │   ├── ProcessRegistry.java
│   │   │       │   ├── ManagedProcess.java
│   │   │       │   └── ipc/
│   │   │       │       ├── IpcProtocol.java
│   │   │       │       ├── UnixSocketServer.java
│   │   │       │       └── UnixSocketClient.java
│   │   │       │
│   │   │       │  # ===== Phase 5: Analysis & Resilience =====
│   │   │       ├── analyzer/
│   │   │       │   ├── ExceptionAnalyzer.java
│   │   │       │   ├── PatternDetector.java
│   │   │       │   └── HistoricalAnalyzer.java
│   │   │       ├── resilience/
│   │   │       │   ├── RetryMechanism.java
│   │   │       │   ├── RetryConfig.java
│   │   │       │   ├── CircuitBreaker.java
│   │   │       │   └── CircuitBreakerConfig.java
│   │   │       │
│   │   │       │  # ===== Phase 6: Dashboard & Alert =====
│   │   │       ├── dashboard/
│   │   │       │   ├── ExceptionDashboard.java
│   │   │       │   ├── DashboardServer.java
│   │   │       │   ├── WebSocketHandler.java
│   │   │       │   ├── RestApiHandler.java
│   │   │       │   └── StatisticsAggregator.java
│   │   │       ├── alert/
│   │   │       │   ├── AlertManager.java
│   │   │       │   ├── AlertRule.java
│   │   │       │   ├── AlertChannel.java
│   │   │       │   └── channels/
│   │   │       │       ├── SlackChannel.java
│   │   │       │       ├── EmailChannel.java
│   │   │       │       └── WebhookChannel.java
│   │   │       │
│   │   │       │  # ===== Phase 7: OS Level =====
│   │   │       ├── agent/
│   │   │       │   ├── ExceptionRadarAgent.java
│   │   │       │   ├── AgentConfig.java
│   │   │       │   ├── transformer/
│   │   │       │   │   ├── ExecutorTransformer.java
│   │   │       │   │   └── ThreadPoolTransformer.java
│   │   │       │   └── native/
│   │   │       │       ├── SignalHandler.java
│   │   │       │       └── NativeCrashDetector.java
│   │   │       │
│   │   │       │  # ===== Phase 8: Distributed Tracing =====
│   │   │       ├── tracing/
│   │   │       │   ├── TracingIntegration.java
│   │   │       │   ├── TraceContext.java
│   │   │       │   └── TraceContextPropagator.java
│   │   │       │
│   │   │       └── Main.java
│   │   │
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── logback.xml
│   │       └── dashboard/
│   │           ├── index.html
│   │           ├── css/
│   │           └── js/
│   │
│   └── test/
│       └── java/
│           └── com/exceptionradar/
│               ├── core/
│               ├── thread/
│               ├── async/
│               ├── process/
│               ├── analyzer/
│               ├── dashboard/
│               ├── agent/
│               └── integration/
│
├── build.gradle
├── settings.gradle
└── README.md
```

## 빌드 및 실행

> 아직 없음. 소스가 없고 `build.gradle`의 `mainClass`(`com.exceptionradar.Main`)도 없어서 아래 명령은 지금 동작하지 않는다. Gradle wrapper jar(`gradle/wrapper/gradle-wrapper.jar`)도 저장소에 없다.

### 요구사항
- Java 21 이상
- Gradle 8.5 이상

### 빌드
```bash
./gradlew build
```

### 테스트
```bash
./gradlew test
```

### 실행
```bash
./gradlew run
```

### Fat JAR 생성
```bash
./gradlew fatJar
java -jar build/libs/ExceptionRadar-1.0.0-SNAPSHOT-all.jar
```

### Java Agent 빌드 (Phase 7)
```bash
./gradlew agentJar
java -javaagent:build/libs/exception-radar-agent.jar -jar your-app.jar
```

## 예제 코드

> 설계안이다. 아래 클래스는 아직 없다.

### 기본 사용법

```java
public class Main {
    public static void main(String[] args) {
        // ExceptionRadar 초기화
        ExceptionRadar radar = ExceptionRadar.builder()
            .enableThreadTracking()
            .enableProcessTracking()
            .enableAsyncTracking()
            .withReporter(new ConsoleReporter())
            .withReporter(new FileReporter("exceptions.log"))
            .withAnalyzer(new PatternDetector())
            .build();

        // 글로벌 레이더 설정
        ExceptionRadar.setGlobal(radar);

        // 대시보드 시작
        radar.startDashboard(8080);

        // 애플리케이션 로직
        try (radar) {
            runApplication();
        }
    }
}
```

### 커스텀 예외 핸들러

```java
public class CustomExceptionHandler implements ExceptionHandler {
    @Override
    public void handle(ExceptionContext context) {
        Throwable e = context.getException();
        Thread thread = context.getThread();
        Instant timestamp = context.getTimestamp();
        CausationChain chain = context.getCausationChain();

        // 커스텀 처리 로직
        if (e instanceof CriticalException) {
            alertOpsTeam(e, thread, timestamp);
        }

        // 인과 관계 로깅
        chain.forEach(cause -> 
            log.debug("Caused by: {}", cause)
        );
    }
}
```

### 프로세스 간 통신

```java
// Parent Process
public class ParentProcess {
    public static void main(String[] args) {
        ExceptionRadar radar = ExceptionRadar.builder()
            .enableProcessTracking()
            .build();

        ProcessMonitor monitor = radar.createProcessMonitor();

        ManagedProcess child = monitor.spawn(
            "java", "-cp", "child.jar", "ChildProcess"
        );

        monitor.onException((processId, exception) -> {
            System.err.printf("Process %d failed: %s%n",
                processId, exception.getMessage());
        });

        // 활성 프로세스 모니터링
        ProcessRegistry registry = monitor.getRegistry();
        registry.onProcessExit((pid, exitCode) -> {
            log.info("Process {} exited with code {}", pid, exitCode);
        });

        monitor.waitForAll();
    }
}

// Child Process
public class ChildProcess {
    public static void main(String[] args) {
        ExceptionRadar.enableIPCReporting();

        // 이 예외는 부모 프로세스로 전달됨
        throw new RuntimeException("Child process error");
    }
}
```

## 라이센스

MIT License