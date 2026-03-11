---
{"dg-publish":true,"permalink":"/einz-notes/guide/k8s/k8s-study-log-5/","title":"5: Helm 모니터링 구축 및 자동 확장(HPA)"}
---


# Helm 모니터링 구축 및 자동 확장(HPA)

## 1. Helm을 활용한 Prometheus & Grafana 구축

수십 개의 K8S 객체로 이루어진 복잡한 모니터링 시스템을 패키지 매니저인 Helm을 통해 단일 명령어로 배포하는 실습.

### 1.1 Helm 설치 및 저장소 추가

우분투 가상 머신에 Helm을 설치하고 공식 차트 저장소를 등록한다.

```bash
# Helm 설치 스크립트 실행
curl -fsSL -o get_helm.sh [https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3](https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3)
chmod 700 get_helm.sh
./get_helm.sh

# Prometheus 공식 차트 저장소 추가 및 업데이트
helm repo add prometheus-community [https://prometheus-community.github.io/helm-charts](https://prometheus-community.github.io/helm-charts)
helm repo update
````

### 1.2 모니터링 스택 배포

격리된 네임스페이스를 생성하고 `kube-prometheus-stack` 패키지를 설치한다.

```bash
kubectl create namespace monitoring
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring

# 파드 정상 구동 대기
kubectl get pods -n monitoring -w
```

### 1.3 Grafana 관리자 비밀번호 추출

보안 정책에 의해 자동 생성된 관리자 비밀번호를 K8S Secret 객체에서 직접 추출 및 디코딩한다.

```bash
kubectl get secret --namespace monitoring -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echo
```

### 4. 대시보드 접속 및 확인

포트 포워딩을 통해 외부에서 접속 환경을 구성한다.

```bash
# Grafana 포트 포워딩 (30002 포트 사용)
kubectl port-forward --address 0.0.0.0 -n monitoring svc/prometheus-grafana 30002:80
```

- 접속 경로: `http://localhost:30002`
- 계정: `admin` / 추출한 비밀번호
- 주요 확인 메뉴: `Kubernetes / Compute Resources / Namespace (Pods)`

---

## 2. HPA 및 부하 테스트

파드의 리소스(CPU) 사용량을 감지하여 파드 개수를 자동으로 조절하는 HPA를 선언적 방식(YAML)으로 구성하고 트래픽 폭주 상황을 시뮬레이션 해본다.

### 2.1 Metrics Server 활성화

K8S 내부에서 파드의 실시간 리소스 사용량을 수집하기 위한 애드온 활성화.

```bash
minikube addons enable metrics-server
```

### 2.2 Deployment 리소스 제한 및 HPA YAML 선언

명령어 대신 `my-node-app.yaml`에 리소스 기준치(`resources`)와 자동 확장 규칙(`HorizontalPodAutoscaler`)을 명시하여 코드 기반 인프라(IaC)를 구현.

```yaml
# my-node-app.yaml 내 Deployment 및 HPA 선언부 발췌
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-app-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: node-app
  template:
    metadata:
      labels:
        app: node-app
    spec:
      containers:
      - name: node-app-container
        image: my-node-app:1.2
        imagePullPolicy: Never
        ports:
        - containerPort: 3000
        # HPA가 참고할 CPU 기준점 명시
        resources:
          requests:
            cpu: "50m"
          limits:
            cpu: "100m"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: node-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: node-app-deployment
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

```bash
# YAML 설정 클러스터에 반영
kubectl apply -f my-node-app.yaml
```

### 2.3 부하 생성 및 HPA 동작 확인

별도의 터미널에서 무한 루프 스크립트를 통해 대량의 트래픽을 발생시키고, K8S의 스케일아웃 동작을 관찰해본다.

```bash
# 1. HPA 실시간 모니터링 켜기 (터미널 1)
kubectl get hpa -w

# 2. 무한 접속 부하 발생기 실행 (터미널 2)
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://node-app-service; done"
```

- **관찰 대상 1:** `kubectl get hpa` 창에서 CPU 사용량이 50%를 초과하며 REPLICAS(파드 수)가 최대 5개까지 늘어나는 현상 확인.

- **관찰 대상 2:** Grafana 대시보드에서 해당 네임스페이스(`default`)의 CPU 그래프 폭증 및 파드 추가 생성 시각적 확인.

- **종료:** 부하 발생기(터미널 2)에서 프로세스 종료 후, 약 5분에 걸쳐 파드가 다시 2개로 줄어드는 스케일인(Scale-in) 현상 확인.

---
