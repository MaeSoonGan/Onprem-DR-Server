# DR 서버 네트워크 및 라우팅 구성

## 1. 개요

DR 서버는 Main 서버 장애 발생 시 원장시스템, 체결 엔진, DB, Kafka 서비스를 복구하기 위한 대기 환경입니다.

Main 서버의 원장시스템 VM과 체결 엔진 VM은 Veeam Replication을 통해 DR 서버로 복제됩니다. 장애 발생 시 DR 서버에서 Replica VM을 실행하게 되며, 이때 복제된 VM은 Main 서버의 네트워크 설정을 그대로 가지고 있을 수 있습니다.

따라서 DR 환경에서 정상적으로 통신할 수 있도록 별도의 네트워크 대역, Gateway, 라우팅, IP 전환 구성이 필요합니다.

<br />

## 2. 구성 목적

| 목적                 | 설명                                                   |
| ------------------ | ---------------------------------------------------- |
| DR 서버 내부 통신 구성     | DR DB, DR Kafka, 원장/체결 Replica VM 간 통신이 가능하도록 구성 |
| Failover 후 네트워크 전환 | Main 대역으로 복제된 VM을 DR 대역 IP로 변경                   |
| DB 승격 대응           | DR DB가 Master로 승격되었을 때 서비스가 정상적으로 접속할 수 있도록 구성   |
| 서비스 복구 검증          | DR 전환 후 DB, Kafka, 애플리케이션 통신을 확인                 |

<br />

## 3. DR 서버 기본 구성

DR 서버는 별도의 ESXi 환경으로 구성되며, Main 서버 장애 시 복제 VM을 실행하기 위한 리소스를 제공합니다.

| 구성 요소                | 역할                            |
| -------------------- | ----------------------------- |
| DR ESXi              | Veeam Replica VM 실행 대상    |
| Veeam Replica VM     | Main 원장시스템/체결 엔진 VM 복제본   |
| DR DB                | Main DB 장애 시 Master 승격 대상 |
| DR Kafka             | DR 환경의 주문·체결 이벤트 처리를 담당   |
| DR Gateway           | DR 서버 내부 VM의 기본 Gateway   |
| DR Port Group / VLAN | DR VM들이 연결되는 네트워크 영역      |

<br />

## 4. DR 서버 IP 구성

DR 서버의 VM들은 DR 전용 대역을 사용하도록 구성합니다.

| 구성 요소         | IP 예시       | 설명                               |
| ------------- | ----------- | -------------------------------- |
| DR Gateway    | `10.5.10.1` | DR VM들의 기본 Gateway          |
| 원장시스템 Replica | `10.5.10.11` | Failover 후 원장시스템 VM에 적용되는 IP. |
| 체결 엔진 Replica | `10.5.10.22` | Failover 후 체결 엔진 VM에 적용되는 IP |
| DR Kafka      | `10.5.10.33` | DR 이벤트 처리용 Kafka            |
| Veeam Server  | `10.5.10.44` | 복제 및 Failover를 관리            |
| DR DB         | `10.5.110.11` | Master 승격 대상 DB              |


<br />

## 5. DR Port Group / VLAN 구성

DR 서버에는 복구 대상 VM들이 연결될 수 있도록 DR 전용 Port Group 또는 VLAN을 구성합니다.

| 항목            | 내용                                             |
| ------------- | ---------------------------------------------- |
| VLAN ID       | VLAN 10, VLAN 110 (DB VLAN)                                 |
| Gateway       | `10.5.10.1     `                              |
| 연결 대상         | 원장시스템 Replica, 체결 엔진 Replica, DR DB, DR Kafka  |
| 목적            | Failover 이후 VM들이 DR 네트워크에서 정상 통신할 수 있도록 구성 |

<br />

## 6. 라우팅 구성

DR VM들은 기본적으로 DR Gateway를 통해 외부 대역과 통신합니다.

| 출발지           | 목적지                     | 설명                              |
| ------------- | ----------------------- | ------------------------------- |
| 원장시스템 Replica | DR DB                   | 원장 데이터 저장 및 조회를 수행        |
| 체결 엔진 Replica | DR Kafka                | 주문·체결 이벤트를 송수신              |
| 체결 엔진 Replica | DR DB       | 주문 처리 결과를 반영              |
| DR DB         | ProxySQL / Orchestrator | DB 상태 관리 및 승격 대상에 포함       |
| DR VM         | 외부 네트워크                 | DR Gateway를 통해 필요한 대역으로 라우팅 |

<br />

## 7. Failover 후 네트워크 전환

Veeam으로 복제된 Replica VM은 Main 서버의 IP 설정을 그대로 가지고 있습니다.

따라서 DR 서버에서 Replica VM을 실행한 뒤에는 DR 환경에 맞게 네트워크 설정을 변경해야 합니다.

```bash
Import-Module VMware.VimAutomation.Core -ErrorAction Stop

$logDir = "C:\DR\logs"
if (!(Test-Path $logDir)) {
New-Item -ItemType Directory -Path $logDir -Force | Out-Null
}

$timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
$logFile = "$logDir\set-dr-network-$timestamp.log"

Start-Transcript -Path $logFile -Append

try {
Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -Scope Session -Confirm:$false | Out-Null
Set-PowerCLIConfiguration -DefaultVIServerMode Single -Scope Session -Confirm:$false | Out-Null
Set-PowerCLIConfiguration -Scope User -ParticipateInCEIP $false -Confirm:$false | Out-Null

$viServer = "10.0.0.10"

# 10.0.0.10이 vCenter면 vCenter 계정
# 10.0.0.10이 ESXi 단독이면 root 사용
$viUser = "administrator@ce06.fisa"
$viPassword = "VMware1!"

# Linux guest OS 내부 계정
# 이 계정은 sudo -n 이 가능해야 함
$guestUser = "admin"
$guestPassword = "VMware1!"

# ==============================
# Credential 생성
# ==============================

$viSecure = ConvertTo-SecureString $viPassword -AsPlainText -Force
$viCred = New-Object System.Management.Automation.PSCredential ($viUser, $viSecure)

$guestSecure = ConvertTo-SecureString $guestPassword -AsPlainText -Force
$guestCred = New-Object System.Management.Automation.PSCredential ($guestUser, $guestSecure)

# 기존 연결 정리
if ($global:DefaultVIServers -and $global:DefaultVIServers.Count -gt 0) {
    Disconnect-VIServer -Server $global:DefaultVIServers -Confirm:$false | Out-Null
}

# vCenter 또는 ESXi 접속
Connect-VIServer -Server $viServer -Credential $viCred | Out-Null

function Set-DrNetwork {
    param(
        [string]$VmName,
        [string]$Device,
        [string]$NewHostname,
        [string]$IpCidr,
        [string]$Gateway,
        [string]$Dns
    )

    Write-Host "========================================"
    Write-Host "Configuring VM: $VmName"
    Write-Host "Target IP: $IpCidr"
    Write-Host "========================================"

    $vm = Get-VM -Name $VmName -ErrorAction Stop

    $vm | Select Name, PowerState, VMHost | Format-Table -AutoSize

    if ($vm.PowerState -ne "PoweredOn") {
        throw "$VmName is not PoweredOn. Failover must power on the replica VM first."
    }

    Write-Host "Waiting for VMware Tools..."
    Start-Sleep -Seconds 30

    # Linux로 넘길 Bash 명령어
    # 여러 줄 heredoc을 쓰면 Invoke-VMScript가 ; 로 합치면서 깨질 수 있어서 한 줄 명령으로 구성
    $parts = @(
'set -e;',
'CONN=$(nmcli -t -f NAME,DEVICE con show --active | grep -v ":lo$" | head -n1 | cut -d: -f1);',
'echo "CONN=$CONN";',
'test -n "$CONN" || { echo "No active NetworkManager connection found"; nmcli -t -f NAME,DEVICE con show --active; exit 1; };',
'echo "=== Before ===";',
'ip -br addr;',
'ip route;',
'hostname;',
'sudo -n hostnamectl set-hostname "' + $NewHostname + '";',
'sudo -n nmcli con mod "$CONN" ipv4.addresses "' + $IpCidr + '" ipv4.gateway "' + $Gateway + '" ipv4.dns "' + $Dns + '" ipv4.method manual;',
'sudo -n nmcli con up "$CONN";',
'echo "=== After ===";',
'ip -br addr;',
'ip route;',
'hostname;'
)

    $bash = ($parts -join ' ')

    $result = Invoke-VMScript `
        -VM $vm `
        -GuestCredential $guestCred `
        -ScriptType Bash `
        -ScriptText $bash `
        -ToolsWaitSecs 180

    Write-Host $result.ScriptOutput
}

# ==============================
# ledger-vm_replica DR IP 설정
# ==============================
Set-DrNetwork `
    -VmName "ledger-vm_replica" `
    -Device "ens192" `
    -NewHostname "ledger-vm-dr" `
    -IpCidr "10.5.10.11/24" `
    -Gateway "10.5.10.1" `
    -Dns "8.8.8.8"

# ==============================
# matching-vm_replica DR IP 설정
# ==============================
Set-DrNetwork `
    -VmName "matching-vm_replica" `
    -Device "ens192" `
    -NewHostname "matching-vm-dr" `
    -IpCidr "10.5.10.22/24" `
    -Gateway "10.5.10.1" `
    -Dns "8.8.8.8"

Write-Host "DR network configuration completed."

} catch {
Write-Host "ERROR:"
Write-Host $_
throw
} finally {
if ($global:DefaultVIServers -and $global:DefaultVIServers.Count -gt 0) {
Disconnect-VIServer -Server $global:DefaultVIServers -Confirm:$false | Out-Null
}

Stop-Transcript

}

```

Linux 기반 Replica VM의 경우 본 프로젝트 환경에서는 Veeam의 기본 Replica Re-IP 기능만으로 안정적인 IP 전환을 적용하기 어려웠습니다. 따라서 Failover 이후 PowerCLI의 Invoke-VMScript와 VMware Tools를 이용해 VM 내부에서 nmcli 명령을 실행하고, DR 대역의 IP/Gateway/DNS/Hostname을 자동으로 변경하도록 구성했습니다.

<br />

## 8. 네트워크 전환 자동화 방식

DR VM의 네트워크 전환은 다음 흐름으로 수행합니다.

1. Veeam에서 Replica VM에 대해 `Failover Now`를 실행합니다.
2. Veeam이 DR ESXi에 있는 Replica VM을 기동합니다.
3. Veeam Windows VM에서 PowerCLI 스크립트를 실행합니다.
4. PowerCLI가 vCenter/ESXi에 접속하여 대상 Replica VM을 찾습니다.
5. VMware Tools를 통해 Guest OS 내부 명령을 실행합니다.
6. `nmcli`를 사용하여 IP, Gateway, DNS를 DR 대역으로 변경합니다.
7. `hostnamectl`을 사용하여 Hostname을 변경합니다.
8. `nmcli con up`으로 변경된 네트워크 설정을 적용합니다.
9. Ping, Port, 서비스 상태를 확인합니다.

<br />

## 9. 통신 검증

DR VM 기동 및 네트워크 전환 후 다음 항목을 확인합니다.


<br />

## 10. 장애 전환 시 확인 사항

| 확인 항목       | 설명                                                 |
| ----------- | -------------------------------------------------- |
| IP 중복 여부    | Main VM과 DR Replica VM이 동시에 같은 IP를 사용하지 않는지 확인합니다. |
| DR IP 적용 여부 | Failover 후 DR 대역 IP로 변경되었는지 확인합니다.                 |
| Gateway 설정  | 기본 Gateway가 DR Gateway로 설정되었는지 확인합니다.              |
| DNS 설정      | 서비스 도메인 또는 내부 이름 해석이 가능한지 확인합니다.                   |
| DB 접속       | ProxySQL 또는 DR DB 접속이 가능한지 확인합니다.                  |
| Kafka 접속    | DR Kafka Broker 접속 및 Topic 조회가 가능한지 확인합니다.         |
| 서비스 상태      | 원장시스템, 체결 엔진 애플리케이션이 정상 기동되었는지 확인합니다.              |

<br />

## 11. 정리

DR 서버 네트워크 구성의 핵심은 Veeam으로 복제된 원장시스템과 체결 엔진 VM이 DR 서버에서 기동되었을 때, Main 서버 대역이 아닌 DR 서버 대역으로 정상 통신하도록 만드는 것입니다.

이를 위해 DR 전용 Port Group/VLAN, DR Gateway, DR IP 계획을 구성하고, Failover 이후 PowerCLI와 VMware Tools 기반 스크립트를 통해 VM 내부 네트워크 설정을 자동으로 전환하도록 구성했습니다.
