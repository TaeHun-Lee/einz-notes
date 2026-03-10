---
{"dg-publish":true,"permalink":"/einz-notes/guide/k8-s/k8-s-study-log-10/","title":"10: Argo Rollouts"}
---


# Argo Rollouts 기반 Canary 배포

Argo Rollouts를 도입하여 무중단 고급 배포 전략인 카나리(Canary) 배포를 구현한 과정 요약.

## 1. Argo Rollouts 제어 환경 구성

Kubernetes 클러스터에 배포를 통제할 컨트롤러를 띄우고, 로컬 터미널에 제어용 플러그인을 설치한다.

### 1.1 컨트롤러 설치

```Bash
# argo-rollouts 전용 네임스페이스 생성
kubectl create namespace argo-rollouts

# Argo Rollouts 컨트롤러 설치 매니페스트 적용
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

### 1.2 Kubectl 플러그인 설치

터미널에서 Rollout 상태를 모니터링하고 제어하기 위한 CLI 도구를 설치한다.

```Bash
# 플러그인 바이너리 다운로드
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64

# 실행 권한 부여
chmod +x ./kubectl-argo-rollouts-linux-amd64

# 전역 사용을 위해 실행 경로로 이동
sudo mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

# 설치 정상 완료 확인
kubectl argo rollouts version
```

## 2. Helm Chart Manifest 수정 (Deployment -> Rollout)

기존의 기본 배포 방식인 `Deployment`를 Argo Rollouts가 인식할 수 있는 `Rollout` 리소스로 변경한다.

트래픽의 50%만 새 버전으로 교체한 뒤 무기한 대기(Pause)하도록 카나리 전략을 추가한다.

### `my-node-app-chart/templates/deployment.yaml` 변경

```YAML
# apiVersion을 argoproj.io로 변경
apiVersion: argoproj.io/v1alpha1
# kind를 Deployment에서 Rollout으로 변경
kind: Rollout
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  
  # 카나리 배포 전략 설정 부분
  strategy:
    canary:
      steps:
      # 1단계: 전체 레플리카의 50%만 새 버전으로 배포
      - setWeight: 50
      # 2단계: 관리자가 수동으로 승인(Promote)할 때까지 대기
      - pause: {}

  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: node-app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 3000
```

## 3. 리소스 정리 및 GitOps 동기화

Kubernetes 리소스 타입 충돌 에러를 방지하기 위해 기존 리소스를 정리하고 변경 사항을 반영한다.

```Bash
# 기존에 구동 중이던 Deployment 리소스 강제 삭제
kubectl delete deployment my-node-app-deployment

# Helm Chart 변경 사항을 Git 저장소에 푸시
git add my-node-app-chart/
git commit -m "Change Deployment to Rollout for Canary strategy"
git push origin master
```

_참고: Rollout 리소스의 최초 생성 시점(Revision 1)에는 트래픽을 나눌 구버전이 존재하지 않으므로, 카나리 단계(Step)를 무시하고 즉시 100% 배포 상태(Healthy)가 된다._

## 4. 카나리 배포 검증 및 수동 승인(Promote)

애플리케이션 소스 코드를 수정하여 새로운 배포(Revision 2)를 트리거하고, 카나리 배포의 대기 상태와 승인 과정을 검증한다.

### 4.1 배포 대기(Pause) 상태 확인

소스 코드 업데이트로 인해 CI/CD 파이프라인이 동작하면 터미널에서 아래 명령어로 실시간 상태를 확인한다.

```Bash
# Rollout 리소스의 실시간 배포 상태 모니터링
kubectl argo rollouts get rollout my-node-app-deployment -w
```

- 결과: 새 버전(Revision 2) 파드가 전체의 50%(1개)만 생성된 후 `Paused` 상태로 멈춘다. 구버전 파드는 여전히 동작 중이다.

- 검증: 브라우저에서 서비스 주소 접속 후 새로고침을 반복하면, 트래픽이 구버전과 신버전으로 분산되어 라우팅되는 것을 확인할 수 있다.


### 4.2 배포 최종 승인(Promote)

신규 버전에 문제가 없음을 확인한 후, 남은 50%의 트래픽도 마저 배포하도록 수동 승인 명령을 내린다.

```Bash
# 대기(Pause) 상태를 해제하고 다음 단계(100% 배포)로 진행
kubectl argo rollouts promote my-node-app-deployment
```

### 4.3 100% 배포 완료 확인

승인 직후 배포 모니터링 화면에서 다음 변화를 확인한다.

1. 신규 버전(Revision 2)의 파드가 2개(100%)로 스케일 업(Scaled Up) 된다.

2. 기존 버전(Revision 1)의 파드는 정상적으로 스케일 다운(Scaled Down) 되어 제거된다.
  
3. 최종 상태가 `Healthy`로 전환되며 카나리 배포 사이클이 종료된다.

---
