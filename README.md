# 📊 MaeSoonGan Observability Server

## ✍️ 프로젝트 한 줄 소개

- AWS와 On-Premise 환경에서 발생하는 Metric, Log, Trace를 수집하고, Grafana Dashboard와 Slack Alert를 통해 장애 상황을 확인할 수 있도록 구성한 On-Premise 기반 Observability 서버입니다.

<br />

## 🛫 레포지토리 개요

- 이 레포지토리는 MaeSoonGan 프로젝트의 Observability 서버 구축 및 운영 문서를 관리합니다.

- Observability 서버는 On-Premise Main 서버, On-Premise DR 서버, AWS EKS, DB, Kafka, 애플리케이션의 상태를 수집하고 시각화하기 위해 구성했습니다. Prometheus는 Metric 수집과 Alert Rule 평가를 담당하고, Loki는 Log 저장 및 조회를 담당하며, Tempo는 Trace 저장 및 조회를 담당합니다. Grafana는 수집된 데이터를 Dashboard로 시각화하고, Alertmanager는 장애 조건이 감지되었을 때 Slack으로 알림을 전달합니다.

- 또한 Monitoring 영역의 네트워크 연결과 외부 접근을 위해 VyOS 라우터와 임시 NAT용 pfSense를 함께 구성했습니다. Alloy는 각 서버와 서비스의 Metric, Log, Trace를 수집하여 Prometheus, Loki, Tempo로 전달하는 Collector 역할을 수행합니다.

<br />

## 🖥️ 서버 사양

| 구분              | 내용          |
| --------------- | ----------- |
| Hypervisor      | VMware ESXi |
| VM Name         | 입력 예정       |
| OS              | 입력 예정       |
| vCPU            | 입력 예정       |
| RAM             | 입력 예정       |
| Disk            | 입력 예정       |
| Network Adapter | 입력 예정       |
| IP Address      | 입력 예정       |
| Gateway         | 입력 예정       |
| DNS             | 입력 예정       |

<br />

## 🧩 구성 요소

| 구성 요소        | 역할                                           |
| ------------ | -------------------------------------------- |
| VyOS Router  | Observability 서버와 각 수집 대상 간 라우팅 및 네트워크 연결 구성 |
| Grafana      | Metric, Log, Trace를 통합 시각화하는 Dashboard 제공    |
| Alertmanager | Prometheus Alert를 수신하고 Slack으로 알림 전송         |
| Prometheus   | Metric 수집, 저장, PromQL 조회, Alert Rule 평가      |
| Loki         | 서버 및 애플리케이션 Log 저장 및 LogQL 기반 조회             |
| Tempo        | Trace 저장 및 Trace ID 기반 요청 흐름 조회              |
| pfSense      | 임시 NAT 및 외부 통신 지원                            |
| Alloy        | Metric, Log, Trace 수집 Agent 및 Pipeline 구성    |

<br />

## 🏗️ Observability 서버 아키텍처
<img width="1021" height="456" alt="image" src="https://github.com/user-attachments/assets/194bbcdb-6d26-473f-924a-739e887f9103" />


<br />

## 📡 수집 대상

### On-Premise Main Server

* CPU, Memory, Disk, Network 사용량
* 원장 시스템 상태
* 체결 엔진 상태
* DB 상태
* Kafka 상태
* 애플리케이션 Log

### On-Premise DR Server

* DR 서버 리소스 상태
* DR DB 상태
* DR Kafka 상태
* Replication 상태
* Failover 이후 서비스 상태

### AWS / EKS

* EKS Node 상태
* Pod 상태
* Service 상태
* 애플리케이션 Metric
* 애플리케이션 Log
* Trace 데이터

### Observability Server

* Prometheus 상태
* Grafana 상태
* Loki 상태
* Tempo 상태
* Alertmanager 상태
* Alloy 상태

<br />

## 🔁 데이터 흐름

1. 각 서버와 애플리케이션에서 Metric, Log, Trace가 발생합니다.
2. Alloy가 수집 대상의 데이터를 수집합니다.
3. Metric은 Prometheus로 전달됩니다.
4. Log는 Loki로 전달됩니다.
5. Trace는 Tempo로 전달됩니다.
6. Grafana는 Prometheus, Loki, Tempo를 Data Source로 연결합니다.
7. 운영자는 Grafana Dashboard에서 시스템 상태를 확인합니다.
8. Prometheus Alert Rule이 장애 조건을 감지합니다.
9. Alertmanager가 알림을 그룹화하고 Slack으로 전송합니다.
10. 운영자는 Slack 알림을 확인한 뒤 Grafana에서 원인을 분석합니다.

<br />

## 📋 목차

### 네트워크 구성

* [VyOS 라우터 구성](./docs/network/vyos-router.md)
* [임시 NAT용 pfSense 구성](./docs/network/pfsense-nat.md)
* [라우팅 설정](./docs/network/routing.md)
* [방화벽 및 포트 설정](./docs/network/firewall-port.md)

### Metric 수집

* [Prometheus 설치](./docs/prometheus/prometheus-install.md)
* [Prometheus 설정](./docs/prometheus/prometheus-config.md)
* [Scrape Target 구성](./docs/prometheus/scrape-target.md)
* [Prometheus Rule 작성](./docs/prometheus/prometheus-rule.md)

### Log 수집

* [Loki 설치](./docs/loki/loki-install.md)
* [Loki 설정](./docs/loki/loki-config.md)
* [Log 수집 Pipeline](./docs/loki/log-pipeline.md)
* [LogQL 조회 예시](./docs/loki/logql.md)

### Trace 수집

* [Tempo 설치](./docs/tempo/tempo-install.md)
* [Tempo 설정](./docs/tempo/tempo-config.md)
* [Trace 수집 Pipeline](./docs/tempo/trace-pipeline.md)
* [Trace ID 기반 조회](./docs/tempo/trace-query.md)

### Collector 구성

* [Alloy 설치](./docs/alloy/alloy-install.md)
* [Alloy 설정](./docs/alloy/alloy-config.md)
* [Metric 수집 설정](./docs/alloy/alloy-metric.md)
* [Log 수집 설정](./docs/alloy/alloy-log.md)
* [Trace 수집 설정](./docs/alloy/alloy-trace.md)

### Dashboard 구성

* [Grafana 설치](./docs/grafana/grafana-install.md)
* [Data Source 설정](./docs/grafana/datasource.md)
* [Overview Dashboard](./docs/grafana/overview-dashboard.md)
* [Main Server Dashboard](./docs/grafana/main-server-dashboard.md)
* [DR Server Dashboard](./docs/grafana/dr-server-dashboard.md)
* [Monitoring Server Dashboard](./docs/grafana/monitoring-server-dashboard.md)
* [AWS/EKS Dashboard](./docs/grafana/aws-eks-dashboard.md)

### Alert 구성

* [Alertmanager 설치](./docs/alertmanager/alertmanager-install.md)
* [Alertmanager 설정](./docs/alertmanager/alertmanager-config.md)
* [Slack Webhook 연동](./docs/alertmanager/slack-webhook.md)
* [장애 알림 Rule](./docs/alertmanager/alert-rule.md)
* [알림 그룹화 및 중복 제거](./docs/alertmanager/alert-routing.md)
