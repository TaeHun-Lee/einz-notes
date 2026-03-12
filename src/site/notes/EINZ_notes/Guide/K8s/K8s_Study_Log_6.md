---
{"dg-publish":true,"permalink":"/einz-notes/guide/k8s/k8s-study-log-6/","title":"6: GitOps 기반 배포 자동화 (ArgoCD)"}
---


# GitOps 기반 배포 자동화 (ArgoCD)

## 1. ArgoCD 설치 및 GitOps 파이프라인 구축

작업자가 클러스터에 직접 명령을 내리는 수동 방식에서 벗어나, GitHub 저장소의 상태를 K8S 클러스터와 자동으로 동기화하는 GitOps 아키텍처 실습.

### 1.1 ArgoCD 시스템 설치

클러스터 내부에 전용 네임스페이스를 생성하고 공식 매니페스트를 통해 ArgoCD를 설치한다.

```bash
# 1. 네임스페이스 생성
kubectl create namespace argocd

# 2. 공식 배포 설계도(YAML) 적용
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. 파드 정상 구동 대기
kubectl get pods -n argocd -w
````

### 1.2 초기 관리자 비밀번호 추출 및 UI 포트 포워딩

ArgoCD 웹 대시보드 접근을 위해 자동 생성된 초기 비밀번호를 K8S Secret에서 추출하고, 외부 접속을 위한 포트 포워딩을 구성한다.

```bash
# 초기 관리자(admin) 비밀번호 추출 및 Base64 디코딩
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# 포트 포워딩 (ArgoCD는 HTTPS 통신을 위해 기본적으로 443 포트를 사용함)
kubectl port-forward --address 0.0.0.0 svc/argocd-server -n argocd 8080:443
```

### 1.3 브라우저 접속 및 초기 애플리케이션 동기화

- **접속 경로:** `https://localhost:8080` (자체 서명 인증서 경고 발생 시 고급 설정에서 무시하고 진행)

- **로그인 정보:** 계정명 `admin` / 추출한 비밀번호

- **애플리케이션 생성 (Guestbook 예제):**
    - **General:** Name(`guestbook`), Project(`default`), Sync Policy(`Automatic`, Prune 및 Self Heal 활성화)
    - **Source:** Repository URL(`https://github.com/argoproj/argocd-example-apps.git`), Revision(`HEAD`), Path(`guestbook`)
    - **Destination:** Cluster URL(`https://kubernetes.default.svc`), Namespace(`default`)
- **결과:** 생성 즉시 GitHub 저장소의 YAML을 읽어와 클러스터에 Deployment와 Service를 자동 배포하며, 시각화된 토폴로지를 제공.

---

## 2. 자가 치유(Self-Heal) 및 구성 표류(Drift) 방어 테스트

클러스터 내부에 수동 조작이나 장애가 발생했을 때, ArgoCD가 GitHub(Source of Truth)의 상태를 기준으로 자동 복구하는 메커니즘 검증.

### 2.1 고의적 삭제 복구 테스트

운영 중인 디플로이먼트를 강제로 삭제하여 장애 상황을 시뮬레이션 해본다.

```bash
# 수동으로 리소스 삭제
kubectl delete deployment guestbook-ui
```

- **검증:** ArgoCD UI에서 즉시 상태 불일치(Out of Sync)를 감지하고, 관리자의 개입 없이 자동으로 디플로이먼트와 파드를 재생성하여 원상 복구.

### 2.2 임의 스케일링 방어 테스트

파드 개수를 수동으로 조작하여 설정 위반 상황을 유도.

```bash
# 파드 개수를 강제로 5개로 증가
kubectl scale deployment guestbook-ui --replicas=5
```

- **검증:** K8S가 파드를 5개로 늘리려 시도하나, ArgoCD가 GitHub 설계도(기본 1개)와 다름을 감지하고 즉시 불필요한 파드 4개를 강제 종료(Terminate)시켜 원래 상태로 되돌림.

---
