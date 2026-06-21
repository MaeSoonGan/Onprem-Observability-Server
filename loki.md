# Loki 로그 수집 환경 구성

## 1. 개요 및 프로젝트 내 역할

- Loki는 MaeSoonGan Observability 서버에서 Log 저장과 조회를 담당하는 로그 저장소입니다.

- 본 프로젝트에서는 Monitoring Log/Trace VM에 Loki를 구성하여 On-Premise Main 서버, DR 서버, Monitoring 서버, 애플리케이션에서 발생하는 로그를 저장하고 Grafana에서 조회할 수 있도록 구성했습니다.

- 각 VM 또는 애플리케이션에서 발생한 로그는 Alloy가 수집하여 Loki로 Push합니다. 이후 Grafana는 Loki를 Data Source로 연결하여 운영자가 서버 로그, 인증 로그, 애플리케이션 로그를 한 화면에서 확인할 수 있도록 합니다.

- Loki는 Prometheus처럼 Label 기반으로 로그를 구분할 수 있어 `area`, `host`, `service`, `job` 등의 Label을 기준으로 로그를 조회하기 쉽습니다. 이를 통해 장애 발생 시 Metric과 Log를 연결하여 원인을 추적할 수 있습니다.

</br>

## 2. 구성 환경 및 사양

| 구분              | 내용                                       |
| --------------- | ---------------------------------------- |
| VM              | Monitoring Log/Trace VM                  |
| 역할              | Log 저장 및 조회                              |
| Hypervisor      | VMware ESXi                              |
| OS              | Ubuntu Server                            |
| vCPU            | 2 vCPU                                   |
| RAM             | 8GB                                      |
| Disk            | 90GB                                     |
| IP Address      | 10.6.10.33/24                            |
| Gateway         | 10.6.10.1                                |
| 작업 디렉터리         | /opt/monitoring/logtrace                 |
| Data Directory  | /opt/monitoring/logtrace/loki-data       |
| Config File     | /opt/monitoring/logtrace/loki-config.yml |
| Service Port    | 3100                                     |
| Container Image | grafana/loki:latest                      |
| 조회 도구           | Grafana Data Source                      |
| 로그 수집 Agent     | Alloy                                    |

- Loki는 로그 Chunk와 Index 데이터를 저장해야 하므로 별도의 데이터 디렉터리를 구성했습니다. 본 프로젝트에서는 Log/Trace 저장소를 같은 VM에 배치하되, Loki와 Tempo의 데이터 디렉터리는 각각 분리했습니다.

</br>

## 3. 선택 이유 및 대안 비교

### 선택 이유

- Loki를 선택한 이유는 Grafana와의 연동성이 높고, Prometheus와 유사한 Label 기반 방식으로 로그를 조회할 수 있기 때문입니다.

- 본 프로젝트에서는 Metric, Log, Trace를 Grafana 중심으로 통합 조회하는 구조를 목표로 했습니다. Loki는 Grafana에서 바로 Data Source로 연결할 수 있고, Alloy와도 자연스럽게 연동됩니다. 또한 Elasticsearch나 OpenSearch보다 상대적으로 가볍게 구성할 수 있어 실습 환경의 제한된 리소스에 적합했습니다.

### 대안 비교

| 대안                         | 장점                                    | 한계                                   | 최종 판단      |
| -------------------------- | ------------------------------------- | ------------------------------------ | ---------- |
| Loki                       | Grafana 연동이 쉽고 Label 기반 로그 조회가 가능합니다. | 전문 검색엔진 수준의 복잡한 검색 기능은 제한적입니다.       | 최종 선택했습니다. |
| Elasticsearch / OpenSearch | 검색 기능이 강력하고 대규모 로그 분석에 적합합니다.         | RAM과 Disk 사용량이 크고 운영 복잡도가 높습니다.      | 제외했습니다.    |
| CloudWatch Logs            | AWS 로그 수집에는 적합합니다.                    | On-Premise 로그까지 통합 저장하기에는 구조가 분리됩니다. | 제외했습니다.    |
| 단순 파일 로그 확인                | 구성이 필요 없습니다.                          | SSH 접속이 필요하고 중앙 조회가 어렵습니다.           | 제외했습니다.    |

</br>

## 4. 설치 및 주요 설정

### 4-1. 작업 디렉터리 생성

- Loki와 Tempo를 함께 구성하는 Log/Trace VM에서 작업 디렉터리를 생성합니다.

```
sudo mkdir -p /opt/monitoring/logtrace
sudo chown -R fisa:fisa /opt/monitoring
cd /opt/monitoring/logtrace
mkdir -p loki-data tempo-data
```

- 최종적으로 `/opt/monitoring/logtrace` 하위에는 다음 파일과 디렉터리가 위치합니다.

| 경로              | 설명                                |
| --------------- | --------------------------------- |
| compose.yml     | Loki와 Tempo 실행용 Docker Compose 파일 |
| loki-config.yml | Loki 설정 파일                        |
| tempo.yml       | Tempo 설정 파일                       |
| loki-data/      | Loki 로그 Chunk 및 Index 저장 디렉터리     |
| tempo-data/     | Tempo Trace 및 WAL 저장 디렉터리         |

### 4-2. Loki 설정 파일 작성

- Loki 설정 파일을 생성합니다.

```
nano loki-config.yml
```

- 주요 설정은 다음과 같습니다.

```
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 72h
  allow_structured_metadata: false

compactor:
  working_directory: /loki/compactor
  retention_enabled: true
  delete_request_store: filesystem
```

- 주요 설정 의미는 다음과 같습니다.

| 설정                            | 설명                                     |
| ----------------------------- | -------------------------------------- |
| `auth_enabled: false`         | 내부망 실습 환경이므로 인증 설정을 단순화했습니다.           |
| `http_listen_port: 3100`      | Loki HTTP API 포트입니다.                   |
| `filesystem`                  | 단일 VM 구성에서 로컬 디스크에 로그를 저장합니다.          |
| `schema: v13`                 | TSDB 기반 Loki 저장 스키마를 사용합니다.            |
| `retention_period: 72h`       | 디스크 사용량을 고려하여 로그 보관 기간을 72시간으로 제한했습니다. |
| `compactor.retention_enabled` | 보관 기간이 지난 로그를 정리할 수 있도록 설정했습니다.        |

### 4-3. Docker Compose 설정

- Loki는 Docker Compose로 실행합니다.

```
nano compose.yml
```

- Loki 서비스 설정은 다음과 같습니다.

```
services:
  loki:
    image: grafana/loki:latest
    container_name: loki
    restart: unless-stopped
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml
      - ./loki-data:/loki
```

- 주요 설정 의미는 다음과 같습니다.

| 설정                        | 설명                                   |
| ------------------------- | ------------------------------------ |
| `3100:3100`               | Grafana와 Alloy가 접근할 Loki HTTP 포트입니다. |
| `loki-config.yml` Mount   | 호스트의 설정 파일을 컨테이너 내부 설정 파일로 연결합니다.    |
| `loki-data:/loki`         | Loki 데이터 저장 경로를 호스트 디렉터리로 분리합니다.     |
| `restart: unless-stopped` | VM 재부팅 또는 컨테이너 종료 시 자동 재시작합니다.       |

### 4-4. 데이터 디렉터리 권한 설정

- Loki 컨테이너가 데이터 디렉터리에 쓰기 작업을 수행할 수 있도록 권한을 설정합니다.

```
sudo chown -R 10001:10001 loki-data tempo-data
```

- Loki는 로그 Chunk와 Index를 `loki-data`에 저장합니다. 컨테이너가 root가 아닌 사용자로 실행되기 때문에, 데이터 디렉터리 권한이 맞지 않으면 `permission denied` 문제가 발생할 수 있습니다.

### 4-5. 컨테이너 실행 및 상태 확인

- Docker Compose 문법을 확인합니다.

```
docker compose config
```

- 컨테이너를 실행합니다.

```
docker compose up -d
```

- 컨테이너 상태를 확인합니다.

```
docker ps
docker ps -a
```

- 포트 Listen 상태를 확인합니다.

```
sudo ss -lntp | grep -E '3100|3200|4317|4318'
```

- Loki Ready 상태를 확인합니다.

```
curl http://localhost:3100/ready
```

- 시작 직후에는 `Ingester not ready: waiting for 15s after being ready` 메시지가 나올 수 있습니다. 이는 Loki가 초기화되는 동안 발생할 수 있는 상태이므로, 15초에서 1분 정도 기다린 뒤 다시 확인합니다.

- 정상 상태에서는 다음과 같은 응답을 기대할 수 있습니다.

```
ready
```

</br>

## 5. 동작 흐름 및 트러블슈팅

### 동작 흐름

1. 각 VM 또는 애플리케이션에서 시스템 로그와 애플리케이션 로그가 발생합니다.
2. Alloy가 지정된 로그 파일을 읽습니다.
3. Alloy는 Loki Push API로 로그를 전송합니다.
4. Loki는 수신한 로그를 Label과 함께 저장합니다.
5. Grafana는 Loki를 Data Source로 연결합니다.
6. 운영자는 Grafana Explore 또는 Dashboard에서 LogQL을 사용하여 로그를 조회합니다.
7. 장애 발생 시 Prometheus Metric과 Loki Log를 함께 확인하여 원인을 분석합니다.

### 트러블슈팅

#### 문제 1. Loki Ready 상태가 바로 나오지 않는 문제

| 구분    | 내용                                                                         |
| ----- | -------------------------------------------------------------------------- |
| 현상    | `curl http://localhost:3100/ready` 실행 시 `Ingester not ready` 메시지가 출력되었습니다. |
| 원인    | Loki 시작 직후 Ingester가 완전히 준비되기 전까지 대기 시간이 필요했습니다.                           |
| 해결 과정 | 15초에서 1분 정도 기다린 뒤 Ready Endpoint를 다시 확인했습니다.                               |
| 결과    | `ready` 응답이 반환되어 Loki가 정상적으로 실행 중임을 확인했습니다.                                |
| 재발 방지 | Loki 컨테이너 기동 직후 Ready 상태 확인 시 초기 대기 시간이 있을 수 있음을 문서화했습니다.                  |

#### 문제 2. 데이터 디렉터리 권한 문제

| 구분    | 내용                                                                          |
| ----- | --------------------------------------------------------------------------- |
| 현상    | Loki 컨테이너가 실행되지 않거나 데이터 디렉터리에 쓰기 실패가 발생할 수 있습니다.                            |
| 원인    | `loki-data` 디렉터리 소유권이 컨테이너 실행 사용자와 맞지 않았습니다.                                |
| 해결 과정 | `sudo chown -R 10001:10001 loki-data tempo-data` 명령어로 데이터 디렉터리 소유권을 수정했습니다. |
| 결과    | Loki가 로그 Chunk와 Index 데이터를 정상적으로 저장할 수 있었습니다.                               |
| 재발 방지 | Docker Compose 실행 전 데이터 디렉터리 권한을 먼저 설정하도록 절차화했습니다.                          |

#### 문제 3. Compose 파일 문법 오류

| 구분    | 내용                                                                                    |
| ----- | ------------------------------------------------------------------------------------- |
| 현상    | Docker Compose 실행 또는 검증 시 `additional properties '-services' not allowed` 오류가 발생했습니다. |
| 원인    | `compose.yml` 첫 줄을 `services:`가 아니라 `-services:`로 잘못 작성했습니다.                          |
| 해결 과정 | Compose 파일 첫 줄을 `services:`로 수정하고 `docker compose config`로 문법을 다시 확인했습니다.             |
| 결과    | Docker Compose 문법 검증이 정상 통과되었습니다.                                                     |
| 재발 방지 | 컨테이너 실행 전 항상 `docker compose config`로 문법을 확인하도록 정리했습니다.                               |
