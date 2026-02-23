---
{"dg-publish":true,"permalink":"/einz-notes/guide/k8-s/k8-s-study-log-1/","title":"K8S 스터디 로그 1"}
---


## Kubernetes 학습 정리 (Windows 11 실습 환경)

### Step 1: 개념 이해 및 환경 설계

- **Kubernetes 정의:** 컨테이너화된 애플리케이션의 배포, 확장 및 관리를 자동화하는 오케스트레이션 도구임.

- **환경 구성:** Windows 11 호스트 OS 위에 VirtualBox 가상 머신(Ubuntu 24.04 LTS)을 설치하여 완전한 시스템 격리 환경을 구축함.

- **리소스 할당:** Intel Core Ultra 9 285H 사양을 고려하여 가상 머신에 CPU 4 Cores, RAM 8GB를 할당하여 안정성을 확보함.

### Step 2: 격리 환경(VirtualBox) 구축 및 설정

- **OS 설치:** Ubuntu ISO 이미지를 수동으로 마운트하여 설치함.

- **게스트 확장(Guest Additions):** 호스트-게스트 간 복사 붙여넣기 및 화면 공유를 위해 설치함. 과정 중 `bzip2`, `build-essential` 등 필수 라이브러리를 선제적으로 설치함.

- **네트워크 설정:** NAT 네트워크 환경에서 윈도우(호스트)와 우분투(게스트) 간의 통신을 위해 포트 포워딩(8888 -> 8080/30001, 2222 -> 22)을 설정함.

### Step 3: K8S 핵심 도구 설치 및 클러스터 구동

- **Docker 설치:** 컨테이너 런타임 엔진으로 Docker를 설치하고, `sudo` 없이 명령어를 사용하도록 권한 설정을 완료함.
```bash
# 1. 'docker'라는 이름의 그룹을 생성
sudo groupadd docker

# 2. 현재 로그인한 사용자($USER)를 docker 그룹에 추가
sudo usermod -aG docker $USER

# 3. 그룹 변경 사항을 현재 셸 세션에 즉시 적용함
newgrp docker
```

- **Minikube & kubectl:** 로컬 K8S 클러스터 구동기인 `minikube`와 클러스터 조종 도구인 `kubectl`을 설치함.

- **클러스터 시작:** `minikube start --driver=docker` 명령을 통해 가상 머신 내부에서 싱글 노드 클러스터를 활성화함.

### Step 4: 서비스 배포 및 객체 관리 실습

- **Pod & Deployment:** `nginx` 이미지를 사용하여 파드를 생성하고, `replicas: 2` 설정을 통해 고가용성 설계를 실습함.

- **Service (NodePort/LoadBalancer):** 생성된 파드를 외부로 노출하기 위해 Service 객체를 생성함. `port-forward` 명령을 통해 3중 네트워크 격리 환경을 뚫고 윈도우 브라우저에서 접속에 성공함.
	- 실습 환경은 아래와 같은 계층으로 격리되어 있어 단계별 포트 포워딩이 필요함.
	1. **L1 (K8S 내부):** 파드 간 통신만 가능하며 외부 접근이 원천 차단됨.
	2. **L2 (Minikube/VM 내부):** 가상 머신(Ubuntu) 내에서는 접근 가능하나, 호스트 OS(Windows)에서는 접근 불가함.
	3. **L3 (Host OS):** Windows 브라우저가 위치한 계층임.

	- **kubectl port-forward:** Kubernetes API를 사용하여 호스트 OS와 K8S 서비스 간의 논리적 터널을 생성함.
	```bash
	kubectl expose deployment nginx-deployment --type=NodePort --port=80
	kubectl port-forward --address 0.0.0.0 service/nginx-service 8888:80
	```
	- **VirtualBox Port Forwarding:** 가상 머신 소프트웨어 설정을 통해 Windows의 특정 포트(8888)로 들어온 신호를 Ubuntu의 포트(30001)로 전달하도록 물리적 통로를 지정함.


- **YAML 기반 IaC:** 명령어 방식이 아닌 YAML 파일을 작성하여 인프라를 코드로서 정의하고 `kubectl apply -f` 명령으로 반영함.

### Step 5: 고급 기능 테스트 및 트러블슈팅

- **자가 치유(Self-healing):** 실행 중인 파드를 강제로 삭제하여 K8S가 설계도(Replicas)에 따라 자동으로 파드를 복구하는 과정을 확인함.
```bash
kubectl delete [pod 이름]
```

- **ConfigMap 주입:** 외부 HTML 파일을 ConfigMap으로 만들어 컨테이너 내부에 주입함. 이때 발생한 `Read-only file system` 에러를 통해 K8S의 보안 및 파일 시스템 마운트 원리를 학습함.

- **로드 밸런싱:** 여러 개의 파드 내용(I am Pod 1/2)을 각각 수정하여 서비스가 트러블을 분산 처리하는 것을 확인함.

---

## 향후 실습 과제 (Next Steps)

1. **영구 저장소(PV/PVC) 적용:** 파드 재시작 시에도 데이터가 유지되도록 가상 디스크 연결.
2. **리소스 제한(Limit/Request):** 가상 머신의 자원을 효율적으로 쓰기 위한 제한 설정.
3. **애플리케이션 컨테이너화:** 직접 만든 소스 코드를 Dockerfile로 빌드하여 K8S에 배포.

---
*Last Updated: 2026-02-23*