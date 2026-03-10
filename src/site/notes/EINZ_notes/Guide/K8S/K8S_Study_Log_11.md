---
{"dg-publish":true,"permalink":"/einz-notes/guide/k8-s/k8-s-study-log-11/","title":"11: Grafana, Loki, Prometheus"}
---


# K8S 통합 모니터링 시스템 구축: Prometheus, Grafana, Loki

Kubernetes 클러스터의 시스템 자원 메트릭과 파드(Pod) 로그를 중앙에서 수집하고 시각화하는 통합 모니터링 파이프라인 구축 과정이다.

## 1. 메트릭 수집 및 시각화 스택 설치 (Prometheus & Grafana)

kube-prometheus-stack을 배포하여 클러스터 상태를 수집하는 Prometheus와 이를 시각화하는 Grafana를 설치한다.

### 1.1 Helm 차트 설치

```Bash
# monitoring 전용 네임스페이스 생성
kubectl create namespace monitoring

# Prometheus 커뮤니티 저장소 추가 및 업데이트
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# kube-prometheus-stack 설치 (Prometheus, Grafana 통합)
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring
```

### 1.2 Grafana 자동 포트 포워딩 구성 (Systemd)

가상 머신 재부팅 시에도 백그라운드에서 포트가 상시 개방되도록 systemd 서비스로 등록한다.

**서비스 파일 생성 (`/etc/systemd/system/grafana-pf.service`)**

```Bash
sudo nano /etc/systemd/system/grafana-pf.service
```

```Ini, TOML
[Unit]
Description=Grafana Port Forward Auto Start
After=network.target

[Service]
Type=simple
User=xtra
Environment=KUBECONFIG=/home/xtra/.kube/config
# Grafana 웹 UI를 8089 포트로 상시 개방
ExecStart=/usr/bin/kubectl port-forward --address 0.0.0.0 svc/prometheus-grafana -n monitoring 8089:80
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**서비스 등록 및 실행**

```Bash
# systemd 데몬 새로고침
sudo systemctl daemon-reload

# 부팅 시 자동 실행 활성화 및 즉시 시작
sudo systemctl enable grafana-pf
sudo systemctl start grafana-pf
```

## 2. 중앙 집중형 로그 수집 스택 설치 (Loki & Promtail)

각 파드에 흩어진 텍스트 로그를 수집하는 Promtail과 이를 중앙 저장소에 보관하는 Loki를 설치한다.

### 2.1 Helm 차트 설치

```Bash
# Grafana 공식 저장소 추가 및 업데이트
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# loki-stack 차트 설치 (Loki + Promtail 구성)
helm install loki grafana/loki-stack -n monitoring
```

## 3. Grafana 대시보드 연동 및 데이터 조회

### 3.1 Loki 데이터 소스 연결

Grafana에서 Loki가 수집한 로그를 읽어올 수 있도록 내부 통신 경로를 연결한다.

1. 브라우저에서 Grafana 접속 (`http://localhost:8089`)

2. 초기 계정(`admin` / `비밀번호`)으로 로그인
```Bash
# 그라파나 초기 비밀번호
kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

3. 좌측 메뉴 `Connections` -> `Data sources` 이동

4. `Add data source` 클릭 후 `Loki` 선택

5. HTTP URL 항목에 K8S 내부 서비스 주소 입력: `http://loki:3100`

6. `Save & test` 클릭하여 연결 저장 완료

***트러블슈팅 - Save & Test 시 오류 발생한다면 Grafana와 Loki 버전이 서로 호환되지 않아서 그럴 수 있음. Explore 탭에서 Query 조회로 {namespace: default} 조회하여 Logs가 조회될 경우 정상 설치된 것***

### 3.2 메트릭 및 로그 통합 모니터링

- **메트릭(Metrics) 확인:** `Dashboards` 메뉴에서 사전 구성된 `Kubernetes / Compute Resources / Namespace (Pods)` 대시보드를 선택한다. 파드의 자원(CPU, 메모리) 사용량을 시각화된 그래프로 모니터링한다.

- **로그(Logs) 검색:** `Explore` 메뉴로 이동하여 데이터 소스를 `Loki`로 변경한다. 검색 시간(Time range)을 조정하고 LogQL 문법을 사용하여 실시간 텍스트 로그를 조회한다.

```Plaintext
# default 네임스페이스에서 발생한 모든 로그 조회
{namespace="default"}
```

---

