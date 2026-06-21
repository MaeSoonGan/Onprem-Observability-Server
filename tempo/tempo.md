# Tempo Trace 수집 환경 구성

## 1. 개요 및 프로젝트 내 역할

- Tempo는 MaeSoonGan Observability 서버에서 Trace 저장과 조회를 담당하는 분산 추적 저장소입니다.

- 본 프로젝트에서는 Monitoring Log/Trace VM에 Tempo를 구성하여 애플리케이션 요청 흐름을 Trace ID 기반으로 조회할 수 있도록 했습니다. Spring Boot 애플리케이션 또는 각 서비스에서 생성된 Trace 데이터는 OpenTelemetry 또는 Alloy를 통해 Tempo로 전달됩니다.

- Grafana는 Tempo를 Data Source로 연결하여 운영자가 Trace ID를 기준으로 요청 흐름을 확인할 수 있도록 합니다. 이를 통해 단순히 서버가 정상인지 확인하는 수준을 넘어, 특정 요청이 어느 서비스에서 지연되었는지, 어느 구간에서 오류가 발생했는지 추적할 수 있습니다.

</br>

## 2. 구성 환경 및 사양

| 구분              | 내용                                  |
| --------------- | ----------------------------------- |
| VM              | Monitoring Log/Trace VM             |
| 역할              | Trace 저장 및 Trace ID 기반 조회           |
| Hypervisor      | VMware ESXi                         |
| OS              | Ubuntu Server                       |
| vCPU            | 2 vCPU                              |
| RAM             | 8GB                                 |
| Disk            | 90GB                                |
| IP Address      | 10.6.10.33/24                       |
| Gateway         | 10.6.10.1                           |
| 작업 디렉터리         | /opt/monitoring/logtrace            |
| Data Directory  | /opt/monitoring/logtrace/tempo-data |
| Config File     | /opt/monitoring/logtrace/tempo.yml  |
| HTTP Port       | 3200                                |
| OTLP gRPC Port  | 4317                                |
| OTLP HTTP Port  | 4318                                |
| Container Image | grafana/tempo:latest                |
| 조회 도구           | Grafana Data Source                 |
| Trace 수집 방식     | OTLP 기반 수집                          |

- Tempo는 Loki와 같은 Log/Trace VM에 배치했습니다. Trace 데이터와 WAL 파일은 `tempo-data` 디렉터리에 저장하며, Loki 데이터와 분리하여 관리합니다.

</br>

## 3. 선택 이유 및 대안 비교

### 선택 이유

- Tempo를 선택한 이유는 Grafana, Prometheus, Loki와의 통합성이 높고, Grafana 안에서 Metric, Log, Trace를 함께 연결해 장애 원인을 분석하기 쉽기 때문입니다.

- 본 프로젝트에서는 Grafana 중심의 통합 관측성 구조를 목표로 했습니다. Tempo는 별도의 복잡한 검색 저장소 없이 Trace ID 기반 조회에 집중할 수 있고, OpenTelemetry 및 Alloy와도 자연스럽게 연동됩니다. 또한 단일 VM 실습 환경에서는 Local Backend로 구성할 수 있어 운영 부담을 줄일 수 있었습니다.

### 대안 비교

| 대안        | 장점                                              | 한계                                                                     | 최종 판단      |
| --------- | ----------------------------------------------- | ---------------------------------------------------------------------- | ---------- |
| Tempo     | Grafana Stack과 통합이 자연스럽고 Trace ID 기반 조회에 적합합니다. | Trace 검색 기능은 별도 인덱싱 기반 도구보다 제한적입니다.                                    | 최종 선택했습니다. |
| Jaeger    | 분산 트레이싱 도구로 널리 사용됩니다.                           | 운영 환경에서는 Elasticsearch, OpenSearch, Cassandra 등 별도 저장소 구성이 필요할 수 있습니다. | 제외했습니다.    |

</br>

## 4. 설치 및 주요 설정

### 4-1. 작업 디렉터리 생성

- Tempo는 Loki와 함께 Log/Trace VM에서 실행합니다.

```
sudo mkdir -p /opt/monitoring/logtrace
sudo chown -R fisa:fisa /opt/monitoring
cd /opt/monitoring/logtrace
mkdir -p loki-data tempo-data
```

- Tempo 관련 파일과 디렉터리는 다음과 같습니다.

| 경로          | 설명                  |
| ----------- | ------------------- |
| tempo.yml   | Tempo 설정 파일         |
| tempo-data/ | Trace 및 WAL 저장 디렉터리 |
| compose.yml | Tempo 컨테이너 실행 설정 파일 |

### 4-2. Tempo 설정 파일 작성

- Tempo 설정 파일을 생성합니다.

```
nano tempo.yml
```

- 설정 내용은 다음과 같습니다.

```
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318

storage:
  trace:
    backend: local
    local:
      path: /var/tempo/traces
    wal:
      path: /var/tempo/wal
```

- 주요 설정 의미는 다음과 같습니다.

| 설정                       | 설명                                                   |
| ------------------------ | ---------------------------------------------------- |
| `http_listen_port: 3200` | Grafana가 Tempo를 조회할 때 사용하는 HTTP 포트입니다.               |
| `otlp.grpc 4317`         | OpenTelemetry 또는 Alloy가 gRPC 방식으로 Trace를 전달하는 포트입니다. |
| `otlp.http 4318`         | OpenTelemetry 또는 Alloy가 HTTP 방식으로 Trace를 전달하는 포트입니다. |
| `backend: local`         | 단일 VM 실습 환경에 맞춰 로컬 디스크를 Trace 저장소로 사용합니다.            |
| `/var/tempo/traces`      | Trace 데이터 저장 경로입니다.                                  |
| `/var/tempo/wal`         | Tempo WAL 저장 경로입니다.                                  |

`/tempo` 경로는 컨테이너 내부 실행 파일 경로와 충돌할 수 있어 사용하지 않았고, `/var/tempo` 경로를 사용했습니다.

### 4-3. Docker Compose 설정

- Tempo는 Docker Compose로 실행합니다.

```
nano compose.yml
```

- Tempo 서비스 설정은 다음과 같습니다.

```
services:
  tempo:
    image: grafana/tempo:latest
    container_name: tempo
    restart: unless-stopped
    ports:
      - "3200:3200"
      - "4317:4317"
      - "4318:4318"
    command: ["-config.file=/etc/tempo.yml"]
    volumes:
      - ./tempo.yml:/etc/tempo.yml
      - ./tempo-data:/var/tempo
```

- 주요 설정 의미는 다음과 같습니다.

| 설정                      | 설명                                   |
| ----------------------- | ------------------------------------ |
| `3200:3200`             | Grafana에서 Tempo를 조회하기 위한 HTTP 포트입니다. |
| `4317:4317`             | OTLP gRPC Trace 수신 포트입니다.            |
| `4318:4318`             | OTLP HTTP Trace 수신 포트입니다.            |
| `tempo.yml` Mount       | 호스트 설정 파일을 컨테이너 내부 설정 파일로 연결합니다.     |
| `tempo-data:/var/tempo` | Trace와 WAL 데이터를 호스트 디렉터리에 저장합니다.     |

### 4-4. 데이터 디렉터리 권한 설정

- Tempo 컨테이너가 Trace와 WAL 데이터를 저장할 수 있도록 데이터 디렉터리 권한을 설정합니다.

```
sudo chown -R 10001:10001 loki-data tempo-data
```

- `tempo-data`는 Tempo Trace와 WAL이 저장되는 경로입니다. 컨테이너가 root가 아닌 사용자로 실행될 수 있으므로, 쓰기 권한이 없으면 컨테이너가 정상적으로 실행되지 않을 수 있습니다.

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

- Tempo Ready 상태를 확인합니다.

```
curl http://localhost:3200/ready
```

- 연결 실패가 발생하면 Tempo 컨테이너 로그를 확인합니다.

```
docker logs tempo --tail=100
```

</br>

## 5. 동작 흐름 및 트러블슈팅

### 동작 흐름

1. Spring Boot 애플리케이션 또는 서비스에서 요청 Trace가 생성됩니다.
2. OpenTelemetry 또는 Alloy가 Trace 데이터를 OTLP 형식으로 수집합니다.
3. Trace 데이터는 Tempo의 4317 또는 4318 포트로 전달됩니다.
4. Tempo는 수신한 Trace 데이터를 로컬 디스크에 저장합니다.
5. Grafana는 Tempo를 Data Source로 연결합니다.
6. 운영자는 Grafana Explore에서 Trace ID 또는 Service Name을 기준으로 요청 흐름을 조회합니다.
7. 장애 발생 시 Metric, Log와 함께 Trace를 확인하여 어느 구간에서 지연 또는 오류가 발생했는지 분석합니다.

### 트러블슈팅

#### 문제 1. Tempo Mount 오류

| 구분    | 내용                                                                                                                                                     |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 현상    | Tempo 컨테이너 실행 시 `not a directory` 또는 `mounting ./tempo-data to /tempo` 오류가 발생했습니다.                                                                     |
| 원인    | `/tempo` 경로가 컨테이너 내부에서 데이터 디렉터리로 사용하기 부적절하거나 실행 파일 경로와 충돌할 수 있었습니다.                                                                                    |
| 해결 과정 | Docker Compose의 Mount 경로를 `./tempo-data:/tempo`에서 `./tempo-data:/var/tempo`로 수정했습니다. Tempo 설정 파일의 저장 경로도 `/var/tempo/traces`, `/var/tempo/wal`로 맞췄습니다. |
| 결과    | Tempo 컨테이너가 정상적으로 실행되었고 Trace 저장 경로가 안정적으로 분리되었습니다.                                                                                                    |
| 재발 방지 | Tempo 데이터 저장 경로는 컨테이너 내부 실행 경로와 충돌하지 않는 `/var/tempo` 하위로 통일했습니다.                                                                                       |

#### 문제 2. Docker Compose 문법 오류

| 구분    | 내용                                                                              |
| ----- | ------------------------------------------------------------------------------- |
| 현상    | Docker Compose 검증 시 `additional properties '-services' not allowed` 오류가 발생했습니다. |
| 원인    | `compose.yml` 첫 줄을 `services:`가 아니라 `-services:`로 잘못 작성했습니다.                    |
| 해결 과정 | 첫 줄을 `services:`로 수정하고 `docker compose config` 명령어로 문법을 다시 확인했습니다.              |
| 결과    | Compose 파일이 정상적으로 인식되었고 컨테이너 실행이 가능해졌습니다.                                       |
| 재발 방지 | Docker Compose 실행 전 반드시 `docker compose config`로 문법 검증을 수행하도록 정리했습니다.           |

#### 문제 3. Tempo 설정 파싱 실패

| 구분    | 내용                                                                                                                   |
| ----- | -------------------------------------------------------------------------------------------------------------------- |
| 현상    | Tempo 실행 시 `field ingester not found in type app.Config`, `field compactor not found in type app.Config` 오류가 발생했습니다. |
| 원인    | 현재 사용한 Tempo 이미지 버전과 설정 파일 스키마가 맞지 않았습니다. `ingester`, `compactor` 항목을 현재 설정에서 인식하지 못했습니다.                            |
| 해결 과정 | `ingester`, `compactor` 항목을 제거하고, OTLP Receiver와 Local Storage 중심의 최소 설정으로 Tempo를 기동했습니다.                            |
| 결과    | Tempo가 정상적으로 실행되었고, 3200, 4317, 4318 포트가 정상적으로 열렸습니다.                                                                |
| 재발 방지 | Tempo 설정을 변경할 때는 현재 컨테이너 이미지 버전에서 지원하는 설정 스키마를 기준으로 최소 설정부터 적용하도록 정리했습니다.                                            |

#### 문제 4. Tempo Ready 확인 실패

| 구분    | 내용                                                                             |               |      |                                                        |
| ----- | ------------------------------------------------------------------------------ | ------------- | ---- | ------------------------------------------------------ |
| 현상    | `curl http://localhost:3200/ready` 실행 시 연결 실패가 발생했습니다.                         |               |      |                                                        |
| 원인    | Tempo 컨테이너가 정상적으로 실행되지 않았거나, 포트 매핑 또는 설정 파일 오류가 있었을 가능성이 있습니다.                 |               |      |                                                        |
| 해결 과정 | `docker ps -a`, `sudo ss -lntp                                                 | grep -E '3200 | 4317 | 4318'`, `docker logs tempo --tail=100` 순서로 상태를 확인했습니다. |
| 결과    | 설정 오류를 수정한 뒤 Tempo Ready Endpoint가 정상 응답했습니다.                                  |               |      |                                                        |
| 재발 방지 | Tempo 실행 후 컨테이너 상태, 포트 Listen 상태, Ready Endpoint, 컨테이너 로그를 순서대로 확인하도록 절차화했습니다. |               |      |                                                        |
