# ExceptionRadar

> Cross-Process/Thread Exception Tracking System for Java Applications

## Overview

ExceptionRadar는 Java 애플리케이션에서 프로세스 및 스레드 경계를 넘어 예외를 추적하고 감시하는 시스템입니다. 특히 async/await 패턴과 멀티스레드 환경에서 발생하는 예외를 OS 레벨에서 감지하고 추적할 수 있는 메커니즘을 제공합니다.

### 핵심 문제 정의

```java
// 기존 문제점
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
3. **OS 레벨 모니터링**: JVM을 넘어선 시스템 레벨의 예외 감지
4. **실시간 예외 리포팅**: 예외 발생 즉시 중앙 모니터링 시스템으로 전달
5. **예외 컨텍스트 보존**: 예외가 발생한 전체 컨텍스트와 스택 트레이스 유지

## 아키텍처

### 시스템 구성도

```
┌─────────────────────────────────────────────────────────┐
│                    ExceptionRadar Core                   │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │   Thread     │  │   Process    │  │   Exception  │ │
│  │   Monitor    │  │   Monitor    │  │   Collector  │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘ │
│         │                  │                  │         │
│  ┌──────▼──────────────────▼──────────────────▼──────┐ │
│  │            Exception Event Bus                     │ │
│  └──────┬──────────────────┬──────────────────┬──────┘ │
│         │                  │                  │         │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼──────┐ │
│  │   Reporter   │  │   Analyzer   │  │   Storage   │ │
│  └──────────────┘  └──────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### 핵심 컴포넌트

#### 1. Thread Monitor
- **ThreadPoolMonitor**: ExecutorService 기반 스레드풀 감시
- **VirtualThreadMonitor**: Java 21 Virtual Thread 감시
- **UncaughtExceptionInterceptor**: 캐치되지 않은 예외 가로채기

#### 2. Process Monitor
- **ProcessWatcher**: 자식 프로세스 생성 및 감시
- **ProcessCommunicator**: IPC를 통한 예외 정보 전달
- **ProcessRegistry**: 활성 프로세스 추적 및 관리

#### 3. Exception Collector
- **ExceptionContext**: 예외 발생 컨텍스트 수집
- **StackTraceAnalyzer**: 스택 트레이스 분석 및 정리
- **ExceptionChain**: 예외 체인 추적 및 관리

#### 4. Event Bus
- **ExceptionEvent**: 예외 이벤트 모델
- **EventDispatcher**: 이벤트 라우팅 및 배포
- **EventSubscriber**: 이벤트 구독 및 처리

#### 5. Reporting & Analysis
- **RealTimeReporter**: 실시간 예외 리포팅
- **ExceptionAnalyzer**: 예외 패턴 분석
- **MetricsCollector**: 예외 관련 메트릭 수집

## 기술 스택

- **Java 21**: Virtual Threads, Pattern Matching, Record Classes
- **Concurrency**: CompletableFuture, ExecutorService, ForkJoinPool
- **IPC**: Unix Domain Sockets, Named Pipes, Shared Memory
- **Serialization**: Jackson (JSON), Protocol Buffers
- **Monitoring**: Micrometer, JMX
- **Logging**: SLF4J + Logback

## 주요 기능

### 1. Thread Exception Tracking

```java
// ExceptionRadar가 자동으로 추적
ExceptionRadar radar = ExceptionRadar.getInstance();
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

## 구현 계획

### Phase 1: Core Foundation
- [ ] Basic thread exception tracking
- [ ] UncaughtExceptionHandler 구현
- [ ] ThreadLocal exception context
- [ ] Simple event bus

### Phase 2: Thread Pool Integration
- [ ] ExecutorService wrapper
- [ ] ForkJoinPool integration
- [ ] Virtual Thread support (Java 21)
- [ ] Custom ThreadFactory

### Phase 3: Process Monitoring
- [ ] Process spawn wrapper
- [ ] IPC mechanism (Unix Domain Socket)
- [ ] Process registry
- [ ] Parent-child communication protocol

### Phase 4: Advanced Features
- [ ] Exception pattern recognition
- [ ] Automatic retry mechanism
- [ ] Circuit breaker integration
- [ ] Distributed tracing support

### Phase 5: Reporting & Analytics
- [ ] Real-time dashboard
- [ ] Metrics collection
- [ ] Alert system
- [ ] Historical analysis

## 사용 사례

### 1. Microservice Architecture
- 서비스 간 예외 전파 추적
- 분산 트랜잭션 실패 원인 분석
- 서비스 메시 내 예외 플로우 시각화

### 2. Batch Processing
- 병렬 배치 작업 예외 모니터링
- 실패한 작업 자동 재시도
- 예외 패턴 기반 작업 최적화

### 3. Real-time Systems
- 실시간 예외 알림
- 임계 예외 자동 에스컬레이션
- 예외 발생 예측 및 예방

## 예제 코드

### 기본 사용법

```java
public class Main {
    public static void main(String[] args) {
        // ExceptionRadar 초기화
        ExceptionRadar radar = ExceptionRadar.builder()
            .enableThreadTracking()
            .enableProcessTracking()
            .withReporter(new ConsoleReporter())
            .withReporter(new FileReporter("exceptions.log"))
            .build();
        
        // 글로벌 레이더 설정
        ExceptionRadar.setGlobal(radar);
        
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
        Exception e = context.getException();
        Thread thread = context.getThread();
        Instant timestamp = context.getTimestamp();
        
        // 커스텀 처리 로직
        if (e instanceof CriticalException) {
            alertOpsTeam(e, thread, timestamp);
        }
        
        // 메트릭 수집
        MetricsCollector.record(e.getClass(), thread.getName());
    }
}
```

### 프로세스 간 통신

```java
// Parent Process
public class ParentProcess {
    public static void main(String[] args) {
        ProcessMonitor monitor = new ProcessMonitor();
        
        Process child = monitor.spawn(
            "java", "-cp", "child.jar", "ChildProcess"
        );
        
        monitor.onException((processId, exception) -> {
            System.err.printf("Process %d failed: %s%n", 
                processId, exception.getMessage());
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

## 빌드 및 실행

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

### 디버그 모드
```bash
./gradlew debugRun
```

## 프로젝트 구조

```
ExceptionRadar/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/exceptionradar/
│   │   │       ├── core/
│   │   │       │   ├── ExceptionRadar.java
│   │   │       │   ├── ExceptionContext.java
│   │   │       │   └── ExceptionHandler.java
│   │   │       ├── thread/
│   │   │       │   ├── ThreadMonitor.java
│   │   │       │   ├── ThreadPoolWrapper.java
│   │   │       │   └── VirtualThreadMonitor.java
│   │   │       ├── process/
│   │   │       │   ├── ProcessMonitor.java
│   │   │       │   ├── ProcessCommunicator.java
│   │   │       │   └── ProcessRegistry.java
│   │   │       ├── event/
│   │   │       │   ├── EventBus.java
│   │   │       │   ├── ExceptionEvent.java
│   │   │       │   └── EventSubscriber.java
│   │   │       ├── reporter/
│   │   │       │   ├── Reporter.java
│   │   │       │   ├── ConsoleReporter.java
│   │   │       │   └── FileReporter.java
│   │   │       └── Main.java
│   │   └── resources/
│   │       ├── logback.xml
│   │       └── 

│   └── test/
│       └── java/
│           └── com/exceptionradar/
│               ├── core/
│               ├── thread/
│               └── process/
├── build.gradle
├── settings.gradle
└── README.md
```

## 향후 계획

1. **Kubernetes Integration**: K8s 환경에서 Pod 간 예외 추적
2. **OpenTelemetry Support**: 분산 추적 표준 지원
3. **Machine Learning**: 예외 패턴 학습 및 예측
4. **Auto-healing**: 자동 복구 메커니즘
5. **IDE Plugin**: IntelliJ IDEA, VS Code 플러그인

## 라이센스

MIT License

