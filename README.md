## Inner Architecture 팀의 역할

### 요구사항

- Microservices Architecture (MSA)에서는 각 서비스가 독립적으로 동작하고, 서로 간에 직접 호출 없이 데이터를 주고받아야 할 때가 많다.
- 이때, 서비스 간의 연결을 느슨하게 유지하면서도 실시간 통신이 가능해야 한다.

### 역할

- Inner Architecture 팀은 Kafka를 **중앙 이벤트 버스**로 활용해 해당 요구사항을 구현한다.
    1. Kafka 클러스터를 구축한다.
    2. 이벤트 흐름을 설계한다.
    3. 서비스 간 Decoupling을 유지한다.
    4. DLQ(Dead Letter Queue)를 구성하고 장애에 대응한다.
    5. 모니터링을 통해 메시지 흐름을 추적하고 운영을 유지한다.

## Kafka란?

Apache Kafka는 마이크로서비스 간의 통신을 **보다 확장 가능하고 안정적**으로 만들어주는 강력한 **분산 스트리밍 플랫폼**입니다.

Kafka는 MSA에서 서비스 간 메시지를 주고받는 핵심 역할을 수행하며, **비동기 통신**, **느슨한 결합**, **확장성 높은 처리 구조**를 제공합니다.

---

## Kafka의 기본 구성요소

| 구성 요소 | 설명 |
| --- | --- |
| **Producer** | 데이터를 Kafka로 보내는 주체 (예: 주문 생성 서비스) |
| **Consumer** | Kafka에서 데이터를 받아 처리하는 주체 (예: 이메일 서비스) |
| **Broker** | Kafka 서버 인스턴스. 메시지를 저장하고 배포함 |
| **Topic** | 메시지를 분류하는 논리적 단위 (하나의 이벤트 스트림) |
| **Partition** | 토픽을 수평으로 분할하여 병렬 처리 가능하게 함 |
| **Consumer Group** | 여러 Consumer가 같은 토픽을 병렬로 처리할 수 있게 함 |

---

## Kafka의 동작 방식

1. `Producer`가 `Broker`에 특정 `Topic`의 메시지를 전송
2. `Broker`는 메시지를 `Topic`별로 나눠 저장
3. `Consumer`는 지정된 `Topic`의 메시지를 **순차적으로 소비**
    
    → 구독 형식으로, 구독한 이벤트에 대해서만 소비
    
4. `Consumer`는 Offset을 기준으로 소비된 메시지 위치를 관리 (다시 읽을 수도 있음)

---

## Kafka의 주요 장점

### 1. **확장성과 내결함성**

- 같은 토픽의 메시지를 여러 파티션에 나눠 저장 → 병렬 처리 가능
- 브로커 클러스터 구성으로 장애 발생 시 복구 가능

### 2. **비동기 통신**

- 마이크로서비스 간 느슨한 결합 구현 가능
- 메시지를 일단 Kafka에 저장하고 나중에 처리 가능

### 3. **고성능 처리**

- 초당 수십만 ~ 수백만 건의 메시지를 처리 가능
- 디스크 기반 저장이지만 매우 빠른 처리 성능 제공

### 4. **내구성 & 재처리 가능**

- 메시지가 디스크에 저장되므로, Consumer가 죽어도 재시도 가능 → 카프카 브로커가 실행중인 PC에 저장
- Offset 기반으로 메시지 재처리, 재소비가 쉬움 → 각 메시지가 인덱스를 가짐

### 5. **언어 독립적**

- Java, Python, Go 등 다양한 언어 지원 (다양한 팀과 연동 용이)

### 6. 서버 없이도 동작

- Consumer와 Producer는 서버 환경이 아닐 때도 동작 가능
    
    → AWS Lambda 등 서버리스 아키텍처에서도 사용 가능
    

---

## Kafka의 단점 및 고려사항

| 항목 | 설명 |
| --- | --- |
| **순서 보장 한계** | 같은 파티션 안에서만 순서 보장 |
| **실시간 처리에는 부적합** | 몇 밀리초 지연이 치명적인 경우 (ex. 티켓팅 등)엔 부적절 |
| **운영 복잡도** | Zookeeper, 파티션 수, 브로커 관리 등 복잡함 |
| **메시지 삭제 정책 고려 필요** | 메시지 보관 기간 설정([retention.ms](http://retention.ms/))을 주의해야 함 |

---

## Kafka vs 다른 메시지 시스템

| 항목 | Kafka | Redis Streams | RabbitMQ |
| --- | --- | --- | --- |
| 처리 방식 | Pull 기반 스트리밍 | In-memory 스트림 큐 | Push 기반 큐  |
| 순서 보장 | 파티션 내 보장 | 보장 가능 (XREADGROUP) | 기본 보장 |
| 성능 | 높음 | 매우 높음 (메모리 기반) | 낮음 ~ 보통 |
| 사용 목적 | 대용량 이벤트 처리 | 실시간 처리 및 TTL 큐 | 단순 작업 큐 |

### 처리 방식 비교

- **Kafka**
    - Consumer가 주기적으로 메시지를 요청해서 직접 가져가는 구조
        
        → 브로커는 Producer로부터 메시지를 받아 저장하고, Consumer의 요청이 들어와야만 전달
        
    - Consumer가 속도 조절(Poll 주기), 메시지 중복, 재시도, 재처리(Offset 관리) 등이 쉬움
- **Redis Streams**
    - 메모리 기반이기 때문에 빠르지만, 대용량 처리에 부적합
    - In-memory 기반으로 Producer가 메모리에 쓰고 Consumer가 읽음 (브로커 없음)
- **RabbitMQ**
    - 메시지가 도착하면 브로커가 즉시 등록된 Consumer에게 전달
    - 구현이 단순하지만, Consumer가 느리면 오버플로우 발생 가능

---

## 도입 시 체크리스트

- [ ]  **메시지 포맷 통일** : JSON / Avro 등 표준화 필요 → 적어도 동일한 이벤트에는 동일한 포맷
- [ ]  **토픽 및 파티션 설계** : 이벤트 성격에 맞게 분리
- [ ]  **Consumer 그룹 설정** : 부하 분산 or 개별 소비 전략 선택
- [ ]  **메시지 유실 대비** : 에러 처리, 리트라이, DLQ(Dead Letter Queue) 구조 고려
- [x]  Kafka 운영/모니터링 체계 필요 (Kafka UI, Grafana 등)

---

## Kafka에서의 파티션 동작 원리

### 파티션이란?

- Kafka 토픽은 여러 **파티션(partition)**으로 나뉘며, 각 파티션은 메시지를 순서대로 저장하는 **로그 구조**
    
    → 한 파티션에는 반드시 하나의 토픽만 위치
    
- 파티션을 통해 **병렬 처리**, **확장성**, **순서 제어**가 가능함

---

### 순서 보장의 전제조건

| 조건 | 순서 보장 여부 |
| --- | --- |
| 같은 파티션, 하나의 Producer | ✅ 순서 보장됨 |
| 같은 파티션, 여러 Producer | ❌ 순서 보장 안 됨 (네트워크/스레드 경쟁 등) |
| 다른 파티션 | ❌ 순서 보장 불가 |

→ 여러 프로듀서가 같은 파티션에 여러 메시지를 보낼 경우 순서가 보장되지 않음.

---

### 파티션 지정 방법

### 1. **Key 기반 자동 파티셔닝**

```java
kafkaTemplate.send("topic", "my-key", value); // 동일 key → 동일 파티션
```

### 2. 파티션 수동 지정

```java
ProducerRecord<String, String> record = new ProducerRecord<>("topic", 0, null, value);
kafkaTemplate.send(record);
```

→ 0번 파티션으로 수동 지정

---

## 예시 시나리오 (MSA 기반) → Demo 진행

- [x]  1. 주문 서비스가 주문 이벤트 발행 → 브로커로 전달 → 재고 서비스가 이벤트 처리 (재고 차감)
- [x]  2. 부하 테스트 (메시지가 늘어날 때 처리 시간 비교)
- [x]  3. 도착 순서 보장 테스트 (서로 다른 프로듀서가 같은 파티션으로 이벤트를 발행할 때)
- [x]  4. Lambda를 활용한 서버리스 아키텍처 구현 테스트
- [x]  5. kafka UI 데모

---

## 각 서비스팀에서 설정해야하는것

스프링의 경우

```java
spring.kafka.bootstrap-servers: 13.125.55.123:9092
```

Kafka가 실행 중인 **EC2의 퍼블릭 IP 또는 도메인을 사용**

**Docker로 구성 시 KAFKA_ADVERTISED_LISTENERS 환경변수로 설정 가능**

```java
environment:
  KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://13.125.55.123:9092
  KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
```

## 토픽 네이밍

### 기본 구조

```

<도메인>.<서비스명>.<이벤트명 or 용도>
```

### 예시 규칙

| 규칙 요소 | 예시 | 설명 |
| --- | --- | --- |
| 도메인 | `order`, `payment`, `user`, `inventory` | 시스템의 bounded context |
| 서비스명 | `order-service`, `payment-api` 등 | 토픽을 발행하는 서비스 |
| 이벤트명 or 목적 | `created`, `updated`, `logs`, `events`, `result` | 의미를 드러내는 키워드 |

토픽 네이밍 규칙을 만들어 제공 예정

### 토픽 생성 요청

내부 지라 티켓이나 슬랙 채널 사용 예정

---

## 참고 자료

> https://kafka.apache.org/
> 

> https://www.confluent.io/
> 

> https://www.confluent.io/solutions/microservices/
>
