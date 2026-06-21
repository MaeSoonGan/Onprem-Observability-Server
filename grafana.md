# Grafana Dashboard 구성

## 1. 개요 및 프로젝트 내 역할

- Grafana는 MaeSoonGan Observability 서버에서 Metric, Log, Trace를 통합 시각화하는 Dashboard 도구입니다.

- 본 프로젝트에서는 Monitoring UI VM에 Grafana를 구성하여 Prometheus, Loki, Tempo를 Data Source로 연결했습니다. Prometheus는 Metric, Loki는 Log, Tempo는 Trace 데이터를 제공하고, Grafana는 이를 Dashboard와 Explore 화면에서 조회할 수 있도록 합니다.

- 운영자는 Grafana를 통해 On-Premise Main 서버, DR 서버, Monitoring 서버, AWS/EKS 영역의 상태를 한 화면에서 확인할 수 있습니다. 또한 Alertmanager와 연계된 알림 상태를 확인하여 장애 발생 시 어떤 서버 또는 서비스에서 문제가 발생했는지 빠르게 파악할 수 있습니다.

</br>

## 2. 구성 환경 및 사양

| 구분              | 내용                              |
| --------------- | ------------------------------- |
| VM              | Monitoring UI VM                |
| 역할              | Dashboard 시각화                   |
| Hypervisor      | VMware ESXi                     |
| OS              | Ubuntu Server                   |
| vCPU            | 2 vCPU                          |
| RAM             | 2~3GB                           |
| Disk            | 20GB                            |
| IP Address      | 10.6.10.11/24                   |
| Gateway         | 10.6.10.1                       |
| 작업 디렉터리         | /opt/monitoring/ui              |
| Data Directory  | /opt/monitoring/ui/grafana-data |
| Service Port    | 3000                            |
| Container Image | grafana/grafana:latest          |
| 기본 관리자 계정       | admin                           |
| 기본 관리자 비밀번호     | VMware1!                        |
| 연결 Data Source  | Prometheus, Loki, Tempo         |

- Grafana는 Dashboard UI를 제공하는 구성 요소이므로 Monitoring UI VM에 Alertmanager와 함께 배치했습니다.

</br>

## 3. 선택 이유 및 대안 비교

### 선택 이유

- Grafana를 선택한 이유는 Prometheus, Loki, Tempo와의 연동성이 높고, Metric, Log, Trace를 하나의 UI에서 확인할 수 있기 때문입니다.

- 본 프로젝트는 Observability 데이터를 통합적으로 수집하고 장애 상황을 빠르게 파악하는 것이 목표였습니다. Grafana는 Dashboard, Explore, Variable, Panel 기능을 제공하여 서버별 상태, 서비스별 메트릭, 로그, Trace를 운영 대상별로 구성하기에 적합했습니다.

### 대안 비교

| 대안                   | 장점                                                    | 한계                                                   | 최종 판단      |
| -------------------- | ----------------------------------------------------- | ---------------------------------------------------- | ---------- |
| Grafana              | Prometheus, Loki, Tempo 연동이 자연스럽고 Dashboard 구성이 쉽습니다. | Dashboard 설계를 직접 구성해야 합니다.                           | 최종 선택했습니다. |
| Kibana               | Elasticsearch 기반 로그 분석에 강합니다.                         | Prometheus, Tempo 중심 구조와는 연동성이 약합니다.                 | 제외했습니다.    |
| CloudWatch Dashboard | AWS 리소스 시각화에 적합합니다.                                   | On-Premise Metric, Log, Trace 통합 시각화에는 추가 구성이 필요합니다. | 제외했습니다.    |
| Prometheus UI        | Metric 조회가 가능합니다.                                     | Dashboard, Log, Trace 통합 시각화에는 한계가 있습니다.             | 제외했습니다.    |

</br>

## 4. 설치 및 주요 설정

### 4-1. 작업 디렉터리 생성

- Monitoring UI VM에서 Grafana와 Alertmanager 실행을 위한 작업 디렉터리를 생성합니다.

```
sudo mkdir -p /opt/monitoring/ui
sudo chown -R fisa:fisa /opt/monitoring
cd /opt/monitoring/ui
mkdir -p grafana-data alertmanager-data
```

- `grafana-data`는 Grafana Dashboard, Data Source, 사용자 설정 등을 저장하는 디렉터리입니다.

### 4-2. Docker Compose 설정

- Grafana는 Alertmanager와 함께 Docker Compose로 실행합니다.

```
nano compose.yml
```

- Grafana 서비스 설정은 다음과 같습니다.

```
services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=VMware1!
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - ./grafana-data:/var/lib/grafana
```

- 주요 설정 의미는 다음과 같습니다.

| 설정                              | 설명                        |
| ------------------------------- | ------------------------- |
| `3000:3000`                     | Grafana Web UI 접근 포트입니다.  |
| `GF_SECURITY_ADMIN_USER`        | 초기 관리자 계정입니다.             |
| `GF_SECURITY_ADMIN_PASSWORD`    | 초기 관리자 비밀번호입니다.           |
| `GF_USERS_ALLOW_SIGN_UP=false`  | 임의 사용자 가입을 비활성화합니다.       |
| `grafana-data:/var/lib/grafana` | Dashboard와 설정 데이터를 보존합니다. |

### 4-3. 데이터 디렉터리 권한 설정

- Grafana 컨테이너는 일반적으로 UID 472를 사용하므로 데이터 디렉터리 권한을 맞춥니다.

```
sudo chown -R 472:472 grafana-data
```

- Docker Compose 문법을 확인합니다.

```
docker compose config
```

- Grafana와 Alertmanager를 실행합니다.

```
docker compose up -d
```

- 컨테이너와 포트 상태를 확인합니다.

```
docker ps
sudo ss -lntp | grep -E '3000|9093'
```

- Grafana는 다음 주소로 접속합니다.

```
http://10.6.10.11:3000
```

- 초기 로그인 정보는 다음과 같습니다.

| 항목 | 값        |
| -- | -------- |
| ID | admin    |
| PW | VMware1! |

### 4-4. Data Source 연결

- Grafana에서 `Connections` → `Data sources` → `Add data source`로 이동하여 Prometheus, Loki, Tempo를 연결합니다.

| Type       | URL                    | 용도        |
| ---------- | ---------------------- | --------- |
| Prometheus | http://10.6.10.22:9090 | Metric 조회 |
| Loki       | http://10.6.10.33:3100 | Log 조회    |
| Tempo      | http://10.6.10.33:3200 | Trace 조회  |

- 각 Data Source를 추가한 뒤 `Save & test`를 실행하여 연결 상태를 확인합니다.

### 4-5. Explore 확인

- Prometheus는 Grafana Explore에서 다음 Query로 확인합니다.

```
up
```

- 결과가 조회되면 Prometheus Data Source 연결이 정상입니다.

- Loki는 초기에는 로그를 보내는 Agent가 없으면 결과가 없을 수 있습니다. 이 경우 Data Source 연결만 성공해도 초기 단계에서는 정상입니다.

- Tempo 역시 아직 Trace를 보내는 애플리케이션이 없다면 결과가 없을 수 있습니다. 이후 Spring Boot 서비스에 OpenTelemetry Java Agent와 Alloy OTLP Receiver를 구성한 뒤 Trace를 확인합니다.

### 4-6. Dashboard 구성

- Grafana Dashboard는 운영 대상별로 폴더를 나누어 구성했습니다.

| 폴더                   | 용도                    |
| -------------------- | --------------------- |
| 00-Overview          | 전체 시스템 요약             |
| 01-Main-Server       | On-Premise Main 서버 상태 |
| 02-DR-Server         | DR 서버 상태              |
| 03-Monitoring-Server | Observability 서버 상태   |
| 04-AWS/EKS           | AWS 및 EKS 상태          |
| 05-Alert             | 알림 및 장애 상태            |

- Dashboard에서는 Variable을 생성하여 VM Host를 선택할 수 있도록 구성했습니다. 이를 통해 동일한 Panel Query를 사용하더라도 Host별 상태를 필터링할 수 있습니다.

- 주요 Panel 예시는 다음과 같습니다.

| Panel         | Data Source               | 목적                  |
| ------------- | ------------------------- | ------------------- |
| VM UP         | Prometheus                | VM 생존 여부 확인         |
| CPU Usage     | Prometheus                | CPU 사용률 확인          |
| Memory Usage  | Prometheus                | 메모리 사용률 확인          |
| Disk Usage    | Prometheus                | 디스크 사용률 확인          |
| Recent Logs   | Loki                      | 최근 시스템/애플리케이션 로그 확인 |
| Recent Traces | Tempo                     | 최근 요청 Trace 확인      |
| Alert Status  | Prometheus / Alertmanager | 현재 알림 상태 확인         |

### 4-7. 설정 파일 백업

- 정상 기동 후 설정 파일을 백업합니다.

```
cd /opt/monitoring/ui
cp compose.yml compose.yml.bak
cp alertmanager.yml alertmanager.yml.bak
```

- Grafana Dashboard는 Export 기능을 사용하여 JSON 형태로 백업할 수 있습니다.

</br>

## 5. 동작 흐름 및 트러블슈팅

### 동작 흐름

1. Prometheus가 각 VM과 서비스의 Metric을 수집합니다.
2. Loki가 Alloy로부터 전달받은 Log를 저장합니다.
3. Tempo가 OpenTelemetry 또는 Alloy로부터 전달받은 Trace를 저장합니다.
4. Grafana는 Prometheus, Loki, Tempo를 Data Source로 연결합니다.
5. 운영자는 Dashboard에서 전체 시스템 상태를 확인합니다.
6. 이상 징후가 발생하면 Alert Dashboard와 Slack 알림을 확인합니다.
7. 이후 Metric, Log, Trace를 함께 조회하여 원인을 분석합니다.

### 트러블슈팅

#### 문제 1. Grafana Web UI에 접속되지 않는 문제

| 구분    | 내용                                                          |                                   |
| ----- | ----------------------------------------------------------- | --------------------------------- |
| 현상    | `http://10.6.10.11:3000`으로 접속되지 않았습니다.                      |                                   |
| 원인    | Grafana 컨테이너가 정상 실행되지 않았거나, 3000 포트가 열리지 않았을 가능성이 있었습니다.    |                                   |
| 해결 과정 | `docker ps`, `docker ps -a`, `sudo ss -lntp                 | grep 3000`으로 컨테이너와 포트 상태를 확인했습니다. |
| 결과    | Grafana 컨테이너가 정상 실행된 뒤 Web UI에 접속할 수 있었습니다.                 |                                   |
| 재발 방지 | Docker Compose 실행 후 컨테이너 상태와 포트 Listen 상태를 함께 확인하도록 정리했습니다. |                                   |

#### 문제 2. Grafana 데이터 디렉터리 권한 문제

| 구분    | 내용                                                          |
| ----- | ----------------------------------------------------------- |
| 현상    | Grafana 컨테이너가 정상 실행되지 않거나 Dashboard 설정이 저장되지 않았습니다.         |
| 원인    | `grafana-data` 디렉터리의 소유권이 Grafana 컨테이너 실행 사용자와 맞지 않았습니다.    |
| 해결 과정 | `sudo chown -R 472:472 grafana-data` 명령어로 권한을 수정했습니다.       |
| 결과    | Grafana가 정상 실행되었고 Dashboard와 Data Source 설정이 저장되었습니다.       |
| 재발 방지 | Docker Compose 실행 전에 Grafana 데이터 디렉터리 권한을 먼저 설정하도록 절차화했습니다. |

#### 문제 3. Data Source 연결 실패

| 구분    | 내용                                                                                              |
| ----- | ----------------------------------------------------------------------------------------------- |
| 현상    | Grafana에서 Prometheus, Loki, Tempo Data Source의 `Save & test`가 실패했습니다.                           |
| 원인    | 대상 서비스가 실행되지 않았거나 URL, 포트, 네트워크 경로가 잘못되었을 가능성이 있었습니다.                                           |
| 해결 과정 | Prometheus `10.6.10.22:9090`, Loki `10.6.10.33:3100`, Tempo `10.6.10.33:3200` 포트 접근 여부를 확인했습니다. |
| 결과    | 각 서비스 컨테이너와 포트 상태를 정상화한 뒤 Data Source 연결이 성공했습니다.                                               |
| 재발 방지 | Data Source 등록 전 각 서비스의 Ready Endpoint와 포트 Listen 상태를 먼저 확인하도록 정리했습니다.                          |

#### 문제 4. Dashboard에 데이터가 표시되지 않는 문제

| 구분    | 내용                                                                                        |
| ----- | ----------------------------------------------------------------------------------------- |
| 현상    | Dashboard Panel은 생성되었지만 데이터가 표시되지 않았습니다.                                                  |
| 원인    | Prometheus Target이 DOWN이거나, Label 조건이 실제 수집 데이터와 일치하지 않았습니다.                              |
| 해결 과정 | Prometheus에서 `up` Query를 확인하고, Grafana Variable Preview에서 Host Label이 정상적으로 표시되는지 확인했습니다. |
| 결과    | Label 조건을 실제 수집 데이터와 맞춘 뒤 Dashboard에 VM 상태가 표시되었습니다.                                      |
| 재발 방지 | Panel 작성 전 Prometheus Explore에서 Query 결과를 먼저 확인하고 Dashboard에 적용하도록 정리했습니다.                |
