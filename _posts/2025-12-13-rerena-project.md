---

layout: post
title: "Rerena Consumer"
date: 2025-12-13
excerpt: "Hot-swappable Java Consumer architecture with Redis, RabbitMQ, and NATS"
tag:

* java
* messaging
* architecture
* mermaid
* jekyll
  comments: true

---

# 📝 메시징 브로커 핫 스와핑을 위한 자바 Consumer 아키텍처 분석

## Rerena 프로젝트

---

## 1. 개요 및 프로젝트 목표 (Introduction)

**Rerena 프로젝트**는 다양한 메시징 브로커(**Redis, RabbitMQ, NATS**)를 유연하게 지원하는 자바 Consumer 아키텍처입니다.

핵심 목표는 설정 파일(`config.properties`) 변경만으로 애플리케이션 재시작 없이 **메시징 브로커를 동적으로 전환(Hot Reloading)** 하는 것입니다.

본 글에서는 다음 두 관점에서 구조를 분석합니다.

* **정적 구조**: 클래스 및 책임 분리
* **동적 동작**: 설정 변경 시 Consumer 전환 흐름

모든 설명은 **Mermaid 다이어그램**을 기반으로 합니다.

---

## 2. 정적 구조 분석: 클래스 다이어그램

프로젝트는 **Interface → Abstract Class → Concrete Implementation** 구조를 사용하며, 이는 전형적인 **전략 패턴(Strategy Pattern)** 구현입니다.

### 2.1 클래스 다이어그램

<div class="mermaid">
classDiagram
    direction LR

```
%% Core Abstraction
class MessageConsumer {
    «interface»
    +connect()
    +consumeMessages()
    +close()
}

class AbstractConsumer {
    «abstract»
    #String host
    #int port
    #String queue
    +connect()
}

class BrokerType {
    «enumeration»
    REDIS
    RABBITMQ
    NATS
}

%% Implementations
class RedisConsumer {
    +consumeMessages()
}

class RabbitMQConsumer {
    +consumeMessages()
}

class NatsConsumer {
    +consumeMessages()
}

%% Config
class ConfigLoader {
    +static load()
    +static get(String key)
    +static watch(Runnable onChange)
}

%% Service
class Rerenaconsumer {
    -MessageConsumer consumer
    -ExecutorService executor
    +start()
    -startConsumer()
    -restartConsumer()
}

MessageConsumer <|.. AbstractConsumer
AbstractConsumer <|-- RedisConsumer
AbstractConsumer <|-- RabbitMQConsumer
AbstractConsumer <|-- NatsConsumer

Rerenaconsumer --> MessageConsumer
Rerenaconsumer ..> ConfigLoader
Rerenaconsumer ..> BrokerType
```

</div>

---

## 3. 동적 동작 분석: 시퀀스 다이어그램

아래는 설정 파일 변경 시 **Consumer가 안전하게 교체되는 전체 흐름**입니다.

### 3.1 핫 리로딩 시퀀스

<div class="mermaid">
sequenceDiagram
    autonumber
    participant OS as OS / config.properties
    participant App as Rerenaconsumer
    participant Config as ConfigLoader
    participant Exec as ExecutorService
    participant Consumer as Current Consumer

```
App->>Config: load()
App->>Config: watch(restartConsumer)

App->>App: startConsumer()
App->>Config: get("use")
App->>Exec: submit(consumeMessages)

OS-->>Config: 파일 변경 감지
Config->>Config: load()
Config->>App: restartConsumer()

App->>Consumer: close()
App->>Exec: shutdownNow()

App->>Exec: new Executor
App->>Config: get("use")
App->>Exec: submit(new consumeMessages)
```

</div>

---

## 4. 사용한 오픈소스 및 라이브러리 분석

### 4.1 Java Standard Library (JDK)

#### java.util.concurrent (ExecutorService)

* **역할**: Consumer 메시지 수신을 비동기로 실행
* **핵심 메서드**:

  * `submit(Runnable)` : 메시지 소비 실행
  * `shutdownNow()` : 핫 리로딩 시 즉시 중단
  * `Executors.newSingleThreadExecutor()` : Consumer 단일 스레드 보장

#### java.nio.file.WatchService

* **역할**: `config.properties` 파일 변경 감지
* **핵심 메서드**:

  * `newWatchService()` : 감시 서비스 생성
  * `register()` : 디렉토리 이벤트 등록
  * `pollEvents()` : 변경 이벤트 수신

#### java.util.Properties

* **역할**: 설정 파일 관리
* **핵심 메서드**:

  * `load(InputStream)`
  * `getProperty(String key)`

---

## 5. 주요 클래스 및 함수 단위 기능 설명

### 5.1 MessageConsumer (Interface)

```java
public interface MessageConsumer {
    void connect();
    void consumeMessages();
    void close();
}
```

* `connect()` : 브로커 연결 초기화
* `consumeMessages()` : 메시지 지속 수신
* `close()` : 리소스 정리 (핫 리로딩 필수)

---

### 5.2 AbstractConsumer

* 공통 설정 로딩 (host, port, queue)
* 중복 코드 제거용 템플릿 클래스

---

### 5.3 RedisConsumer / RabbitMQConsumer / NatsConsumer

* 브로커별 메시지 소비 구현
* 브로커 교체 시 코드 수정 불필요

---

### 5.4 ConfigLoader

* 설정 관리의 단일 진입점

주요 메서드:

* `load()` : 설정 로드
* `get(String key)` : 설정 조회
* `watch(Runnable)` : 변경 감지 및 콜백

---

### 5.5 Rerenaconsumer

* 애플리케이션 전체 생명주기 관리

주요 메서드:

* `start()`
* `startConsumer()`
* `restartConsumer()`

---

## 6. 결론 및 확장성

* 브로커 독립 구조
* 무중단 핫 스와핑 지원
* Kafka / AWS SQS 확장 용이

실무 환경에서 **안정성·확장성·유지보수성**을 모두 만족하는 Consumer 아키텍처입니다.
