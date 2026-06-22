# DR 전환 테스트 및 체크리스트

## 1. 개요

본 문서는 Main 서버 장애 상황을 가정하여 DR 서버로 서비스가 정상 전환되는지 검증한 내용을 정리합니다.

DR 전환 테스트는 단순히 Replica VM을 실행하는 것뿐만 아니라, DB Master 승격, ProxySQL 접속 경로 전환, 원장시스템/체결 엔진 VM Failover, 네트워크 전환, Kafka 통신, 애플리케이션 정상 동작 여부까지 확인하는 것을 목표로 합니다.

<br />

## 2. 테스트 목적

| 목적             | 설명                                                          |
| -------------- | ----------------------------------------------------------- |
| VM DR 검증       | Main 서버의 원장시스템/체결 엔진 VM 장애 시 DR Replica VM이 정상 기동되는지 확인합니다. |
| DB Failover 검증 | Main Master DB 장애 시 DR DB가 Master로 승격되는지 확인합니다.             |
| 단일 엔드포인트 검증    | ProxySQL 단일 엔드포인트를 통해 애플리케이션 DB 접속이 유지되는지 확인합니다.            |
| 네트워크 전환 검증     | DR VM이 DR 대역 IP, Gateway, DNS로 정상 전환되는지 확인합니다.              |
| 서비스 복구 검증      | 원장시스템, 체결 엔진, Kafka, DB 연동이 정상 동작하는지 확인합니다.                 |

<br />

## 3. 테스트 범위

본 테스트에서 확인한 범위는 다음과 같습니다.

| 구분          | 테스트 항목                                       |
| ----------- | -------------------------------------------- |
| Veeam DR    | Replica VM Failover, VM 기동 상태 확인             |
| Network     | DR IP 적용, Gateway 통신, 포트 통신 확인               |
| DB          | DR DB Master 승격, `read_only` 해제, 쓰기 가능 여부 확인 |
| ProxySQL    | 단일 엔드포인트 접속, Write Hostgroup 라우팅 확인          |
| Kafka       | DR Kafka Broker 접속, Topic 조회, 이벤트 흐름 확인      |
| Application | 원장시스템/체결 엔진 Health Check 및 주요 API 확인         |

<br />

## 4. 테스트 전 확인 사항

DR 전환 테스트 전 다음 항목을 확인합니다.

| 확인 항목            | 확인 내용                                             |
| ---------------- | ------------------------------------------------- |
| Veeam Replica 상태 | 원장시스템/체결 엔진 VM Replica가 `Ready` 상태인지 확인합니다.       |
| VMware Tools 상태  | Replica VM 내부에서 VMware Tools가 정상 동작하는지 확인합니다.     |
| DR IP 계획         | Failover 후 적용할 IP, Gateway, DNS, Hostname을 확인합니다. |
| DB 복제 상태         | Main DB와 DR DB 간 복제가 정상인지 확인합니다.                  |
| Orchestrator 상태  | DR DB가 승격 대상에 포함되어 있는지 확인합니다.                     |
| ProxySQL 상태      | DR DB가 ProxySQL 서버 목록에 등록되어 있는지 확인합니다.            |
| Kafka 상태         | DR Kafka Broker가 정상 기동되어 있는지 확인합니다.               |

<br />

## 5. 테스트 시나리오

### 5.1 Main Master DB 장애 테스트

Main Master DB 장애 상황을 가정하여 DR DB가 Master로 승격되는지 확인합니다.

#### 테스트 절차

1. Main Master DB 중지 또는 장애 상황을 발생시킵니다.
2. Orchestrator가 Master 장애를 감지하는지 확인합니다.
3. DR DB가 새로운 Master로 승격되는지 확인합니다.
4. ProxySQL이 Write 트래픽을 새로운 Master로 라우팅하는지 확인합니다.
5. 애플리케이션이 기존 ProxySQL 엔드포인트로 DB에 정상 접속하는지 확인합니다.

#### 확인 명령어

```sql
SELECT @@hostname, @@read_only;
```

`@@read_only` 값이 `0`이면 쓰기 가능한 Master DB입니다.

```sql
SELECT hostgroup_id, hostname, port, status
FROM runtime_mysql_servers;
```

ProxySQL Runtime 서버 목록에서 새로운 Master가 Write Hostgroup에 포함되어 있는지 확인합니다.

<br />

### 5.2 원장시스템/체결 엔진 VM Failover 테스트

Main 서버의 원장시스템 VM과 체결 엔진 VM 장애 상황을 가정하여 DR Replica VM을 실행합니다.

#### 테스트 절차

1. Veeam에서 원장시스템/체결 엔진 Replica VM의 `Failover Now`를 실행합니다.
2. DR ESXi에서 Replica VM이 정상 기동되는지 확인합니다.
3. Replica VM 내부 네트워크 설정을 DR 환경에 맞게 변경합니다.
4. DR IP, Gateway, DNS, Hostname이 정상 적용되었는지 확인합니다.
5. 애플리케이션 프로세스가 정상 기동되는지 확인합니다.

#### 확인 명령어

```bash
ip addr
ip route
hostname
```

```bash
ping <DR_GATEWAY_IP>
curl http://<DR_APP_IP>:<PORT>/actuator/health
```

<br />

### 5.3 Kafka DR 통신 테스트

DR 환경에서도 주문·체결 이벤트 흐름을 처리할 수 있는지 확인합니다.

#### 테스트 절차

1. DR Kafka Broker가 정상 실행 중인지 확인합니다.
2. DR Kafka Topic 목록을 조회합니다.
3. 원장시스템/체결 엔진에서 Kafka Broker로 접속 가능한지 확인합니다.
4. 테스트 이벤트를 발행하거나 기존 이벤트 흐름이 정상 동작하는지 확인합니다.

#### 확인 명령어

```bash
nc -vz <DR_KAFKA_IP> 9092
```

```bash
kafka-topics.sh --bootstrap-server <DR_KAFKA_IP>:9092 --list
```

<br />

### 5.4 서비스 정상 동작 테스트

DR 전환 이후 주요 서비스가 정상 동작하는지 확인합니다.

| 테스트 항목       | 확인 내용                                |
| ------------ | ------------------------------------ |
| Health Check | 원장시스템/체결 엔진 `/actuator/health` 정상 응답 |
| DB 접속        | ProxySQL 단일 엔드포인트를 통한 DB 접속 가능       |
| 쓰기 테스트       | DR DB Master에 INSERT/UPDATE 가능       |
| Kafka 통신     | Kafka Broker 접속 및 Topic 조회 가능        |
| 주문 흐름        | 주문 요청 또는 테스트 이벤트 처리 가능               |
| 로그 확인        | 애플리케이션 에러 로그 발생 여부 확인                |

<br />

## 6. DR 전환 체크리스트

| 단계 | 확인 항목                          | 결과 |
| -- | ------------------------------ | -- |
| 1  | Veeam Replica VM Ready 상태 확인   | ☐  |
| 2  | Main Master DB 장애 감지 확인        | ☐  |
| 3  | DR DB Master 승격 확인             | ☐  |
| 4  | ProxySQL Write Hostgroup 전환 확인 | ☐  |
| 5  | 원장시스템 Replica VM 기동 확인         | ☐  |
| 6  | 체결 엔진 Replica VM 기동 확인         | ☐  |
| 7  | DR IP/Gateway/DNS 적용 확인        | ☐  |
| 8  | DR DB 접속 확인                    | ☐  |
| 9  | DR Kafka 접속 확인                 | ☐  |
| 10 | 원장시스템 Health Check 확인          | ☐  |
| 11 | 체결 엔진 Health Check 확인          | ☐  |
| 12 | 주문/체결 흐름 테스트 확인                | ☐  |
| 13 | Grafana 또는 로그에서 장애/복구 상태 확인    | ☐  |

<br />

## 7. Rollback / Undo Failover

DR 전환 테스트 이후에는 테스트 목적에 따라 Rollback 또는 Undo Failover를 수행합니다.

| 항목                  | 설명                                                 |
| ------------------- | -------------------------------------------------- |
| Veeam Undo Failover | 테스트용으로 실행한 Replica VM을 종료하고 기존 상태로 되돌립니다.          |
| DB 복구               | 장애 처리한 Main DB를 복구하고 현재 Master 기준으로 복제 구성을 재정리합니다. |
| ProxySQL 확인         | Write Hostgroup이 현재 Master를 정확히 바라보는지 확인합니다.       |
| 서비스 확인              | 애플리케이션이 정상 DB 엔드포인트로 접속하는지 확인합니다.                  |

> 실제 장애 상황에서는 바로 기존 Main DB로 되돌리지 않고, 데이터 정합성과 복제 상태를 먼저 확인한 뒤 계획된 절차에 따라 전환합니다.

<br />

## 8. 테스트 결과 정리

| 테스트 항목              | 결과      | 비고 |
| ------------------- | ------- | -- |
| Veeam Replica VM 기동 | 성공 / 실패 |    |
| DR VM 네트워크 전환       | 성공 / 실패 |    |
| DR DB Master 승격     | 성공 / 실패 |    |
| ProxySQL 접속 경로 전환   | 성공 / 실패 |    |
| Kafka 통신 확인         | 성공 / 실패 |    |
| 애플리케이션 Health Check | 성공 / 실패 |    |
| 주문/체결 흐름 확인         | 성공 / 실패 |    |

<br />

## 9. 증적 자료

테스트 결과를 확인할 수 있는 캡처 또는 로그를 첨부합니다.

| 증적                       | 설명                    |
| ------------------------ | --------------------- |
| Veeam Replica Ready 화면   | Replica VM 상태 확인      |
| Failover Now 실행 화면       | VM Failover 수행 확인     |
| Orchestrator Topology 화면 | DR DB Master 승격 확인    |
| ProxySQL Hostgroup 조회 결과 | Write Hostgroup 전환 확인 |
| DB `@@read_only` 조회 결과   | DR DB 쓰기 가능 여부 확인     |
| Kafka Topic 조회 결과        | DR Kafka 통신 확인        |
| 애플리케이션 Health Check 결과   | 서비스 정상 기동 확인          |

<br />

## 10. 정리

DR 전환 테스트를 통해 Main 서버 장애 상황에서 DR 서버의 Replica VM, DR DB, ProxySQL, Kafka가 정상적으로 동작하는지 확인했습니다.

특히 DR DB가 Orchestrator의 승격 대상에 포함되어 Main Master 장애 시 새로운 Master로 승격될 수 있음을 확인했고, 애플리케이션은 ProxySQL 단일 엔드포인트를 통해 DB 접속 경로를 유지할 수 있음을 검증했습니다.

또한 Veeam Replica VM 기동 이후 DR 네트워크 설정을 적용하고, 원장시스템과 체결 엔진의 Health Check 및 Kafka 통신을 확인하여 DR 환경에서 핵심 서비스 복구가 가능함을 검증했습니다.
