# Alertmanager 알림 구성

## 1. 개요 및 프로젝트 내 역할

- Alertmanager는 MaeSoonGan Observability 서버에서 Prometheus Alert를 수신하고 Slack으로 전달하는 알림 관리 도구입니다.

- 본 프로젝트에서는 Prometheus가 Metric 기반 Alert Rule을 평가하고, 장애 조건이 만족되면 Alertmanager로 알림을 전달하도록 구성했습니다. Alertmanager는 수신한 알림을 Severity, Host, Service 기준으로 그룹화하고, 중복 알림을 줄인 뒤 Slack Webhook으로 전송합니다.

- 이를 통해 운영자는 Grafana Dashboard를 계속 보고 있지 않아도 Slack 알림을 통해 장애 발생 여부를 빠르게 인지할 수 있습니다.

</br>

## 2. 구성 환경 및 사양

| 구분              | 내용                                   |
| --------------- | ------------------------------------ |
| VM              | Monitoring UI VM                     |
| 역할              | Alert 수신, 그룹화, 중복 제거, Slack 전송       |
| Hypervisor      | VMware ESXi                          |
| OS              | Ubuntu Server                        |
| vCPU            | 2 vCPU                               |
| RAM             | 2~3GB                                |
| Disk            | 20GB                                 |
| IP Address      | 10.6.10.11/24                        |
| Gateway         | 10.6.10.1                            |
| 작업 디렉터리         | /opt/monitoring/ui                   |
| Data Directory  | /opt/monitoring/ui/alertmanager-data |
| Config File     | /opt/monitoring/ui/alertmanager.yml  |
| Service Port    | 9093                                 |
| Container Image | prom/alertmanager:latest             |
| 알림 수신           | Prometheus                           |
| 알림 전송           | Slack Incoming Webhook               |

</br>

## 3. 선택 이유 및 대안 비교

### 선택 이유

- Alertmanager를 선택한 이유는 Prometheus Alert Rule과 가장 자연스럽게 연동되며, 알림 그룹화, 라우팅, 중복 억제 기능을 제공하기 때문입니다.

- 본 프로젝트에서는 서버 Down, Disk 사용률 증가, CPU/Memory 사용률 증가, Loki/Tempo 장애 등 Metric 기반 장애 조건을 Prometheus에서 판단했습니다. Alertmanager는 이 알림을 Slack으로 전달하고, Critical 알림과 Warning 알림을 구분하여 운영자가 장애 우선순위를 빠르게 판단할 수 있도록 합니다.

### 대안 비교

| 대안                  | 장점                                            | 한계                                    | 최종 판단             |
| ------------------- | --------------------------------------------- | ------------------------------------- | ----------------- |
| Alertmanager        | Prometheus와 연동이 쉽고 알림 그룹화, 라우팅, 중복 억제가 가능합니다. | Prometheus Alert 기반 구성에 최적화되어 있습니다.   | 최종 선택했습니다.        |
| Grafana Alerting    | Grafana UI에서 알림을 관리할 수 있습니다.                  | Prometheus Rule 기반 알림 구조와 분리될 수 있습니다. | 보조 확인 용도로 사용했습니다. |
| Slack Webhook 직접 호출 | 구성이 단순합니다.                                    | 알림 그룹화와 중복 제거가 어렵습니다.                 | 제외했습니다.           |
| Email 알림            | 범용성이 높습니다.                                    | 실시간 대응성은 Slack보다 낮습니다.                | 제외했습니다.           |

</br>

## 4. 설치 및 주요 설정

### 4-1. 작업 디렉터리 생성

- Monitoring UI VM에서 Alertmanager 설정과 데이터를 관리할 디렉터리를 생성합니다.

```
sudo mkdir -p /opt/monitoring/ui
sudo chown -R fisa:fisa /opt/monitoring
cd /opt/monitoring/ui
mkdir -p grafana-data alertmanager-data
```

### 4-2. 초기 Alertmanager 설정 파일 작성

- 초기에는 최소 설정으로 Alertmanager를 기동합니다.

```
nano alertmanager.yml
```

- 초기 설정은 다음과 같습니다.

```
global:
  resolve_timeout: 5m

route:
  receiver: "default"

receivers:
  - name: "default"
```

- 이후 Slack Webhook 연동 단계에서 Receiver와 Route를 확장합니다.

### 4-3. Docker Compose 설정

- Alertmanager는 Grafana와 함께 Docker Compose로 실행합니다.

```
nano compose.yml
```

- Alertmanager 서비스 설정은 다음과 같습니다.

```
services:
  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: unless-stopped
    ports:
      - "9093:9093"
    command:
      - "--config.file=/etc/alertmanager/alertmanager.yml"
      - "--storage.path=/alertmanager"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - ./alertmanager-data:/alertmanager
```

- 주요 설정 의미는 다음과 같습니다.

| 설정                                | 설명                               |
| --------------------------------- | -------------------------------- |
| `9093:9093`                       | Alertmanager Web UI 및 API 포트입니다. |
| `--config.file`                   | Alertmanager 설정 파일 경로입니다.        |
| `--storage.path`                  | 알림 상태 저장 경로입니다.                  |
| `alertmanager.yml` Mount          | 호스트 설정 파일을 컨테이너 내부로 연결합니다.       |
| `alertmanager-data:/alertmanager` | Alertmanager 상태 데이터를 보존합니다.      |

### 4-4. 데이터 디렉터리 권한 설정

- Alertmanager 데이터 디렉터리의 권한을 설정합니다.

```
sudo chown -R 65534:65534 alertmanager-data
```

- Docker Compose 문법을 확인합니다.

```
docker compose config
```

- 컨테이너를 실행합니다.

```
docker compose up -d
```

- 상태를 확인합니다.

```
docker ps
sudo ss -lntp | grep -E '3000|9093'
```

- Alertmanager UI는 다음 주소로 접속합니다.

```
http://10.6.10.11:9093
```

### 4-5. Prometheus와 Alertmanager 연결

- Prometheus VM의 `prometheus.yml`에 Alertmanager 주소를 추가합니다.

```
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - "10.6.10.11:9093"
```

- 설정 변경 후 Prometheus를 Reload합니다.

```
curl -X POST http://localhost:9090/-/reload
```

- Prometheus VM에서 Alertmanager Ready 상태를 확인합니다.

```
curl http://10.6.10.11:9093/-/ready
```

- Prometheus UI에서는 다음 화면에서 Alert 상태를 확인합니다.

| 화면                  | 주소                              |
| ------------------- | ------------------------------- |
| Prometheus Alerts   | http://10.6.10.22:9090/alerts   |
| Alertmanager Alerts | http://10.6.10.11:9093/#/alerts |

### 4-6. Prometheus Alert Rule 작성

- Prometheus VM에서 Alert Rule 파일을 작성합니다.

```
nano alert-rules.yml
```

- 주요 Rule은 다음과 같이 구성했습니다.

| Rule                         | 목적                         |
| ---------------------------- | -------------------------- |
| MonitoringTargetDown         | 수집 대상 Down 감지              |
| MonitoringDiskUsageHigh      | Disk 사용률 80% 초과 감지         |
| MonitoringDiskUsageCritical  | Disk 사용률 90% 초과 감지         |
| MonitoringMemoryUsageHigh    | Memory 사용률 85% 초과 감지       |
| MonitoringCPUUsageHigh       | CPU 사용률 80% 초과 감지          |
| PrometheusConfigReloadFailed | Prometheus 설정 Reload 실패 감지 |
| LokiDown                     | Loki 장애 감지                 |
| TempoDown                    | Tempo 장애 감지                |

- Prometheus 설정에 Rule 파일을 추가합니다.

```
rule_files:
  - "/etc/prometheus/alert-rules.yml"
```

- Docker Compose에서 Rule 파일 Mount도 추가합니다.

```
volumes:
  - ./prometheus.yml:/etc/prometheus/prometheus.yml
  - ./alert-rules.yml:/etc/prometheus/alert-rules.yml
  - ./prometheus-data:/prometheus
```

- Rule 문법을 확인합니다.

```
docker exec prometheus promtool check rules /etc/prometheus/alert-rules.yml
```

- Prometheus를 Reload합니다.

```
curl -X POST http://localhost:9090/-/reload
```

### 4-7. Slack Webhook 연동

- Slack에서 알림 수신용 채널을 생성하고 Incoming Webhook을 추가합니다.

- Webhook URL을 발급받은 뒤 임의의 VM에서 테스트 메시지를 전송합니다.

```
curl -X POST -H 'Content-type: application/json' --data '{"text":"FISA monitoring alert test"}' '복사한_Webhook_URL'
```

- 이후 `alertmanager.yml`을 Slack 연동 설정으로 변경합니다.

```
global:
  resolve_timeout: 5m

route:
  receiver: "slack-warning"
  group_by: ["alertname", "area", "host", "service"]
  group_wait: 10s
  group_interval: 1m
  repeat_interval: 1h
  routes:
    - matchers:
        - severity="critical"
      receiver: "slack-critical"
    - matchers:
        - severity="warning"
      receiver: "slack-warning"

receivers:
  - name: "slack-critical"
    slack_configs:
      - api_url: "여기에_Slack_Incoming_Webhook_URL"
        channel: "#fisa-alerts"
        send_resolved: true

  - name: "slack-warning"
    slack_configs:
      - api_url: "여기에_Slack_Incoming_Webhook_URL"
        channel: "#fisa-alerts"
        send_resolved: true

inhibit_rules:
  - source_matchers:
      - severity="critical"
    target_matchers:
      - severity="warning"
    equal: ["alertname", "host", "service"]
```

- 설정 변경 후 Alertmanager를 재기동합니다.

```
docker compose up -d
docker logs alertmanager --tail=50
```

### 4-8. 테스트 Alert 작성

- 실제 장애를 만들기 전에 테스트용 Alert Rule을 작성하여 Slack 알림 흐름을 확인합니다.

```
- name: test.rules
  rules:
    - alert: TestAlwaysFiring
      expr: vector(1)
      for: 30s
      labels:
        severity: warning
        area: monitoring-server
        service: test
      annotations:
        summary: "Test alert is firing"
        description: "This is a test alert for Alertmanager notification."
```

- 수정 후 Rule을 검증하고 Reload합니다.

```
docker exec prometheus promtool check rules /etc/prometheus/alert-rules.yml
curl -X POST http://localhost:9090/-/reload
```

- 테스트가 끝나면 `TestAlwaysFiring` Rule은 삭제하고 다시 Reload합니다.

</br>

## 5. 동작 흐름 및 트러블슈팅

### 동작 흐름

1. Prometheus가 각 Target의 Metric을 수집합니다.
2. Prometheus가 Alert Rule을 주기적으로 평가합니다.
3. 장애 조건이 만족되면 Alert가 Firing 상태가 됩니다.
4. Prometheus가 Alertmanager로 Alert를 전달합니다.
5. Alertmanager가 Alert를 `alertname`, `area`, `host`, `service` 기준으로 그룹화합니다.
6. Severity에 따라 Critical 또는 Warning Receiver로 라우팅합니다.
7. Alertmanager가 Slack Webhook으로 알림을 전송합니다.
8. 운영자는 Slack 알림을 확인한 뒤 Grafana Dashboard에서 원인을 분석합니다.
9. 장애가 해소되면 Resolved 알림이 Slack으로 전송됩니다.

### 트러블슈팅

#### 문제 1. Alertmanager UI에 접속되지 않는 문제

| 구분    | 내용                                                                |                                   |
| ----- | ----------------------------------------------------------------- | --------------------------------- |
| 현상    | `http://10.6.10.11:9093`으로 접속되지 않았습니다.                            |                                   |
| 원인    | Alertmanager 컨테이너가 실행되지 않았거나 9093 포트가 열리지 않았습니다.                  |                                   |
| 해결 과정 | `docker ps`, `docker logs alertmanager --tail=50`, `sudo ss -lntp | grep 9093`으로 컨테이너와 포트 상태를 확인했습니다. |
| 결과    | 컨테이너 실행 상태를 정상화한 뒤 Alertmanager UI에 접속할 수 있었습니다.                  |                                   |
| 재발 방지 | Docker Compose 실행 후 컨테이너 상태와 포트 Listen 상태를 함께 확인하도록 정리했습니다.       |                                   |

#### 문제 2. Prometheus Alert가 Alertmanager로 전달되지 않는 문제

| 구분    | 내용                                                                               |
| ----- | -------------------------------------------------------------------------------- |
| 현상    | Prometheus에서는 Alert가 보이지만 Alertmanager UI에는 표시되지 않았습니다.                          |
| 원인    | `prometheus.yml`의 `alerting.alertmanagers` 설정이 누락되었거나 주소가 잘못되었습니다.               |
| 해결 과정 | Alertmanager Target을 `10.6.10.11:9093`으로 추가하고 Prometheus를 Reload했습니다.            |
| 결과    | Alertmanager UI에서 Prometheus Alert를 확인할 수 있었습니다.                                 |
| 재발 방지 | Alert Rule 작성 후 Prometheus `/alerts`와 Alertmanager `/#/alerts`를 함께 확인하도록 정리했습니다. |

#### 문제 3. Slack 알림이 오지 않는 문제

| 구분    | 내용                                                                                   |
| ----- | ------------------------------------------------------------------------------------ |
| 현상    | Alertmanager에는 Alert가 표시되지만 Slack 채널에는 알림이 오지 않았습니다.                                 |
| 원인    | Slack Webhook URL이 잘못되었거나 Alertmanager Receiver 설정에 Webhook URL이 반영되지 않았습니다.         |
| 해결 과정 | Webhook URL을 curl 명령어로 먼저 테스트하고, `alertmanager.yml`의 `api_url`과 `channel` 값을 확인했습니다. |
| 결과    | Slack 채널로 테스트 메시지와 Alert 알림이 정상 전송되었습니다.                                             |
| 재발 방지 | Webhook URL은 공개 저장소에 직접 노출하지 않고, 실제 운영 환경에서는 Secret으로 분리하도록 정리했습니다.                  |

#### 문제 4. Warning과 Critical 알림이 중복되는 문제

| 구분    | 내용                                                                                                     |
| ----- | ------------------------------------------------------------------------------------------------------ |
| 현상    | 동일한 장애에 대해 Warning과 Critical 알림이 함께 발생했습니다.                                                            |
| 원인    | Critical 조건이 만족되어도 Warning Alert가 동시에 유지되어 중복 알림이 발생했습니다.                                              |
| 해결 과정 | `inhibit_rules`를 설정하여 동일한 `alertname`, `host`, `service`에서 Critical 알림이 발생하면 Warning 알림을 억제하도록 구성했습니다. |
| 결과    | 중복 알림이 줄어들고, 운영자가 중요한 장애를 우선 확인할 수 있게 되었습니다.                                                           |
| 재발 방지 | 신규 Alert Rule을 추가할 때 Severity와 Inhibit 조건을 함께 검토하도록 정리했습니다.                                            |
