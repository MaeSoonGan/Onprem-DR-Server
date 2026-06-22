# Orchestrator·ProxySQL 기반 DR DB Failover

## 1. 개요

본 문서는 Main DB 장애 발생 시 DR DB를 새로운 Master로 승격하고, ProxySQL 단일 엔드포인트를 통해 애플리케이션의 DB 접속 경로를 전환하는 구조를 설명합니다.

MaeSoonGan 프로젝트에서는 Main 서버의 Master DB, Slave DB와 DR 서버의 DB를 하나의 장애 조치 대상으로 구성했습니다. 이를 통해 Main Master DB에 장애가 발생하더라도 Orchestrator가 장애를 감지하고, 승격 가능한 DB 중 하나를 새로운 Master로 승격할 수 있도록 구성했습니다.

애플리케이션은 DB에 직접 접속하지 않고 ProxySQL 단일 엔드포인트를 사용합니다. 따라서 Master DB가 Main 서버에서 DR 서버로 변경되더라도 애플리케이션의 접속 정보는 변경하지 않고, ProxySQL이 내부적으로 새로운 Master로 라우팅하도록 구성했습니다.

<br />

## 2. 구성 목적

| 목적          | 설명                                                        |
| ----------- | --------------------------------------------------------- |
| DB 장애 대응    | Main Master DB 장애 발생 시 Slave 또는 DR DB를 새로운 Master로 승격합니다. |
| 단일 접속 경로 제공 | 애플리케이션은 ProxySQL 단일 엔드포인트로만 DB에 접속합니다.                    |
| 접속 경로 자동 전환 | Master 변경 시 ProxySQL이 Write 트래픽을 새로운 Master로 라우팅합니다.      |
| 데이터 정합성 확보  | MySQL 복제 기반으로 Main DB와 DR DB 간 데이터를 동기화합니다.               |
| 서비스 연속성 확보  | DB 장애 상황에서도 애플리케이션의 DB 접속 경로를 유지합니다.                      |

<br />

## 3. DB 구성

| 구성 요소          | 역할                                             |
| -------------- | ---------------------------------------------- |
| Main Master DB | 평상시 쓰기 처리를 담당하는 Primary DB입니다.                 |
| Main Slave DB  | Main Master DB의 Replica로, 읽기 부하 분산 및 승격 대상입니다. |
| DR DB          | DR 서버에 위치한 DB로, Main 장애 시 Master 승격 대상입니다.     |
| Orchestrator   | DB 토폴로지를 감시하고 Master 장애 시 Failover를 수행합니다.     |
| ProxySQL       | 애플리케이션에 단일 DB 엔드포인트를 제공하고 읽기/쓰기 트래픽을 라우팅합니다.   |

<br />

## 4. DB 토폴로지

평상시 DB 구조는 다음과 같습니다.

```text
Application
    |
    v
ProxySQL Single Endpoint
    |
    +-- Write Hostgroup -> Main Master DB
    |
    +-- Read Hostgroup  -> Main Slave DB / DR DB
```

장애 발생 후 DR DB가 Master로 승격되면 다음과 같은 구조로 전환됩니다.

```text
Application
    |
    v
ProxySQL Single Endpoint
    |
    +-- Write Hostgroup -> DR DB (New Master)
    |
    +-- Read Hostgroup  -> 기존 Slave 또는 복구된 DB
```

<br />

## 5. Orchestrator 구성

Orchestrator는 MySQL 복제 토폴로지를 감시하고, Master 장애 발생 시 승격 가능한 Replica를 새로운 Master로 승격하는 역할을 수행합니다.

본 프로젝트에서는 DR DB도 Orchestrator의 관리 대상에 포함하여, Main 서버 장애 시 DR DB가 새로운 Master로 승격될 수 있도록 구성했습니다.

| 항목          | 내용                                   |
| ----------- | ------------------------------------ |
| 관리 대상       | Main Master DB, Main Slave DB, DR DB |
| 주요 역할       | DB 토폴로지 감시, 장애 감지, Master 승격         |
| Failover 대상 | Main Slave DB 또는 DR DB               |
| 장애 감지 기준    | Master DB 응답 불가, 복제 상태 이상, 헬스체크 실패   |

<br />

## 6. ProxySQL 구성

ProxySQL은 애플리케이션과 DB 사이에 위치하여 단일 DB 엔드포인트를 제공합니다.

애플리케이션은 실제 Master DB IP를 직접 바라보지 않고 ProxySQL로만 접속합니다. 따라서 Master DB가 변경되더라도 애플리케이션 설정을 수정하지 않고 ProxySQL 내부 라우팅만 변경하여 DB 접속을 유지할 수 있습니다.

| 항목                | 내용                                   |
| ----------------- | ------------------------------------ |
| 애플리케이션 접속 포트      | `6033`                               |
| ProxySQL Admin 포트 | `6032`                               |
| Write Hostgroup   | Master DB 연결용 Hostgroup              |
| Read Hostgroup    | Slave/Replica DB 연결용 Hostgroup       |
| 주요 역할             | 읽기/쓰기 분리, 단일 엔드포인트 제공, 장애 시 접속 경로 전환 |

<br />

## 7. 장애 전환 흐름

Main Master DB 장애 발생 시 전환 흐름은 다음과 같습니다.

1. Main Master DB에 장애가 발생합니다.
2. Orchestrator가 Master DB의 장애를 감지합니다.
3. Orchestrator는 복제 상태와 승격 가능 여부를 확인합니다.
4. 승격 가능한 DB 중 DR DB를 새로운 Master로 승격합니다.
5. 기존 Master 기준의 복제 토폴로지를 새로운 Master 기준으로 재구성합니다.
6. ProxySQL은 새로운 Master 정보를 반영합니다.
7. 애플리케이션은 기존과 동일하게 ProxySQL 단일 엔드포인트로 접속합니다.
8. Write 트래픽은 새롭게 승격된 DR DB로 전달됩니다.
9. 서비스에서 주문, 체결, 원장 관련 DB 처리가 정상 수행되는지 검증합니다.

<br />

## 8. 장애 전환 전제 조건

DR DB Failover가 정상적으로 동작하기 위해서는 다음 조건이 충족되어야 합니다.

| 항목          | 확인 내용                                                      |
| ----------- | ---------------------------------------------------------- |
| 복제 상태       | DR DB가 Main DB의 데이터를 정상 복제하고 있어야 합니다.                      |
| 승격 가능 상태    | DR DB가 Orchestrator에서 승격 가능한 Replica로 인식되어야 합니다.           |
| ProxySQL 등록 | DR DB가 ProxySQL의 DB 서버 목록에 등록되어 있어야 합니다.                   |
| 네트워크 통신     | Orchestrator, ProxySQL, DR DB 간 통신이 가능해야 합니다.              |
| 계정 권한       | Orchestrator와 ProxySQL이 DB 상태를 조회하고 라우팅할 수 있는 권한이 있어야 합니다. |
| 데이터 정합성     | 복제 지연이 허용 범위 내에 있어야 합니다.                                   |

<br />

## 9. 상태 확인 명령어

### 9.1 DB 복제 상태 확인

MySQL 8 기준입니다.

```sql
SHOW REPLICA STATUS\G
```

MariaDB 또는 MySQL 구버전에서는 다음 명령어를 사용합니다.

```sql
SHOW SLAVE STATUS\G
```

주요 확인 항목은 다음과 같습니다.

| 항목                                            | 설명                                 |
| --------------------------------------------- | ---------------------------------- |
| Replica_IO_Running / Slave_IO_Running         | Master로부터 바이너리 로그를 가져오는 스레드 상태입니다. |
| Replica_SQL_Running / Slave_SQL_Running       | 가져온 로그를 Replica에 적용하는 스레드 상태입니다.   |
| Seconds_Behind_Source / Seconds_Behind_Master | 복제 지연 시간입니다.                       |
| Last_IO_Error                                 | IO 복제 오류 메시지입니다.                   |
| Last_SQL_Error                                | SQL 적용 오류 메시지입니다.                  |

<br />

### 9.2 Orchestrator 토폴로지 확인

```bash
orchestrator-client -c topology -i <DB_HOST>:3306
```

또는 Orchestrator Web UI에서 Main Master DB, Main Slave DB, DR DB가 하나의 토폴로지로 인식되는지 확인합니다.

<br />

### 9.3 ProxySQL 서버 등록 상태 확인

ProxySQL Admin 포트로 접속합니다.

```bash
mysql -h <PROXYSQL_IP> -P 6032 -u <ADMIN_USER> -p
```

등록된 DB 서버를 확인합니다.

```sql
SELECT hostgroup_id, hostname, port, status, weight
FROM mysql_servers;
```

Runtime 설정을 확인합니다.

```sql
SELECT hostgroup_id, hostname, port, status
FROM runtime_mysql_servers;
```

<br />

### 9.4 애플리케이션 DB 접속 확인

애플리케이션과 동일한 경로로 ProxySQL에 접속합니다.

```bash
mysql -h <PROXYSQL_ENDPOINT> -P 6033 -u <APP_USER> -p
```

현재 접속된 DB가 어느 서버인지 확인합니다.

```sql
SELECT @@hostname, @@read_only;
```

`@@read_only` 값이 `0`이면 쓰기가 가능한 Master DB입니다.

<br />

## 10. 장애 전환 검증

Failover 이후 다음 항목을 확인합니다.

| 검증 항목              | 확인 방법                                              |
| ------------------ | -------------------------------------------------- |
| DR DB Master 승격 여부 | `SELECT @@read_only;` 결과가 `0`인지 확인합니다.             |
| ProxySQL 라우팅 여부    | Write 요청이 DR DB로 전달되는지 확인합니다.                      |
| 애플리케이션 접속 여부       | 애플리케이션이 ProxySQL 엔드포인트로 정상 접속하는지 확인합니다.            |
| 데이터 쓰기 가능 여부       | 테스트 테이블 또는 테스트 트랜잭션으로 INSERT/UPDATE 가능 여부를 확인합니다.  |
| 복제 토폴로지 변경 여부      | Orchestrator에서 새로운 Master 기준으로 토폴로지가 변경되었는지 확인합니다. |
| 서비스 정상 동작 여부       | 주문, 체결, 원장 조회 API가 정상 동작하는지 확인합니다.                 |

<br />

## 11. 장애 복구 후 처리

Main 서버가 복구된 이후에는 바로 기존 Main DB를 Master로 되돌리지 않습니다.

먼저 현재 Master인 DR DB의 데이터가 최신 상태인지 확인하고, 복구된 Main DB를 새로운 Master의 Replica로 재구성해야 합니다. 이후 서비스 영향도와 데이터 정합성을 확인한 뒤, 필요 시 계획된 절차에 따라 Master 역할을 다시 Main DB로 전환합니다.

복구 후 확인 항목은 다음과 같습니다.

| 항목               | 설명                                         |
| ---------------- | ------------------------------------------ |
| 기존 Main DB 상태 확인 | 장애 원인과 데이터 손상 여부를 확인합니다.                   |
| 복제 재구성           | 복구된 Main DB를 현재 Master 기준 Replica로 재구성합니다. |
| 데이터 정합성 검증       | 주요 테이블의 데이터 누락 여부를 확인합니다.                  |
| ProxySQL 라우팅 확인  | Write Hostgroup이 현재 Master를 바라보는지 확인합니다.   |
| 서비스 정상화 확인       | 애플리케이션 기능과 주문/체결 흐름을 검증합니다.                |

<br />

## 12. 정리

본 프로젝트의 DB Failover 구성은 Main Master DB, Main Slave DB, DR DB를 하나의 장애 조치 대상으로 관리하는 구조입니다.

Orchestrator는 Master 장애를 감지하고 승격 가능한 DB를 새로운 Master로 승격합니다. ProxySQL은 애플리케이션에 단일 DB 엔드포인트를 제공하여 Master 변경 이후에도 애플리케이션의 접속 경로를 유지합니다.

이를 통해 Main DB 장애 상황에서도 DR DB를 Master로 승격하여 계정계 서비스의 DB 처리 연속성을 확보할 수 있도록 구성했습니다.
