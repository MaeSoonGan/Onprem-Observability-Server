# Prometheus 메트릭 수집 환경 구성

## 1. 개요 및 프로젝트 내 역할

- Prometheus는 MaeSoonGan Observability 서버에서 Metric 수집과 Alert Rule 평가를 담당하는 메트릭 수집 서버입니다.

- 본 프로젝트에서는 Monitoring Metric VM에 Prometheus를 구성하여 Monitoring UI VM, Monitoring Metric VM, Monitoring Log/Trace VM의 시스템 메트릭을 수집했습니다. 이후 Alloy를 각 VM에 설치한 뒤, Prometheus가 Alloy의 Unix Exporter Endpoint를 주기적으로 Scrape하여 CPU, Memory, Disk, Network 등의 인프라 메트릭을 저장하도록 구성했습니다.

- 또한 Prometheus는 Grafana의 Data Source로 연결되어 대시보드 시각화에 사용되며, Alert Rule 조건이 만족되면 Alertmanager로 알림을 전달하는 역할도 수행합니다. 따라서 Prometheus는 Observability 구조에서 메트릭 저장소이자 장애 감지의 기준점 역할을 합니다.

</br>

## 2. 구성 환경 및 사양

| 구분                  | 내용                                         |
| ------------------- | ------------------------------------------ |
| VM                  | Monitoring Metric VM                       |
| 역할                  | Prometheus 기반 Metric 수집                    |
| Hypervisor          | VMware ESXi                                |
| OS                  | Ubuntu Server                              |
| vCPU                | 2 vCPU                                     |
| RAM                 | 5GB                                        |
| Disk                | 60GB                                       |
| IP Address          | 10.6.10.22/24                              |
| Gateway             | 10.6.10.1                                  |
| 작업 디렉터리             | /opt/monitoring/prometheus                 |
| Data Directory      | /opt/monitoring/prometheus/prometheus-data |
| Service Port        | 9090                                       |
| Container Image     | prom/prometheus:latest                     |
| Retention           | 7d                                         |
| Alertmanager Target | 10.6.10.11:9093                            |

- Prometheus는 메트릭 데이터를 일정 기간 저장해야 하므로 Monitoring UI VM보다 더 큰 디스크를 할당했습니다. 본 프로젝트에서는 실습 환경의 리소스 제약을 고려하여 보관 기간을 7일로 설정했습니다.

</br>

## 3. 선택 이유 및 대안 비교

### 선택 이유

- Prometheus를 선택한 이유는 Kubernetes, Linux VM, Exporter, Alertmanager와의 연동성이 높고, PromQL을 통해 메트릭 기반 장애 조건을 정의하기 쉽기 때문입니다.

- 본 프로젝트에서는 AWS와 On-Premise 환경의 메트릭을 통합적으로 수집하고, Grafana Dashboard와 Slack Alert로 연결해야 했습니다. Prometheus는 Pull 방식으로 수집 대상을 주기적으로 Scrape하고, Alert Rule을 평가하여 Alertmanager로 전달할 수 있어 Metric 수집과 장애 감지 흐름을 단순하게 구성할 수 있었습니다.

### 대안 비교

| 대안         | 장점                                                | 한계                                                   | 최종 판단      |
| ---------- | ------------------------------------------------- | ---------------------------------------------------- | ---------- |
| Prometheus | Exporter 생태계가 넓고 Alertmanager, Grafana와 연동이 쉽습니다. | 장기 보관과 대규모 분산 저장은 별도 구성이 필요합니다.                      | 최종 선택했습니다. |
| Zabbix     | 전통적인 서버 모니터링에 강합니다.                               | Kubernetes, Exporter 기반 구성과는 Prometheus보다 덜 자연스럽습니다. | 제외했습니다.    |
| Datadog    | SaaS 기반 통합 관측성이 강력합니다.                            | 실습 프로젝트 환경에서는 비용과 외부 의존성이 부담됩니다.                     | 제외했습니다.    |

</br>

## 4. 설치 및 주요 설정

### 4-1. 작업 디렉터리 생성

- Prometheus 설정 파일과 데이터 디렉터리를 관리하기 위해 Monitoring Metric VM에 작업 디렉터리를 생성합니다.

```
mkdir -p /opt/monitoring/prometheus
cd /opt/monitoring/prometheus
mkdir -p prometheus-data
```

- `prometheus-data` 디렉터리는 Prometheus가 수집한 시계열 메트릭 데이터를 저장하는 경로입니다.

### 4-2. Prometheus 초기 설정 파일 생성

- `prometheus.yml` 파일을 생성하고 기본 Scrape 설정을 작성합니다.

```
nano prometheus.yml
```

- 초기 설정에서는 Prometheus 자기 자신과 Monitoring 서버 VM들을 수집 대상으로 등록합니다.

```
global:
  scrape_interval: 60s
  evaluation_interval: 60s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "monitoring-ui"
    static_configs:
      - targets: ["10.6.10.11:9100"]

  - job_name: "monitoring-metric"
    static_configs:
      - targets: ["10.6.10.22:9100"]

  - job_name: "monitoring-logtrace"
    static_configs:
      - targets: ["10.6.10.33:9100"]
```

- 초기에는 Node Exporter 방식의 Target을 기준으로 작성했지만, 이후 Alloy를 사용하면서 Scrape Target을 Alloy Endpoint 기준으로 변경했습니다.

### 4-3. Docker Compose 파일 작성

- Prometheus를 Docker Compose 기반으로 실행하기 위해 `compose.yml` 파일을 작성합니다.

```
nano compose.yml
```

- Compose 설정은 다음과 같이 구성합니다.

```
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=7d"
      - "--web.enable-lifecycle"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus-data:/prometheus
```

- 주요 설정 의미는 다음과 같습니다.

| 설정                                 | 설명                                    |
| ---------------------------------- | ------------------------------------- |
| `9090:9090`                        | Prometheus Web UI 및 API 포트입니다.        |
| `--config.file`                    | Prometheus 설정 파일 경로를 지정합니다.           |
| `--storage.tsdb.path`              | 메트릭 데이터 저장 경로를 지정합니다.                 |
| `--storage.tsdb.retention.time=7d` | 메트릭 보관 기간을 7일로 제한합니다.                 |
| `--web.enable-lifecycle`           | 설정 변경 후 HTTP Reload를 사용할 수 있도록 설정합니다. |

### 4-4. Prometheus 실행 및 확인

- Docker Compose로 Prometheus를 실행합니다.

```
docker compose up -d
```

- 컨테이너 상태를 확인합니다.

```
docker ps
```

- Prometheus Ready 상태를 확인합니다.

```
curl http://localhost:9090/-/ready
```

- 브라우저에서는 다음 주소로 접속합니다.

```
http://10.6.10.22:9090
```

### 4-5. Alertmanager 연동 설정

- Prometheus에서 발생한 Alert를 Alertmanager로 전달하기 위해 `prometheus.yml`에 Alertmanager Target을 추가합니다.

```
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - "10.6.10.11:9093"
```

- 설정 반영 후 Prometheus를 Reload합니다.

```
curl -X POST http://localhost:9090/-/reload
```

- 로그를 통해 오류 여부를 확인합니다.

```
docker logs prometheus --tail=50
```

- Alertmanager 연결은 Alert Rule 작성 이후 Prometheus Alert 페이지와 Alertmanager UI에서 확인할 수 있습니다.

### 4-6. Alloy 기반 Metric 수집 설정

- 각 VM에 Alloy를 설치한 뒤에는 Prometheus가 Alloy의 Unix Exporter Endpoint를 Scrape하도록 설정을 변경합니다.

- Monitoring 서버 3대의 Alloy Unix Exporter 수집 Job 예시는 다음과 같습니다.

```
- job_name: "alloy-unix-monitoring"
  metrics_path: "/api/v0/component/prometheus.exporter.unix.local/metrics"
  static_configs:
    - targets:
        - "10.6.10.11:12345"
      labels:
        area: "monitoring-server"
        host: "fs-monitoring-ui"
        instance: "10.6.10.11"

    - targets:
        - "10.6.10.22:12345"
      labels:
        area: "monitoring-server"
        host: "fs-monitoring-metric"
        instance: "10.6.10.22"

    - targets:
        - "10.6.10.33:12345"
      labels:
        area: "monitoring-server"
        host: "fs-monitoring-logtrace"
        instance: "10.6.10.33"
```

- 이 설정을 통해 Prometheus는 Alloy가 노출하는 Linux 시스템 메트릭을 수집합니다. `area`, `host`, `instance` Label을 부여하여 Grafana Dashboard에서 서버별로 필터링할 수 있도록 구성했습니다.

### 4-7. Observability 구성 요소 Metric 수집

- Prometheus는 Monitoring 서버 자체뿐 아니라 Grafana, Alertmanager, Loki, Tempo의 상태도 수집하도록 구성했습니다.

| Job          | Target          | 설명                 |
| ------------ | --------------- | ------------------ |
| grafana      | 10.6.10.11:3000 | Grafana 상태 수집      |
| alertmanager | 10.6.10.11:9093 | Alertmanager 상태 수집 |
| loki         | 10.6.10.33:3100 | Loki 상태 수집         |
| tempo        | 10.6.10.33:3200 | Tempo 상태 수집        |

- 각 Target에는 `area`, `host`, `service` Label을 지정하여 어떤 서버의 어떤 서비스인지 구분할 수 있도록 했습니다.

- 설정 변경 후에는 다음 명령어로 적용합니다.

```
curl -X POST http://localhost:9090/-/reload
```

</br>

## 5. 동작 흐름 및 트러블슈팅

### 동작 흐름

1. Monitoring UI VM, Metric VM, Log/Trace VM에서 Alloy가 시스템 메트릭을 노출합니다.
2. Prometheus는 60초 간격으로 각 Alloy Endpoint를 Scrape합니다.
3. Prometheus는 수집한 Metric을 로컬 TSDB에 저장합니다.
4. Grafana는 Prometheus를 Data Source로 연결하여 Dashboard에서 Metric을 조회합니다.
5. Prometheus는 설정된 Alert Rule을 주기적으로 평가합니다.
6. 장애 조건이 만족되면 Prometheus가 Alertmanager로 Alert를 전달합니다.
7. Alertmanager는 알림을 그룹화하고 Slack으로 전송합니다.
8. 운영자는 Slack 알림을 확인한 뒤 Grafana Dashboard와 Prometheus UI에서 상세 상태를 확인합니다.

### 트러블슈팅

#### 문제 1. Prometheus 컨테이너가 계속 Restarting 상태가 되는 문제

| 구분     | 내용                                                                                                                                                         |
| ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 현상     | Prometheus 컨테이너가 9090 포트를 정상적으로 열지 못하고 계속 Restarting 상태가 되었습니다.                                                                                            |
| 원인     | Prometheus 컨테이너가 데이터 디렉터리인 `prometheus-data`에 파일을 생성할 권한이 없어 실행 직후 종료되었습니다.                                                                                |
| 해결 과정  | `docker logs prometheus` 명령어로 로그를 확인했고, `prometheus-data` 디렉터리에서 `permission denied`가 발생하는 것을 확인했습니다. 이후 Prometheus 컨테이너 사용자와 디렉터리 소유자를 맞추기 위해 권한을 수정했습니다. |
| 해결 명령어 | `sudo chown -R 65534:65534 prometheus-data`                                                                                                                |
| 결과     | Prometheus 컨테이너가 정상적으로 실행되었고, 9090 포트로 Web UI에 접근할 수 있었습니다.                                                                                                |
| 재발 방지  | Prometheus 실행 전 데이터 디렉터리 소유권을 확인하고, Docker Compose 실행 전에 권한 설정을 먼저 적용하도록 정리했습니다.                                                                           |

#### 문제 2. 설정 변경 후 Prometheus에 반영되지 않는 문제

| 구분     | 내용                                                                                                       |
| ------ | -------------------------------------------------------------------------------------------------------- |
| 현상     | `prometheus.yml`을 수정했지만 Prometheus UI에서 Target 또는 Alertmanager 설정이 갱신되지 않았습니다.                           |
| 원인     | 설정 파일 수정 후 Prometheus Reload를 수행하지 않았습니다.                                                                |
| 해결 과정  | `--web.enable-lifecycle` 옵션이 활성화되어 있는지 확인한 뒤 HTTP Reload를 실행했습니다.                                        |
| 해결 명령어 | `curl -X POST http://localhost:9090/-/reload`                                                            |
| 결과     | 수정한 Scrape Target과 Alertmanager 설정이 정상 반영되었습니다.                                                          |
| 재발 방지  | `prometheus.yml` 수정 후에는 반드시 Reload 명령어를 실행하고, `docker logs prometheus --tail=50`으로 오류 여부를 확인하도록 절차화했습니다. |

#### 문제 3. Alloy Target이 DOWN으로 표시되는 문제

| 구분    | 내용                                                                                                                                                |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| 현상    | Prometheus Targets 화면에서 Alloy 수집 대상이 DOWN으로 표시되었습니다.                                                                                              |
| 원인    | Alloy가 12345 포트로 외부 요청을 수신하지 않거나, Prometheus의 `metrics_path`가 Alloy Endpoint와 일치하지 않았을 가능성이 있었습니다.                                                |
| 해결 과정 | 각 VM에서 Alloy가 `0.0.0.0:12345`로 Listen 중인지 확인하고, Prometheus 설정의 `metrics_path`를 `/api/v0/component/prometheus.exporter.unix.local/metrics`로 맞췄습니다. |
| 결과    | Prometheus Targets 화면에서 Monitoring 서버 VM들의 Target 상태가 UP으로 표시되었습니다.                                                                               |
| 재발 방지 | Alloy 설치 후 Prometheus Target에 등록하기 전에 `curl http://VM_IP:12345`로 Alloy Endpoint 접근 여부를 먼저 확인하도록 정리했습니다.                                           |

#### 문제 4. Alertmanager로 알림이 전달되지 않는 문제

| 구분    | 내용                                                                                                                       |
| ----- | ------------------------------------------------------------------------------------------------------------------------ |
| 현상    | Prometheus에서 Alert Rule은 동작하지만 Alertmanager로 알림이 전달되지 않았습니다.                                                             |
| 원인    | `prometheus.yml`의 Alertmanager Target 설정이 누락되었거나, Alertmanager 주소인 `10.6.10.11:9093`에 접근할 수 없는 상태였습니다.                   |
| 해결 과정 | `alerting.alertmanagers` 설정을 추가하고, Prometheus VM에서 `curl http://10.6.10.11:9093/-/ready`로 Alertmanager 접근 가능 여부를 확인했습니다. |
| 결과    | Prometheus Alert가 Alertmanager로 정상 전달되었습니다.                                                                              |
| 재발 방지 | Alertmanager 연동 후 Prometheus `/status`, `/alerts` 페이지와 Alertmanager UI를 함께 확인하도록 정리했습니다.                                 |
