---
{"dg-publish":true,"permalink":"/einz-notes/guide/k8-s/k8-s-study-log-2/","title":"2: 실습 환경 구축 및 배포 가이드"}
---


# Kubernetes 실습 환경 구축 및 배포 가이드

## 1: 가상 머신(Ubuntu) 하드웨어 설정

1. **VirtualBox 실행** 후 '새로 만들기' 클릭.
2. **ISO Image:** 다운로드한 Ubuntu 24.04 LTS 파일 선택.
3. **사용자 설정:** "Proceed with attended Installation"에 체크 해제.
4. **Hardware 설정:**
    - **RAM:** 8192 MB (8GB)
    - **CPU:** 4 Cores
5. **설치 진행:** Ubuntu 설치 화면에서 "Minimal installation" 선택 및 설치 완료 후 재부팅.

---

## 2: 호스트-게스트 공유 및 필수 도구 설치

Ubuntu 바탕화면 진입 후 터미널(`Ctrl+Alt+T`)에서 실행한다.

1. **복사 붙여넣기 활성화:**
	- VM 상단 메뉴: '장치' -> '클립보드 공유' -> '양방향' 설정.

2. **게스트 확장 설치를 위한 필수 패키지 설치:**

    ```bash
    sudo apt update
    sudo apt install -y bzip2 tar build-essential dkms linux-headers-$(uname -r)
    ```

3. **게스트 확장 설치:**
    - '장치' -> '게스트 확장 CD 이미지 삽입' 클릭.

    ```bash
    cd /media/$USER/VBox_GAs_*
    sudo ./VBoxLinuxAdditions.run
    sudo reboot
    ```

---

## 3: Docker 및 K8S 도구 설치

1. **Docker 설치 및 권한 설정:**

    ```bash
    sudo apt install -y docker.io
    sudo usermod -aG docker $USER
    newgrp docker  # 재로그인 없이 그룹 권한 적용
    ```
    
2. **Minikube & kubectl 설치:**

    ```bash
    # Minikube 다운로드 및 설치
    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    sudo install minikube-linux-amd64 /usr/local/bin/minikube
    
    # kubectl 설치
    sudo snap install kubectl --classic
    ```


---

## 4: 클러스터 구동 및 애플리케이션 배포

1. **K8S 클러스터 시작:**

    ```bash
    minikube start --driver=docker
    ```

2. **YAML 설계도 작성 (`my-nginx.yaml`):**

    ```bash
    mkdir ~/k8s-study && cd ~/k8s-study
    nano my-nginx.yaml
    ```
    
    _(YAML 내용: Deployment replicas: 2 및 Service NodePort 30001 설정 포함)_
    
3. **설계도 적용:**

    ```bash
    kubectl apply -f my-nginx.yaml
    ```

---

## 5: 외부(Windows) 접속을 위한 네트워크 터널링

이 단계는 윈도우 브라우저에서 접속하기 위해 필수적이다.

1. **K8S 포트 포워딩 (Ubuntu 터미널):**

    ```bash
    # 터미널 하나를 점유하므로 끄지 말 것
    kubectl port-forward --address 0.0.0.0 service/nginx-service 8888:80
    ```

2. **VirtualBox 포트 포워딩 (Windows 설정):**

    - VM [설정] -> [네트워크] -> [어댑터 1] -> [고급] -> [포트 포워딩]
    - **규칙 추가:** 호스트 IP `127.0.0.1`, 호스트 포트 `8888`, 게스트 포트 `8888`

3. **브라우저 확인:** Windows 크롬에서 `http://localhost:8888` 접속.

---

## 6: 실습 및 트러블슈팅 정리

1. **자가 치유 테스트:** `kubectl delete pod [이름]` 실행 후 새 파드가 생성되는지 확인.
2. **ConfigMap 주입:** 외부 파일을 파드 내부에 마운트할 때, 해당 파일은 **Read-only** 상태임을 인지할 것.
3. **권한 에러 발생 시:** `kubectl` 명령 시 `connection refused`가 발생하면 `sudo`를 빼고 실행하거나 `KUBECONFIG` 경로를 확인할 것.

---