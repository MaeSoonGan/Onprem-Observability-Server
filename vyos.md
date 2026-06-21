# VyOS Router 구성

## 1. 개요 및 프로젝트 내 역할

- VyOS Router는 MaeSoonGan Observability 서버 영역에서 내부망 라우팅과 네트워크 경로 제어를 담당하는 라우터 VM입니다.

- 본 프로젝트에서는 Monitoring UI VM, Monitoring Metric VM, Monitoring Log/Trace VM이 동일한 10.6.10.0/24 대역에서 동작합니다. 이때 Monitoring 서버들이 외부망, NAT VM, Main 서버, DR 서버와 통신하기 위해서는 내부망의 기본 게이트웨이 역할을 수행하는 라우터가 필요합니다.

- VyOS Router는 Monitoring 내부망의 Gateway인 10.6.10.1로 구성되었으며, Aruba 스위치 연결망과 Monitoring 내부망을 연결하는 역할을 수행합니다. 또한 초기 패키지 설치 단계에서는 pfSense NAT VM으로 향하는 기본 라우트를 설정하여 Monitoring VM들이 인터넷에 접근할 수 있도록 구성했습니다.

</br>

## 2. 구성 환경 및 사양

| 구분              | 내용             |
| --------------- | -------------- |
| VM              | VyOS Router VM |
| 역할              | 라우팅, 내부망 경로 제어 |
| Hypervisor      | VMware ESXi    |
| OS              | VyOS           |
| vCPU            | 1 vCPU         |
| RAM             | 1GB            |
| Disk            | 5GB            |
| Network Adapter | 2개             |
| eth0            | Aruba 연결망      |
| eth1            | Monitoring 내부망 |
| eth0 IP         | 10.3.0.3/24    |
| eth1 IP         | 10.6.10.1/24   |
| 임시 NAT Next-Hop | 10.6.10.2      |
| SSH Port        | 22             |

- VyOS Router VM은 Aruba 스위치 연결용 포트그룹과 Monitoring 내부 가상 스위치 포트그룹에 각각 연결했습니다. 이를 위해 네트워크 어댑터를 2개 구성하고, eth0은 Aruba 연결망, eth1은 Monitoring 내부망에 연결했습니다.

</br>

## 3. 선택 이유 및 대안 비교

### 선택 이유

- VyOS를 선택한 이유는 가상화 환경에서 라우터 VM으로 구성하기 쉽고, Static Route와 Interface 설정을 명령어 기반으로 명확하게 관리할 수 있기 때문입니다.

- 본 프로젝트에서는 Monitoring 서버가 여러 네트워크 대역의 수집 대상과 통신해야 했습니다. 따라서 단순 NAT 장비보다 라우팅 경로를 명확히 제어할 수 있는 장비가 필요했습니다. VyOS는 CLI 기반으로 설정을 관리할 수 있어 설정 내역을 문서화하기 쉽고, 장애 발생 시 어떤 라우팅 경로가 적용되었는지 추적하기에도 적합했습니다.

### 대안 비교

| 대안           | 장점                             | 한계                                     | 최종 판단              |
| ------------ | ------------------------------ | -------------------------------------- | ------------------ |
| VyOS         | 라우팅 설정이 명확하고 CLI 기반 문서화가 쉽습니다. | GUI가 없어 초기 설정 난이도가 있습니다.               | 최종 선택했습니다.         |
| pfSense      | GUI 기반으로 NAT, 방화벽 설정이 직관적입니다.  | 라우팅 중심 구성은 VyOS보다 설명과 재현성이 떨어질 수 있습니다. | 임시 NAT 용도로 사용했습니다. |

</br>

## 4. 설치 및 주요 설정

### 4-1. 포트그룹 및 VM 생성

- 먼저 Aruba 스위치 연결용 포트그룹을 생성합니다. 외부 연결용 가상 스위치는 이미 구성되어 있었기 때문에, 해당 스위치에 VyOS Router VM을 연결할 포트그룹만 추가했습니다.

- 이후 VMware ESXi에서 VyOS Router VM을 생성하고, 네트워크 어댑터를 2개 연결합니다.

| Adapter           | 연결 대상                | 역할                     |
| ----------------- | -------------------- | ---------------------- |
| Network Adapter 1 | Aruba 연결 스위치         | 외부 또는 상위망 연결           |
| Network Adapter 2 | Monitoring 내부 가상 스위치 | Monitoring 내부망 Gateway |

### 4-2. VyOS 이미지 설치

- VyOS는 Ubuntu나 pfSense처럼 설치 직후 자동으로 디스크에 영구 설치되는 구조가 아니므로, Live ISO로 부팅한 뒤 반드시 디스크에 이미지를 설치해야 합니다.

- 설치 과정은 다음과 같습니다.

1. VyOS ISO로 VM을 부팅합니다.
2. 기본 계정으로 로그인합니다.
3. `install image` 명령어를 실행합니다.
4. Hostname, 계정, 비밀번호를 설정합니다.
5. KVM Console 사용을 선택합니다.
6. 설치 대상 디스크를 선택합니다.
7. 설치 완료 후 ISO로 다시 부팅하지 않도록 CD/DVD 마운트를 해제합니다.
8. VM을 재부팅합니다.
9. 디스크 설치가 정상적으로 적용되었는지 확인합니다.

### 4-3. Aruba 연결망 IP 설정

- eth0은 Aruba 연결망에 연결된 인터페이스로 사용했습니다.

- 설정 명령어는 다음과 같습니다.

```
configure
set interfaces ethernet eth0 description 'ARUBA'
set interfaces ethernet eth0 address '10.3.0.3/24'
commit
save
```

### 4-4. SSH 활성화

- MobaXterm 또는 원격 터미널에서 VyOS Router에 접근하기 위해 SSH를 활성화했습니다.

```
configure
set service ssh port '22'
commit
save
exit
```

### 4-5. Monitoring 내부망 IP 설정

- eth1은 Monitoring 내부망에 연결된 인터페이스로 사용했습니다. Monitoring VM들은 이 주소를 Gateway로 사용합니다.

```
configure
set interfaces ethernet eth1 description 'LAN'
set interfaces ethernet eth1 address '10.6.10.1/24'
commit
save
```

### 4-6. 임시 NAT 라우트 설정

- 초기 구축 단계에서는 Monitoring VM들이 패키지를 설치하기 위해 인터넷에 접근해야 했습니다. 이를 위해 pfSense NAT VM인 10.6.10.2를 Next-Hop으로 하는 기본 라우트를 임시로 설정했습니다.

```
configure
set protocols static route 0.0.0.0/0 next-hop 10.6.10.2
commit
save
exit
```

- 이 설정은 인터넷 사용을 위한 임시 라우트입니다. 추후 NAT용 pfSense VM을 제거하거나 라우팅 구조가 변경될 경우, 기본 라우트도 함께 수정해야 합니다. 해당 라우트를 그대로 두면 VM에서 외부로 나가는 요청이 존재하지 않는 NAT VM으로 전달될 수 있습니다.

### 4-7. Main 서버 및 DR 서버 접근 라우팅

- Monitoring 서버가 Main 서버와 DR 서버의 Metric, Log, Trace를 수집하려면 각 서버망으로 향하는 정적 라우팅이 필요합니다.

- 프로젝트의 실제 네트워크 대역에 맞춰 목적지 대역과 Next-Hop을 추가합니다.

| 목적지                 | Next-Hop | 설명                |
| ------------------- | -------- | ----------------- |
| Main Server Network | 10.4.10.1    | Main 서버 수집 대상 접근  |
| DR Server Network   | 10.5.10.1    | DR 서버 수집 대상 접근    |
| AWS / VPN Network   | 10.3.0.1    | AWS 또는 VPN 연계망 접근 |

- 예시 형식은 다음과 같습니다.

```
configure
set protocols static route 목적지_대역 next-hop 다음_홉_IP
commit
save
exit
```

</br>

## 5. 동작 흐름 및 트러블슈팅

### 동작 흐름

1. Monitoring VM이 외부 패키지 저장소 또는 수집 대상 서버로 요청을 보냅니다.
2. Monitoring VM의 기본 Gateway인 10.6.10.1로 패킷이 전달됩니다.
3. VyOS Router는 목적지 IP 대역을 기준으로 라우팅 테이블을 확인합니다.
4. 외부 인터넷 접근이 필요한 경우 임시 NAT VM인 10.6.10.2로 전달합니다.
5. Main 서버 또는 DR 서버로 향하는 요청은 해당 대역의 Static Route를 기준으로 전달합니다.
6. 수집 대상 서버가 응답을 반환하면 VyOS Router를 거쳐 Monitoring VM으로 돌아옵니다.
7. 이후 Prometheus, Loki, Tempo, Grafana가 해당 데이터를 수집하거나 조회합니다.

### 트러블슈팅

#### 문제 1. VM 종료 후 VyOS 설정이 사라지는 문제

| 구분    | 내용                                                                                                                   |
| ----- | -------------------------------------------------------------------------------------------------------------------- |
| 현상    | Monitoring 서버 연결 설정을 완료했지만, 이후 동일하게 접속을 시도했을 때 연결이 되지 않았습니다.                                                         |
| 원인    | VyOS가 디스크에 설치된 상태가 아니라 Live ISO로만 부팅 중이었기 때문에, VM 종료 후 설정이 모두 휘발되었습니다.                                               |
| 해결 과정 | Router Gateway로 Ping을 보내 라우터 이상 여부를 확인했습니다. 이후 VyOS VM을 재부팅하고, `install image`를 통해 디스크에 VyOS를 설치한 뒤 ISO 마운트를 해제했습니다. |
| 결과    | 재부팅 후에도 Interface IP와 라우팅 설정이 유지되었습니다.                                                                               |
| 재발 방지 | VyOS 구축 시 Live ISO 부팅 상태에서 설정을 끝내지 않고, 반드시 디스크 설치와 `commit`, `save` 적용 여부를 확인하도록 절차화했습니다.                            |

#### 문제 2. Monitoring VM에서 외부 통신이 되지 않는 문제

| 구분    | 내용                                                                                         |
| ----- | ------------------------------------------------------------------------------------------ |
| 현상    | Monitoring VM에서 패키지 설치를 위한 외부 통신이 되지 않았습니다.                                                |
| 원인    | 외부 인터넷 접근을 위해 pfSense NAT VM으로 향하는 기본 라우트가 필요했지만, 해당 라우트가 누락되었거나 Next-Hop이 잘못 설정될 수 있었습니다. |
| 해결 과정 | VyOS Router에서 `0.0.0.0/0` 기본 라우트가 10.6.10.2로 설정되어 있는지 확인하고, 누락된 경우 다시 추가했습니다.              |
| 결과    | Monitoring VM에서 NAT VM을 통해 외부 패키지 저장소에 접근할 수 있었습니다.                                        |
| 재발 방지 | NAT VM을 사용하는 기간과 제거 시점을 문서화하고, NAT VM 제거 시 기본 라우트 수정 여부를 함께 점검하도록 정리했습니다.                  |

#### 문제 3. 인터페이스 연결 오류로 통신이 되지 않는 문제

| 구분    | 내용                                                                                                |
| ----- | ------------------------------------------------------------------------------------------------- |
| 현상    | eth0 또는 eth1에 IP를 설정했지만 의도한 네트워크와 통신되지 않았습니다.                                                     |
| 원인    | VM의 Network Adapter가 Aruba 연결 포트그룹 또는 Monitoring 내부 포트그룹에 올바르게 연결되지 않았을 가능성이 있었습니다.               |
| 해결 과정 | ESXi에서 VyOS VM의 Network Adapter 연결 상태를 확인하고, eth0은 Aruba 연결망, eth1은 Monitoring 내부망에 연결되도록 수정했습니다. |
| 결과    | 10.3.0.0/24 대역과 10.6.10.0/24 대역의 역할이 분리되었고, Monitoring VM의 Gateway 통신이 정상화되었습니다.                  |
| 재발 방지 | VyOS VM 생성 시 Adapter 번호와 Port Group 연결 관계를 표로 정리하고, IP 설정 전 먼저 연결 상태를 확인하도록 했습니다.                 |
