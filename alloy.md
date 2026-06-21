# Alloy 수집 Agent 구성

## 1. 개요 및 프로젝트 내 역할

- Alloy는 MaeSoonGan Observability 서버에서 Metric, Log, Trace를 수집하는 통합 Agent입니다.

- 본 프로젝트에서는 각 VM의 시스템 메트릭과 로그를 수집하고, Spring 기반 서비스에서 발생하는 Trace를 Tempo로 전달하기 위해 Alloy를 사용했습니다. Alloy는 VM 내부의 CPU, Memory, Disk, Network와 같은 시스템 메트릭을 Prometheus가 Scrape할 수 있도록 Endpoint로 노출하고, `/var/log` 하위의 시스템 로그와 애플리케이션 로그를 Loki로 Push합니다.

- Trace의 경우 Spring Boot 서비스가 OpenTelemetry Java Agent를 통해 Trace를 생성하면, Alloy가 OTLP Receiver로 해당 Trace를 수신한 뒤 Tempo로 전달하는 구조로 구성했습니다.

- 즉, Alloy는 각 VM과 서비스에서 발생하는 관측성 데이터를 Prometheus, Loki, Tempo로 전달하는 수집 파이프라인의 시작점 역할을 합니다.

</br>

## 2. 구성 환경 및 사양

| 구분                   | 내용                                          |
| -------------------- | ------------------------------------------- |
| 구성 요소                | Grafana Alloy                               |
| 역할                   | Metric, Log, Trace 통합 수집 Agent              |
| 설치 대상                | Monitoring VM, Main Server VM, DR Server VM |
| Ubuntu 설치 방식         | Grafana APT Repository 기반 설치                |
| Rocky Linux 설치 방식    | Grafana RPM Repository 기반 설치                |
| Metric Endpoint Port | 12345                                       |
| Metric 전달 방식         | Prometheus가 Alloy Endpoint를 Pull            |
| Log 전달 방식            | Alloy가 Loki로 Push                           |
| Trace 전달 방식          | Alloy OTLP Receiver가 Tempo로 Push            |
| Loki Endpoint        | 10.6.10.33:3100                             |
| Tempo OTLP Endpoint  | 10.6.10.33:4317                             |
| 설정 파일                | /etc/alloy/config.alloy                     |
| Ubuntu 환경 변수 파일      | /etc/default/alloy                          |
| Rocky 환경 변수 파일       | /etc/sysconfig/alloy                        |

### 주요 수집 대상

| 계층                 | 수집 데이터                                            | 수집 방식                                         |
| ------------------ | ------------------------------------------------- | --------------------------------------------- |
| System             | CPU, Memory, Disk, Network, VM 상태                 | Alloy `prometheus.exporter.unix`              |
| System Log         | syslog, auth log, messages, secure                | Alloy file source                             |
| Process            | Java, MySQL, Docker, GitLab 등 프로세스 상태             | Alloy `prometheus.exporter.process`           |
| Application Metric | Spring Actuator, MySQL Exporter, Kafka Exporter 등 | Prometheus Scrape                             |
| Application Log    | Spring Log, MySQL Log, Docker Log, GitLab Log     | Alloy file source                             |
| Trace              | Spring Boot 요청 Trace                              | OpenTelemetry Java Agent → Alloy OTLP → Tempo |

</br>

## 3. 선택 이유 및 대안 비교

### 선택 이유

- Alloy를 선택한 이유는 VM마다 Metric, Log, Trace 수집을 하나의 Agent로 통합할 수 있기 때문입니다.

- Node Exporter, Promtail, OpenTelemetry Collector를 각각 설치하면 기능별로 Agent가 분리되어 설정 파일과 장애 지점이 늘어납니다. Alloy를 사용하면 시스템 메트릭 수집, 로그 파일 수집, OTLP Trace 수신을 하나의 설정 파일에서 관리할 수 있어 실습 환경에서 운영 복잡도를 줄일 수 있습니다.

- 또한 Grafana, Loki, Tempo, Prometheus와의 연동이 자연스럽고, 추후 EKS 환경에서도 DaemonSet 방식으로 유사하게 확장할 수 있습니다.

### 대안 비교

| 대안                                                 | 장점                                     | 한계                                      | 최종 판단      |
| -------------------------------------------------- | -------------------------------------- | --------------------------------------- | ---------- |
| Alloy                                              | Metric, Log, Trace 수집을 하나로 통합할 수 있습니다. | 설정 문법에 익숙해지는 시간이 필요합니다.                 | 최종 선택했습니다. |
| Node Exporter + Promtail + OpenTelemetry Collector | 각 도구의 역할이 명확합니다.                       | VM마다 여러 Agent를 설치해야 해 관리 포인트가 늘어납니다.    | 제외했습니다.    |
| Filebeat                                           | 로그 수집에 강점이 있습니다.                       | Metric과 Trace까지 통합하기 어렵습니다.             | 제외했습니다.    |
| Docker 기반 Alloy 실행                                 | Docker Compose 구성과 통일됩니다.              | Host 로그와 시스템 메트릭 접근 권한 설정이 복잡해질 수 있습니다. | 제외했습니다.    |
| 패키지 기반 Alloy 설치                                    | Host 로그와 시스템 메트릭 접근이 자연스럽습니다.          | Docker Compose 관리 방식과는 분리됩니다.           | 최종 선택했습니다. |

</br>

## 4. 설치 및 주요 설정

### 4-1. Ubuntu 기준 Alloy 설치

- Monitoring Server 계열 VM에서는 Ubuntu 기준으로 Alloy를 설치합니다.

- Grafana APT 저장소 등록을 위해 필요한 패키지를 설치합니다.

```
sudo apt update
sudo apt install -y gpg curl wget
```

- Grafana GPG Key와 APT 저장소를 등록합니다.

```
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
```

- Alloy를 설치합니다.

```
sudo apt update
sudo apt install -y alloy
```

- 설치 여부를 확인합니다.

```
alloy --version
```

### 4-2. Rocky Linux 기준 Alloy 설치

- Main Server 계열 VM에서는 Rocky Linux 기준으로 Alloy를 설치할 수 있습니다.

- 필수 패키지를 설치합니다.

```
sudo dnf install -y wget curl gpg
```

- Grafana RPM 저장소를 등록합니다.

```
sudo tee /etc/yum.repos.d/grafana.repo > /dev/null <<EOF
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF
```

- Alloy를 설치합니다.

```
sudo dnf makecache
sudo dnf install -y alloy
```

- 서비스 상태를 확인합니다.

```
alloy --version
sudo systemctl status alloy
```

### 4-3. Alloy HTTP 포트 설정

- Prometheus가 Alloy의 Metric Endpoint를 Scrape하려면 Alloy가 외부에서 접근 가능한 주소와 포트로 실행되어야 합니다.

- Ubuntu에서는 `/etc/default/alloy` 파일을 수정합니다.

```
sudo nano /etc/default/alloy
```

- 다음 값을 설정합니다.

```
CUSTOM_ARGS="--server.http.listen-addr=0.0.0.0:12345"
```

- Rocky Linux에서는 `/etc/sysconfig/alloy` 파일을 수정합니다.

```
sudo vi /etc/sysconfig/alloy
```

- 동일하게 다음 값을 설정합니다.

```
CUSTOM_ARGS="--server.http.listen-addr=0.0.0.0:12345"
```

- 설정 반영 후 Alloy를 재시작합니다.

```
sudo systemctl daemon-reload
sudo systemctl restart alloy
```

- 포트 상태를 확인합니다.

```
sudo ss -lntp | grep 12345
```

### 4-4. 기본 Metric 및 Log 수집 설정

- 기존 설정 파일을 백업합니다.

```
sudo cp /etc/alloy/config.alloy /etc/alloy/config.alloy.bak
```

- 설정 파일을 수정합니다.

```
sudo nano /etc/alloy/config.alloy
```

- Ubuntu 기반 Monitoring UI VM 예시는 다음과 같습니다.

```
logging {
  level  = "info"
  format = "logfmt"
}

prometheus.exporter.unix "local" {
  include_exporter_metrics = true
}

local.file_match "system_logs" {
  path_targets = [
    {
      __path__  = "/var/log/syslog"
      job       = "system-logs"
      area      = "monitoring-server"
      host      = "fs-monitoring-ui"
      instance  = "10.6.10.11"
    },
    {
      __path__  = "/var/log/auth.log"
      job       = "auth-logs"
      area      = "monitoring-server"
      host      = "fs-monitoring-ui"
      instance  = "10.6.10.11"
    },
  ]
}

loki.source.file "system_logs" {
  targets    = local.file_match.system_logs.targets
  forward_to = [loki.write.local.receiver]
}

loki.write "local" {
  endpoint {
    url = "http://10.6.10.33:3100/loki/api/v1/push"
  }
}
```

- Rocky Linux에서는 로그 경로가 다르므로 다음처럼 변경합니다.

| OS          | 시스템 로그            | 인증 로그             |
| ----------- | ----------------- | ----------------- |
| Ubuntu      | /var/log/syslog   | /var/log/auth.log |
| Rocky Linux | /var/log/messages | /var/log/secure   |

- 설정 변경 후 Alloy를 재시작합니다.

```
sudo systemctl restart alloy
sudo systemctl status alloy
sudo journalctl -u alloy -n 100 --no-pager
```

### 4-5. VM별 Label 설정

- Alloy 설정에서 `area`, `host`, `instance`, `role`, `service` Label을 VM별로 다르게 지정합니다.

| VM                              | host                   | area              | role          |
| ------------------------------- | ---------------------- | ----------------- | ------------- |
| Monitoring UI VM                | fs-monitoring-ui       | monitoring-server | ui            |
| Monitoring Metric VM            | fs-monitoring-metric   | monitoring-server | metric        |
| Monitoring Log/Trace VM         | fs-monitoring-logtrace | monitoring-server | logtrace      |
| VM1 원장 Spring                   | main-ledger            | main-server       | spring        |
| VM2 체결 엔진 Spring                | main-execution         | main-server       | spring        |
| VM3 Kafka/ProxySQL/Orchestrator | main-kafka-proxy       | main-server       | infra         |
| VM4 MySQL Primary               | main-mysql-primary     | main-server       | mysql-primary |
| VM5 MySQL Replica               | main-mysql-replica     | main-server       | mysql-replica |
| VM6 GitLab                      | main-gitlab            | main-server       | devops        |

### 4-6. Prometheus Scrape Target 추가

- Prometheus VM의 `prometheus.yml`에 Alloy Endpoint를 추가합니다.

```
- job_name: "alloy-unix-main"
  metrics_path: "/api/v0/component/prometheus.exporter.unix.local/metrics"
  static_configs:
    - targets:
        - "VM1_IP:12345"
      labels:
        area: "main-server"
        host: "main-ledger"
        instance: "VM1_IP"
        role: "spring"
```

- 설치가 완료되지 않은 VM을 먼저 Target으로 추가하면 Prometheus에서 DOWN으로 표시될 수 있으므로, Alloy 설치가 완료된 VM부터 순차적으로 추가합니다.

- 설정 반영 후 Prometheus를 Reload합니다.

```
curl -X POST http://localhost:9090/-/reload
```

### 4-7. Trace 수집 설정

- Spring Boot 서비스의 Trace를 수집하려면 VM1, VM2에 OpenTelemetry Java Agent를 설치하고 Alloy에 OTLP Receiver를 추가합니다.

- OpenTelemetry Java Agent를 다운로드합니다.

```
sudo mkdir -p /opt/opentelemetry
cd /tmp
wget https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/v2.14.0/opentelemetry-javaagent.jar
sudo mv opentelemetry-javaagent.jar /opt/opentelemetry/opentelemetry-javaagent.jar
sudo chmod 644 /opt/opentelemetry/opentelemetry-javaagent.jar
```

- Alloy 설정에 OTLP Receiver를 추가합니다.

```
otelcol.receiver.otlp "spring" {
  grpc {
    endpoint = "0.0.0.0:4317"
  }

  http {
    endpoint = "0.0.0.0:4318"
  }

  output {
    traces = [otelcol.exporter.otlp.tempo.input]
  }
}

otelcol.exporter.otlp "tempo" {
  client {
    endpoint = "10.6.10.33:4317"

    tls {
      insecure = true
    }
  }
}
```

- 설정 적용 후 Alloy를 재시작합니다.

```
sudo systemctl restart alloy
sudo journalctl -u alloy -n 100 --no-pager
```

</br>

## 5. 동작 흐름 및 트러블슈팅

### 동작 흐름

1. 각 VM에서 CPU, Memory, Disk, Network 등 시스템 메트릭이 발생합니다.
2. Alloy가 `prometheus.exporter.unix`를 통해 시스템 메트릭을 수집합니다.
3. Prometheus가 Alloy의 12345 Endpoint를 Pull 방식으로 Scrape합니다.
4. 각 VM에서 시스템 로그 또는 애플리케이션 로그가 발생합니다.
5. Alloy가 지정된 로그 파일을 읽고 Loki로 Push합니다.
6. Spring Boot 서비스에서 Trace가 생성됩니다.
7. Alloy가 OTLP Receiver로 Trace를 수신하고 Tempo로 전달합니다.
8. Grafana는 Prometheus, Loki, Tempo를 Data Source로 연결하여 Metric, Log, Trace를 조회합니다.

### 트러블슈팅

#### 문제 1. Alloy Endpoint가 Prometheus에서 DOWN으로 표시되는 문제

| 구분    | 내용                                                                                                                                                                                |                                                                |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| 현상    | Prometheus Targets 화면에서 Alloy Target이 DOWN으로 표시되었습니다.                                                                                                                             |                                                                |
| 원인    | Alloy가 12345 포트로 Listen하지 않았거나, 방화벽에서 12345 포트가 열려 있지 않았습니다.                                                                                                                      |                                                                |
| 해결 과정 | `/etc/default/alloy` 또는 `/etc/sysconfig/alloy`에 `CUSTOM_ARGS="--server.http.listen-addr=0.0.0.0:12345"`를 설정하고 Alloy를 재시작했습니다. Rocky Linux에서는 firewalld 상태를 확인하고 12345 포트를 허용했습니다. |                                                                |
| 결과    | Prometheus에서 Alloy Target이 UP으로 표시되었습니다.                                                                                                                                          |                                                                |
| 재발 방지 | Alloy 설치 후 `sudo ss -lntp                                                                                                                                                         | grep 12345`로 Listen 상태를 확인한 뒤 Prometheus Target에 등록하도록 정리했습니다. |

#### 문제 2. Loki에 로그가 보이지 않는 문제

| 구분    | 내용                                                                                                                                          |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 현상    | Grafana Explore에서 Loki Query를 실행했지만 로그가 조회되지 않았습니다.                                                                                         |
| 원인    | Alloy가 로그 파일을 읽을 권한이 없거나, `local.file_match`의 경로가 실제 로그 경로와 일치하지 않았습니다.                                                                     |
| 해결 과정 | Ubuntu는 `/var/log/syslog`, `/var/log/auth.log`를 확인하고 Rocky Linux는 `/var/log/messages`, `/var/log/secure`를 확인했습니다. 권한 문제가 있는 경우 ACL을 부여했습니다. |
| 결과    | Loki에서 `{area="main-server"}` 또는 `{host="main-ledger"}` 기준으로 로그가 조회되었습니다.                                                                   |
| 재발 방지 | OS별 로그 경로와 Alloy 실행 사용자 권한을 먼저 확인한 뒤 로그 수집 설정을 적용하도록 정리했습니다.                                                                                |

#### 문제 3. Trace가 Tempo에 나타나지 않는 문제

| 구분    | 내용                                                                                                                                                      |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 현상    | Grafana Tempo Explore에서 Spring 서비스 Trace가 조회되지 않았습니다.                                                                                                   |
| 원인    | OpenTelemetry Java Agent가 적용되지 않았거나, Alloy OTLP Receiver가 4317/4318 포트로 열려 있지 않았습니다.                                                                    |
| 해결 과정 | Spring 서비스의 systemd `ExecStart`에 `-javaagent:/opt/opentelemetry/opentelemetry-javaagent.jar` 옵션을 추가하고, Alloy에 OTLP Receiver와 Tempo Exporter 설정을 추가했습니다. |
| 결과    | Grafana Tempo에서 `ledger-service`, `execution-engine` 기준 Trace 조회가 가능해졌습니다.                                                                              |
| 재발 방지 | Trace는 요청이 발생해야 생성되므로, 서비스 기동 후 실제 API 요청을 발생시킨 뒤 Tempo Explore에서 확인하도록 정리했습니다.                                                                         |

#### 문제 4. 설정 변경 후 Alloy가 기동되지 않는 문제

| 구분    | 내용                                                                               |
| ----- | -------------------------------------------------------------------------------- |
| 현상    | `config.alloy` 수정 후 Alloy 서비스가 정상 기동되지 않았습니다.                                    |
| 원인    | Alloy 설정 파일의 문법 오류 또는 Label 값 오타가 원인이었습니다.                                       |
| 해결 과정 | `sudo journalctl -u alloy -n 100 --no-pager`로 오류 내용을 확인하고, 문제가 되는 설정 블록을 수정했습니다. |
| 결과    | Alloy 서비스가 정상 기동되었고 Metric, Log 수집이 재개되었습니다.                                     |
| 재발 방지 | 설정 변경 전 기존 파일을 백업하고, 변경 후 서비스 상태와 Journal Log를 반드시 확인하도록 절차화했습니다.                |
