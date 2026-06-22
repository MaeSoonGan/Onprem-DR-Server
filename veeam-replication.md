# Veeam Replication 구성

## 1. 개요

본 문서는 MAIN 서버에서 운영 중인 핵심 VM을 DR 서버로 복제하기 위한 Veeam Replication 구성 내용을 정리한다.

본 프로젝트에서는 원장 시스템과 체결 시스템이 온프레미스 MAIN 서버에서 동작한다. 해당 VM들은 서비스의 핵심 거래 흐름을 담당하므로, MAIN 서버 장애 시 DR 서버에서 빠르게 기동할 수 있도록 Veeam Backup & Replication을 이용해 VM 단위 복제를 구성하였다.

Veeam Replication은 VM을 DR ESXi로 복제해 두고, 장애 발생 시 복제본 VM을 Failover하여 DR 환경에서 서비스를 재개하는 방식이다.

---

## 2. 구성 목적

Veeam Replication 구성의 목적은 다음과 같다.

```text
1. MAIN 서버의 원장 VM과 체결 VM을 DR 서버로 복제
2. MAIN 장애 발생 시 DR 서버에서 Replica VM 기동
3. 원장/체결 시스템의 복구 시간 단축
4. 장애 복구 테스트 시 Undo Failover를 통해 원복 가능
5. 실제 장애 상황에서는 Permanent Failover 또는 Failback 절차 수행 가능
```

본 프로젝트에서는 Veeam Community Edition을 사용했기 때문에, Enterprise 기능인 Failover Plan 기반 일괄 자동화는 사용하지 않고, 각 Replica VM에 대해 `Failover Now`를 수동 실행하는 방식으로 구성하였다.

---

## 3. 전체 구성 구조

```text
[MAIN 서버 영역]

vCenter
 └─ Core Cluster
     ├─ ESXi-01
     └─ ESXi-02
          ├─ ledger-vm
          └─ matching-vm


[Veeam 서버]

Veeam Backup & Replication
 ├─ MAIN vCenter 등록
 ├─ DR ESXi 등록
 └─ Replication Job 구성


[DR 서버 영역]

DR ESXi
 ├─ ledger-vm_replica
 └─ matching-vm_replica
```

---

## 4. 대상 VM

Veeam Replication 대상 VM은 다음과 같다.

| 구분     | MAIN 원본 VM    | DR Replica VM         | 역할                        |
| ------ | ------------- | --------------------- | ------------------------- |
| 원장 시스템 | `ledger-vm`   | `ledger-vm_replica`   | 계좌, 잔고, 거래 내역 등 원장 데이터 처리 |
| 체결 시스템 | `matching-vm` | `matching-vm_replica` | 주문 체결 및 체결 결과 처리          |

Veeam 콘솔의 `Replicas → Ready` 화면에서는 원본 VM 이름인 `ledger-vm`, `matching-vm`으로 보일 수 있다.
하지만 vCenter 또는 DR ESXi에서 실제 생성된 Replica VM 이름은 `_replica`가 붙은 형태이다.

```text
Veeam Ready 화면:
- ledger-vm
- matching-vm

DR ESXi 실제 VM:
- ledger-vm_replica
- matching-vm_replica
```

---

## 5. Veeam 서버 역할

Veeam Backup & Replication 서버는 Windows VM에 설치하였다.

Veeam 서버의 역할은 다음과 같다.

```text
1. MAIN vCenter에 접속하여 원본 VM 조회
2. DR ESXi에 접속하여 Replica VM 생성
3. Replication Job 실행
4. 변경분을 DR Replica VM에 반영
5. Failover Now / Undo Failover 등 복구 작업 수행
```

Veeam 서버가 VM 내부 서비스를 직접 복제하는 것이 아니라, vCenter/ESXi와 연동하여 VM 디스크와 설정을 복제하는 방식이다.

---

## 6. VMware 서버 등록

Veeam에서 복제를 수행하려면 먼저 VMware 인프라를 등록해야 한다.

### 6.1 MAIN vCenter 등록

MAIN 서버는 ESXi 2대로 구성된 vCenter Cluster 구조이므로, Veeam에는 개별 ESXi가 아니라 MAIN vCenter를 등록하였다.

```text
Veeam Console
→ Backup Infrastructure
→ Managed Servers
→ Add Server
→ VMware vSphere
→ vSphere
→ MAIN vCenter 등록
```

MAIN vCenter를 등록한 이유는 다음과 같다.

```text
1. MAIN VM들이 vCenter Cluster에서 관리됨
2. HA/DRS 환경에서는 VM이 특정 ESXi에 고정되어 있지 않을 수 있음
3. vCenter를 등록해야 클러스터 단위로 VM을 안정적으로 조회 가능
```

### 6.2 DR ESXi 등록

DR 서버는 Replica VM이 저장되고 Failover 시 기동될 대상 서버이다.

```text
Veeam Console
→ Backup Infrastructure
→ Managed Servers
→ Add Server
→ VMware vSphere
→ vSphere
→ DR ESXi 등록
```

DR ESXi에는 복제된 VM이 다음과 같이 생성된다.

```text
ledger-vm_replica
matching-vm_replica
```

---

## 7. Replication Job 구성

Replication Job은 MAIN VM을 DR ESXi로 복제하는 작업이다.

```text
Veeam Console
→ Home
→ Jobs
→ Replication
→ New Replication Job
```

### 7.1 Job 이름

Replication Job 이름은 다음과 같이 구성하였다.

```text
Main-Core-to-DR-Replication
```

이름은 MAIN의 핵심 시스템을 DR로 복제한다는 의미를 갖는다.

---

### 7.2 Source VM 선택

복제 대상 VM으로 원장 VM과 체결 VM을 선택하였다.

```text
Source VM:
- ledger-vm
- matching-vm
```

해당 VM들은 MAIN vCenter에서 관리되는 VM이다.

---

### 7.3 Destination 설정

복제 대상은 DR ESXi로 지정하였다.

```text
Destination:
- DR ESXi
```

DR ESXi에는 원본 VM 이름 뒤에 `_replica`가 붙은 Replica VM이 생성된다.

```text
ledger-vm → ledger-vm_replica
matching-vm → matching-vm_replica
```

---

### 7.4 Datastore 설정

Replica VM의 디스크가 저장될 DR ESXi의 Datastore를 지정하였다.

```text
Replica VM Disk
→ DR ESXi Datastore
```

원본 VM이 Thin Provisioning으로 구성된 경우, Replica VM도 기본적으로 Thin 형태로 생성될 수 있다.
복제 후 DR ESXi에서 Replica VM의 디스크 사용량과 Datastore 여유 공간을 확인하였다.

---

### 7.5 Network 설정

Replication Job 구성 시, Replica VM이 연결될 DR 네트워크를 지정한다.

Veeam의 Network Mapping은 원본 VM이 사용하던 MAIN 네트워크를 DR 환경의 Port Group으로 매핑하는 역할을 한다.

```text
MAIN Port Group
→ DR Port Group
```

단, Network Mapping은 VM의 가상 NIC가 연결될 네트워크를 바꾸는 기능이다.
Linux Guest OS 내부의 IP, Gateway, DNS를 직접 변경하는 기능과는 다르다.

따라서 본 프로젝트에서는 Failover 이후 Linux Replica VM 내부 IP 변경을 별도 PowerCLI 스크립트로 처리하였다.

관련 문서:

```text
./veeam/vm-failover-network-switch.md
```

---

### 7.6 Restore Point 설정

Replication Job은 Replica VM의 복구 지점을 생성한다.

복구 지점은 장애 발생 시 어느 시점의 VM 상태로 Failover할지 결정하는 기준이 된다.

```text
Restore Point
→ 마지막 Replication Job 성공 시점 기준
```

즉, Veeam Replication은 실시간 복제가 아니라 Job 실행 시점 기준으로 변경분을 반영하는 방식이다.

예를 들어 마지막 복제가 10:30에 성공했고, 10:40에 MAIN 장애가 발생했다면 DR은 10:30 시점의 Replica VM으로 복구된다.

---

## 8. 복제 방식

본 프로젝트에서 구성한 방식은 일반 Veeam Replication Job 기반의 스케줄/수동 복제 방식이다.

```text
수동 실행:
관리자가 Replication Job을 직접 Start

스케줄 실행:
정해진 주기에 따라 Replication Job 자동 실행
```

일반 Replication Job은 실시간 동기화 방식이 아니며, 마지막 복제 성공 시점까지의 상태를 DR에 반영한다.

따라서 RPO는 Replication Job 실행 주기에 영향을 받는다.

```text
30분 주기 복제 → 최대 약 30분 데이터 손실 가능
10분 주기 복제 → 최대 약 10분 데이터 손실 가능
수동 복제 → 마지막 수동 실행 시점 기준 복구
```

---

## 9. Replication Job 실행

Replication Job은 다음 경로에서 실행한다.

```text
Veeam Console
→ Home
→ Jobs
→ Replication
→ Main-Core-to-DR-Replication 우클릭
→ Start
```

Job이 정상적으로 수행되면 각 VM에 대해 Success 상태가 표시된다.

확인 항목은 다음과 같다.

```text
1. Job 결과가 Success인지 확인
2. ledger-vm 복제 성공 여부 확인
3. matching-vm 복제 성공 여부 확인
4. DR ESXi에 Replica VM 생성 여부 확인
5. Replicas → Ready 상태 확인
```

정상적으로 복제가 완료되면 다음 위치에서 확인할 수 있다.

```text
Home
→ Replicas
→ Ready
```

Ready 상태는 Replica VM이 DR에 준비되어 있지만, 아직 Failover되어 운영 중인 상태는 아니라는 의미이다.

---

## 10. Replica VM 상태

평상시 DR Replica VM은 Ready 상태로 대기한다.

```text
Veeam 상태:
Replicas → Ready

DR ESXi 상태:
ledger-vm_replica      PoweredOff
matching-vm_replica    PoweredOff
```

Replica VM은 평상시에 직접 켜지 않는다.

DR ESXi 또는 vCenter에서 사용자가 직접 Replica VM을 Power On하면 Veeam이 관리하는 Failover 상태와 실제 VM 전원 상태가 어긋날 수 있다.

따라서 Replica VM 기동은 반드시 Veeam의 `Failover Now`를 통해 수행한다.

---

## 11. 장애 발생 시 Failover 절차

MAIN 서버 장애 발생 시 다음 순서로 DR 전환을 수행한다.

### 11.1 MAIN VM 상태 확인

먼저 MAIN 원장/체결 VM이 꺼졌거나 접근 불가능한 상태인지 확인한다.

```text
MAIN ledger-vm      OFF 또는 장애 상태
MAIN matching-vm    OFF 또는 장애 상태
```

MAIN과 DR VM이 동시에 켜지면 같은 IP, Hostname, 서비스가 중복될 수 있으므로 주의해야 한다.

---

### 11.2 Veeam에서 Failover Now 실행

Veeam에서 Replica VM을 Failover한다.

```text
Home
→ Replicas
→ Ready
→ ledger-vm 우클릭
→ Failover Now
```

이후 체결 VM도 동일하게 실행한다.

```text
Home
→ Replicas
→ Ready
→ matching-vm 우클릭
→ Failover Now
```

Failover Now를 실행하면 Veeam이 DR ESXi에 있는 Replica VM을 기동한다.

```text
ledger-vm_replica      PoweredOn
matching-vm_replica    PoweredOn
```

---

### 11.3 DR 네트워크 전환

Failover 직후 Linux Replica VM은 MAIN IP 설정을 가지고 있을 수 있다.

따라서 본 프로젝트에서는 Veeam Windows VM에서 PowerCLI 스크립트를 실행하여 DR IP로 변경한다.

```text
C:\DR\Run-Set-DR-Network.bat
```

해당 스크립트는 PowerCLI와 VMware Tools를 이용해 Linux VM 내부에서 `nmcli` 명령을 실행한다.

최종 변경 값은 다음과 같다.

| VM                    | DR IP           | Gateway     | DNS       | Hostname         |
| --------------------- | --------------- | ----------- | --------- | ---------------- |
| `ledger-vm_replica`   | `10.5.10.11/24` | `10.5.10.1` | `8.8.8.8` | `ledger-vm-dr`   |
| `matching-vm_replica` | `10.5.10.22/24` | `10.5.10.1` | `8.8.8.8` | `matching-vm-dr` |

세부 내용은 별도 문서에서 정리한다.

```text
./veeam/vm-failover-network-switch.md
```

---

## 12. Failover 이후 검증

DR Replica VM이 정상적으로 기동되고 네트워크가 전환되었는지 확인한다.

### 12.1 Ping 확인

```powershell
ping 10.5.10.11
ping 10.5.10.22
```

### 12.2 Port 확인

```powershell
Test-NetConnection 10.5.10.11 -Port 22
Test-NetConnection 10.5.10.22 -Port 22
```

필요한 경우 서비스 포트도 확인한다.

```powershell
Test-NetConnection 10.5.10.11 -Port <서비스포트>
Test-NetConnection 10.5.10.22 -Port <서비스포트>
```

### 12.3 서비스 상태 확인

Linux VM 내부에서 서비스 상태를 확인한다.

```bash
systemctl status <service-name>
```

Docker 기반 서비스인 경우 다음 명령으로 확인한다.

```bash
docker ps
docker logs <container-name>
```

---

## 13. 테스트 종료 시 Undo Failover

DR 전환이 테스트 목적이라면 `Undo Failover`를 수행한다.

```text
Home
→ Replicas
→ Failover
→ ledger-vm, matching-vm 선택
→ Undo Failover
```

Undo Failover를 실행하면 Failover 중 DR Replica VM에서 발생한 변경사항은 폐기되고, Replica VM은 다시 Ready 상태로 돌아간다.

```text
Failover Now
→ Replica VM 기동
→ DR IP 변경
→ 테스트 수행
→ Undo Failover
→ 변경사항 폐기
→ Ready 상태 복귀
```

테스트 종료 후 정상 상태는 다음과 같다.

```text
Veeam:
Replicas → Ready

DR ESXi:
ledger-vm_replica      PoweredOff
matching-vm_replica    PoweredOff
```

---

## 14. 실제 장애 상황에서의 처리

실제 MAIN 장애 상황에서는 테스트와 달리 `Undo Failover`를 실행하면 안 된다.

실제 장애 발생 시 DR Replica VM이 임시 운영 서버가 되므로, 장애 복구 방향을 결정해야 한다.

선택지는 다음과 같다.

```text
1. Permanent Failover
   DR Replica VM을 새로운 운영 VM으로 확정

2. Failback
   MAIN 복구 후 DR에서 운영 중 생긴 변경사항을 MAIN으로 되돌림
```

실제 장애 상황에서는 서비스 정상 여부를 확인한 후, 운영 방침에 따라 Permanent Failover 또는 Failback을 선택한다.

---

## 15. Community Edition 제약

본 프로젝트에서는 Veeam Community Edition을 사용하였다.

Failover Plan 기능을 실행하려고 했을 때 다음과 같은 라이선스 제한 메시지가 발생하였다.

```text
This functionality requires a valid Enterprise edition license.
```

따라서 Failover Plan 기반의 일괄 자동 장애 전환은 사용하지 않았다.

대신 다음과 같은 반자동 절차로 구성하였다.

```text
1. ledger-vm Failover Now 수동 실행
2. matching-vm Failover Now 수동 실행
3. PowerCLI 기반 네트워크 전환 스크립트 수동 실행
4. Ping, Port, 서비스 상태 확인
```

즉, VM Failover 자체는 Veeam 콘솔에서 수동으로 실행하고, Failover 이후 Linux Replica VM의 IP/Gateway/DNS/Hostname 변경은 스크립트로 자동화하였다.

---

## 16. 운영 시 주의사항

Veeam Replication 운영 시 주의사항은 다음과 같다.

```text
1. Replica VM을 vCenter/ESXi에서 직접 Power On하지 않는다.
2. Replica VM 기동은 Veeam Failover Now를 통해 수행한다.
3. 테스트 목적이면 반드시 Undo Failover로 원복한다.
4. 실제 장애 상황에서는 Undo Failover가 아니라 Permanent Failover 또는 Failback을 검토한다.
5. Replication Job의 마지막 성공 시점이 실제 복구 가능 시점이 된다.
6. Replication Job이 실패하면 DR Replica가 최신 상태가 아닐 수 있다.
7. Linux Replica VM의 IP 전환은 Veeam 기본 Re-IP가 아니라 별도 PowerCLI 스크립트로 처리한다.
8. 스크립트에 계정 비밀번호를 평문으로 남길 경우 외부 공유 문서에서는 반드시 마스킹한다.
```

---

## 17. 장애 대응 절차 요약

MAIN 서버 장애 발생 시 전체 절차는 다음과 같다.

```text
1. MAIN ledger-vm / matching-vm 장애 여부 확인
2. Veeam Replication Job 마지막 성공 시점 확인
3. Replicas → Ready에서 ledger-vm Failover Now 실행
4. Replicas → Ready에서 matching-vm Failover Now 실행
5. DR ESXi에서 Replica VM PoweredOn 확인
6. Veeam Windows VM에서 C:\DR\Run-Set-DR-Network.bat 실행
7. ping 10.5.10.11 확인
8. ping 10.5.10.22 확인
9. SSH 또는 서비스 포트 확인
10. 원장/체결 서비스 상태 확인
11. 실제 장애 상황이면 Undo Failover를 실행하지 않음
12. MAIN 복구 후 Permanent Failover 또는 Failback 결정
```

---

## 18. 검증 결과

본 프로젝트에서는 다음 항목을 검증하였다.

```text
1. MAIN vCenter와 DR ESXi를 Veeam에 등록
2. ledger-vm, matching-vm Replication Job 생성
3. DR ESXi에 Replica VM 생성 확인
4. Replication Job 수동 실행 성공
5. Veeam Replicas → Ready 상태 확인
6. Failover Now를 통한 Replica VM 기동 확인
7. PowerCLI 스크립트를 통한 DR IP 전환 확인
8. ping을 통한 DR VM 통신 확인
9. Undo Failover를 통한 테스트 원복 확인
```

---

## 19. 정리

본 구성에서는 Veeam Replication을 통해 MAIN 서버의 원장 VM과 체결 VM을 DR ESXi로 복제하였다.

평상시에는 DR Replica VM이 Ready 상태로 대기하며, MAIN 장애 발생 시 Veeam의 Failover Now를 통해 DR Replica VM을 기동한다. 이후 PowerCLI 기반 네트워크 전환 스크립트를 실행하여 Linux Guest OS 내부의 IP, Gateway, DNS, Hostname을 DR 대역으로 변경한다.

최종적으로 본 프로젝트의 DR 전환 방식은 다음과 같이 정리할 수 있다.

```text
Veeam Replication Job
→ DR Replica VM Ready 상태 유지
→ 장애 시 Failover Now 수동 실행
→ PowerCLI 기반 DR 네트워크 전환
→ 서비스 상태 확인
→ Permanent Failover 또는 Failback 판단
```
