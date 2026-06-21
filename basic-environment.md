# 기본 환경 구축

## 1. 개요 및 프로젝트 내 역할

- 기본 환경 구축은 MaeSoonGan Observability 서버를 구성하기 전에 수행한 공통 인프라 준비 작업입니다.

- 본 프로젝트의 Observability 서버는 Grafana, Alertmanager, Prometheus, Loki, Tempo, Alloy 등을 역할별 VM에 분리하여 배치하는 구조로 구성했습니다. 각 VM이 동일한 내부망에서 통신하고, Docker 기반으로 모니터링 구성 요소를 실행할 수 있도록 VM 생성, 내부 전용 가상 스위치 및 포트그룹 구성, Ubuntu Server 설치, 고정 IP 설정, 공통 패키지 설치, Docker 설치, 작업 디렉터리 생성을 먼저 진행했습니다.

- 이 기본 환경은 Monitoring UI VM, Monitoring Metric VM, Monitoring Log/Trace VM에 공통으로 적용되며, pfSense NAT VM과 VyOS Router VM은 모니터링 서버의 외부 통신과 내부망 라우팅을 지원하는 역할로 함께 구성했습니다.

</br>

## 2. 구성 환경 및 사양

### 공통 구성 환경

| 구분                | 내용                    |
| ----------------- | --------------------- |
| Hypervisor        | VMware ESXi           |
| Virtual Network   | 내부 전용 가상 스위치          |
| OS                | Ubuntu Server         |
| IP 설정 방식          | Manual IPv4           |
| Gateway           | 10.6.10.1             |
| DNS               | 8.8.8.8, 1.1.1.1      |
| Container Runtime | Docker                |
| Compose           | Docker Compose Plugin |
| 공통 작업 경로          | /opt/monitoring       |

### VM별 사양

| VM                      | 역할                                         | vCPU   | RAM   | Disk |
| ----------------------- | ------------------------------------------ | ------ | ----- | ---- |
| pfSense NAT VM          | 임시 NAT, 패키지 설치용 인터넷 연결                     | 1 vCPU | 1~2GB | 8GB  |
| VyOS Router VM          | 라우팅, 내부망 경로 제어                             | 1 vCPU | 1GB   | 5GB  |
| Monitoring UI VM        | Grafana, Alertmanager, CloudWatch Exporter | 2 vCPU | 2~3GB | 20GB |
| Monitoring Metric VM    | Prometheus                                 | 2 vCPU | 5GB   | 60GB |
| Monitoring Log/Trace VM | Loki, Tempo                                | 2 vCPU | 8GB   | 90GB |
| 예비                      | ESXi 여유, 업데이트, 임시 파일                       | -      | 1~3GB | 17GB |

### Monitoring VM 구성

| VM 역할                   | Hostname               | IP Address    | 주요 용도                 | 작업 디렉터리                    |
| ----------------------- | ---------------------- | ------------- | --------------------- | -------------------------- |
| Monitoring UI VM        | fs-monitoring-ui       | 10.6.10.11/24 | Grafana, Alertmanager | /opt/monitoring/ui         |
| Monitoring Metric VM    | fs-monitoring-metric   | 10.6.10.22/24 | Prometheus            | /opt/monitoring/prometheus |
| Monitoring Log/Trace VM | fs-monitoring-logtrace | 10.6.10.33/24 | Loki, Tempo           | /opt/monitoring/logtrace   |

</br>

## 3. 선택 이유 및 대안 비교

### 선택 이유

- Monitoring 서버는 여러 구성 요소를 동시에 실행해야 하므로 역할별 VM 분리를 선택했습니다. UI, Metric, Log/Trace 영역을 분리하면 각 도구의 리소스 사용량과 장애 영향을 구분하기 쉽고, 장애 발생 시 어느 계층에서 문제가 발생했는지 파악하기 쉽습니다.

- 운영체제는 Ubuntu Server를 사용했습니다. Ubuntu Server는 Docker, Grafana, Prometheus 계열 도구 설치가 쉽고 관련 문서가 많아 실습 환경에서 빠르게 구축하기 적합합니다.

- 서비스 실행 방식은 Docker Compose를 사용했습니다. Prometheus, Grafana, Alertmanager, Loki, Tempo를 컨테이너 단위로 실행하면 설정 파일과 데이터 디렉터리를 분리해 관리할 수 있고, 재기동과 백업도 단순해집니다.

### 대안 비교

| 대안             | 장점                               | 한계                                                  | 최종 판단      |
| -------------- | -------------------------------- | --------------------------------------------------- | ---------- |
| 단일 VM 구성       | 구축이 단순하고 리소스를 적게 사용합니다.          | Metric, Log, Trace 저장소가 한 VM에 집중되어 장애 영향 범위가 커집니다.  | 제외했습니다.    |
| 역할별 VM 분리      | 구성 요소별 리소스와 장애 범위를 분리할 수 있습니다.   | VM 수가 늘어나 초기 구성이 조금 복잡합니다.                          | 최종 선택했습니다. |
| Ubuntu Server  | Docker 기반 도구 설치가 쉽고 문서가 많습니다.    | 최소 설치 후 필요한 패키지를 직접 구성해야 합니다.                       | 최종 선택했습니다. |
| Rocky Linux    | 서버 운영 환경에 적합하고 안정적입니다.           | Monitoring VM 공통 설치 절차를 빠르게 맞추기에는 Ubuntu가 더 단순했습니다. | 제외했습니다.    |
| Docker Compose | 설정 파일과 데이터 디렉터리를 명확히 분리할 수 있습니다. | 컨테이너 데이터 디렉터리 권한 설정이 필요합니다.                         | 최종 선택했습니다. |

</br>

## 4. 설치 및 주요 설정

### 4-1. 내부 전용 가상 스위치 및 포트그룹 구성

- ESXi 또는 vSphere에서 Monitoring 서버용 내부 전용 가상 스위치를 생성하고, 해당 스위치에 포트그룹을 생성합니다.

- 이후 pfSense NAT VM, VyOS Router VM, Monitoring UI VM, Monitoring Metric VM, Monitoring Log/Trace VM의 Network Adapter를 필요한 포트그룹에 연결합니다.

- 구성 절차는 다음과 같습니다.

1. ESXi 또는 vSphere에서 가상 스위치를 생성합니다.
2. 생성한 가상 스위치에 포트그룹을 생성합니다.
3. 각 VM의 Network Adapter를 목적에 맞는 포트그룹에 연결합니다.
4. Monitoring VM에는 10.6.10.0/24 대역의 고정 IP를 할당합니다.
5. Gateway는 VyOS Router의 LAN IP인 10.6.10.1로 설정합니다.

### 4-2. Ubuntu Server 설치

- Monitoring UI VM, Monitoring Metric VM, Monitoring Log/Trace VM은 Ubuntu Server로 설치합니다.

- 설치 과정에서는 네트워크 설정에서 IPv4 방식을 Manual로 변경하고, VM별 고정 IP를 입력합니다.

| 항목      | 설정               |
| ------- | ---------------- |
| Address | VM별 IP 입력        |
| Subnet  | 10.6.10.0/24     |
| Gateway | 10.6.10.1        |
| DNS     | 8.8.8.8, 1.1.1.1 |

- SSH 접근을 위해 OpenSSH Server를 설치하고, 설치 완료 후 재부팅합니다.

### 4-3. Netplan 설정

- Ubuntu 설치 과정에서 Gateway를 잘못 입력하면 기본 라우팅이 생성되지 않아 pfSense NAT 또는 외부망으로 통신할 수 없습니다.

- 이 경우 Netplan 설정 파일을 수정하여 Gateway를 10.6.10.1로 맞춥니다.

- 확인 명령어는 다음과 같습니다.

```
ls /etc/netplan
cat /etc/netplan/*.yaml
```

- 수정 대상 파일 예시는 다음과 같습니다.

```
sudo nano /etc/netplan/00-installer-config.yaml
```

- 또는 파일명이 다른 경우 다음 파일을 수정합니다.

```
sudo nano /etc/netplan/50-cloud-init.yaml
```

- 설정 예시는 다음과 같습니다.

```
network:
  version: 2
  ethernets:
    ens192:
      addresses:
        - 10.6.10.11/24
      routes:
        - to: default
          via: 10.6.10.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

- 설정 적용 명령어는 다음과 같습니다.

```
sudo netplan apply
ip route
```

- `ip route` 결과에서 `default via 10.6.10.1` 경로가 보이면 정상입니다.

### 4-4. 공통 패키지 설치

- 각 VM에서 공통 운영 및 점검에 필요한 패키지를 설치합니다.

```
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release vim net-tools unzip wget git htop jq tree
```

- 패키지별 사용 목적은 다음과 같습니다.

| 패키지                    | 사용 목적                       |
| ---------------------- | --------------------------- |
| curl, wget             | 설치 파일 다운로드 및 API 테스트        |
| gnupg, ca-certificates | Docker 및 Grafana 저장소 Key 등록 |
| vim                    | 설정 파일 수정                    |
| net-tools              | 네트워크 상태 확인                  |
| unzip                  | 압축 파일 해제                    |
| git                    | 설정 파일 및 문서 관리               |
| htop                   | 리소스 사용량 확인                  |
| jq                     | JSON 응답 확인                  |
| tree                   | 디렉터리 구조 확인                  |

### 4-5. Docker 및 Docker Compose 설치

- Observability 구성 요소를 Docker Compose 기반으로 실행하기 위해 Docker를 설치합니다.

- Docker GPG Key 및 저장소를 등록합니다.

```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

- Docker와 Docker Compose Plugin을 설치합니다.

```
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

- `fisa` 사용자가 Docker 명령어를 실행할 수 있도록 Docker 그룹에 추가합니다.

```
sudo usermod -aG docker fisa
```

- 그룹 권한 적용을 위해 SSH를 재접속합니다.

```
exit
```

- 재접속 후 Docker 설치 상태를 확인합니다.

```
docker --version
docker compose version
docker run hello-world
```

### 4-6. 공통 디렉터리 생성

- Monitoring 서버의 설정 파일과 데이터 디렉터리를 관리하기 위해 `/opt/monitoring` 하위에 VM별 작업 디렉터리를 생성합니다.

```
sudo mkdir -p /opt/monitoring
sudo chown -R fisa:fisa /opt/monitoring
```

- VM별 디렉터리는 다음과 같이 사용합니다.

| VM                      | 디렉터리                       |
| ----------------------- | -------------------------- |
| Monitoring UI VM        | /opt/monitoring/ui         |
| Monitoring Metric VM    | /opt/monitoring/prometheus |
| Monitoring Log/Trace VM | /opt/monitoring/logtrace   |

</br>

## 5. 동작 흐름 및 트러블슈팅

### 동작 흐름

1. ESXi에서 내부 전용 가상 스위치와 포트그룹을 생성합니다.
2. pfSense NAT VM, VyOS Router VM, Monitoring UI VM, Monitoring Metric VM, Monitoring Log/Trace VM을 생성합니다.
3. 각 VM의 Network Adapter를 목적에 맞는 포트그룹에 연결합니다.
4. Monitoring VM에 Ubuntu Server를 설치합니다.
5. VM별 고정 IP, Gateway, DNS를 설정합니다.
6. Gateway는 VyOS Router의 LAN IP인 10.6.10.1로 지정합니다.
7. 공통 패키지와 Docker를 설치합니다.
8. `/opt/monitoring` 하위에 VM별 작업 디렉터리를 생성합니다.
9. 이후 각 VM 역할에 따라 Grafana, Alertmanager, Prometheus, Loki, Tempo를 Docker Compose로 배치합니다.

### 트러블슈팅

#### 문제 1. Gateway 오입력으로 외부 통신이 되지 않는 문제

| 구분    | 내용                                                                      |
| ----- | ----------------------------------------------------------------------- |
| 현상    | VM에서 외부 통신이 되지 않고, NAT 또는 WAN으로 이동하지 못했습니다.                             |
| 원인    | Gateway를 10.6.10.1이 아니라 10.6.0.1로 잘못 입력하여 default route가 정상 생성되지 않았습니다. |
| 해결 과정 | Netplan 설정 파일에서 Gateway를 10.6.10.1로 수정하고 `sudo netplan apply`를 실행했습니다.  |
| 결과    | `ip route`에서 `default via 10.6.10.1` 경로를 확인했고 외부 통신이 정상화되었습니다.          |
| 재발 방지 | VM 생성 시 IP, Gateway, DNS 설정을 표준 표에 맞춰 확인한 뒤 설치를 완료하도록 절차화했습니다.          |

#### 문제 2. Docker 명령어 실행 권한 문제

| 구분    | 내용                                                                                      |
| ----- | --------------------------------------------------------------------------------------- |
| 현상    | 일반 사용자 `fisa` 계정에서 Docker 명령어 실행 시 권한 오류가 발생했습니다.                                       |
| 원인    | Docker 그룹에 `fisa` 사용자가 추가되지 않았거나, 그룹 변경 후 세션을 재접속하지 않았습니다.                              |
| 해결 과정 | `sudo usermod -aG docker fisa` 명령어를 실행한 뒤 SSH를 재접속했습니다.                                 |
| 결과    | `docker --version`, `docker compose version`, `docker run hello-world` 명령어가 정상 실행되었습니다. |
| 재발 방지 | Docker 설치 직후 사용자 그룹 추가와 SSH 재접속을 공통 절차에 포함했습니다.                                         |

#### 문제 3. VM 간 통신이 되지 않는 문제

| 구분    | 내용                                                                               |
| ----- | -------------------------------------------------------------------------------- |
| 현상    | VM에 IP가 설정되어 있어도 다른 Monitoring VM과 통신되지 않았습니다.                                   |
| 원인    | VM Network Adapter가 의도한 포트그룹이 아닌 다른 네트워크에 연결되어 있었습니다.                            |
| 해결 과정 | ESXi 또는 vSphere에서 각 VM의 Network Adapter가 올바른 포트그룹에 연결되어 있는지 확인했습니다.              |
| 결과    | Port Group 수정 후 VM 간 Ping 통신이 정상적으로 수행되었습니다.                                     |
| 재발 방지 | VM 생성 후 Network Adapter와 Port Group 연결 상태를 먼저 확인한 뒤 OS 설치 및 IP 설정을 진행하도록 정리했습니다. |

#### 문제 4. `/opt/monitoring` 디렉터리 권한 문제

| 구분    | 내용                                                                   |
| ----- | -------------------------------------------------------------------- |
| 현상    | Docker Compose 설정 파일 생성 또는 데이터 디렉터리 생성 시 권한 오류가 발생했습니다.              |
| 원인    | `/opt/monitoring` 디렉터리 소유자가 root로 되어 있어 `fisa` 사용자가 쓰기 권한을 갖지 못했습니다. |
| 해결 과정 | `sudo chown -R fisa:fisa /opt/monitoring` 명령어를 실행했습니다.               |
| 결과    | VM별 작업 디렉터리와 설정 파일을 정상적으로 생성할 수 있었습니다.                               |
| 재발 방지 | 공통 디렉터리 생성 직후 소유권 변경을 필수 절차로 포함했습니다.                                 |
