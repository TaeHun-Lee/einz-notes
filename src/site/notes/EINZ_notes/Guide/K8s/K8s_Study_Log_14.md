---
{"dg-publish":true,"permalink":"/einz-notes/guide/k8s/k8s-study-log-14/","title":"14: Minikube에서 Kubeadm으로: 멀티 노드(Multi-Node) 구축 가이드"}
---


# Minikube에서 Kubeadm으로: 멀티 노드(Multi-Node) 구축

## 1. 인프라 준비 (VirtualBox VM 2대)

실무 환경과 동일한 분산 시스템 구축을 위해 마스터 노드와 워커 노드를 각각 분리하여 생성한다.

- **VM 사양 (공통):** CPU 2 Core, RAM 4GB, Disk 25GB 이상

- **운영체제:** Ubuntu Server 22.04 또는 24.04

- **호스트네임:** `k8s-master`, `k8s-worker1`

- **네트워크 어댑터 설정:**
    - 어댑터 1: `NAT` (인터넷 연결 및 패키지 다운로드용)
	- 어댑터 2: `호스트 전용 어댑터(Host-Only)` (노드 간 K8s 내부 통신 및 SSH 접속용)

---

## 2. OS 커널 및 네트워크 기초 작업

**(마스터, 워커 양쪽 노드 모두 실행)**

### 2-1. Swap 메모리 비활성화

Kubelet의 정상적인 메모리 관리를 위해 리눅스 Swap 기능을 완전히 Off 해야함.

```Bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
- `-a`: 시스템에 활성화된 모든 스왑 장치와 파일의 사용을 중지하는 옵션이다.
- `-i`: `sed` 명령어 사용 시, 텍스트 편집기를 열지 않고 대상 파일(`/etc/fstab`) 원본을 직접 수정하는 옵션이다. (재부팅 시 스왑이 다시 켜지는 것을 막기 위해 주석 처리함)

**Swap(스왑) 메모리란 무엇인가?**

컴퓨터의 RAM(주기억장치)은 아주 빠르지만 용량이 제한적임. 만약 서버에 프로그램(Pod)이 너무 많이 떠서 RAM이 꽉 차면 원래대로라면 메모리 부족(OOM: Out Of Memory)으로 컴퓨터가 뻗어버리거나 프로그램이 강제 종료돼야 한다.

이걸 막기 위해 리눅스가 쓰는 방법 중 하나가 **Swap(스왑)** 으로, **하드디스크(SSD/HDD)의 일부 공간을 떼어내서 마치 가짜 RAM처럼 사용하는 기능**임. RAM이 꽉 차면, 당장 안 쓰는 데이터를 하드디스크(Swap 공간)로 잠시 쫓아내서 RAM에 빈자리를 만드는 것.

K8s의 코어 에이전트인 **Kubelet**은 기본적으로 Swap이 1kb라도 켜져 있으면 아예 부팅을 거부하고 죽어버리게 설계되어 있기 때문에 Swap을 Off해야 한다. 이유는 다음과 같다.

1. 성능의 예측 불가능성 (속도 저하): HDD는 기본적으로 RAM보다 훨씬 느리기 때문에 Kubelet 입장에서 성능의 예측이 불가능해짐.

2. 철저한 자원 통제(cgroup) 실패: K8s는 "할당된 메모리 한도를 넘기는 Pod는 죽여버린다(OOMKill)"는 규칙을 가지고 있는데, Swap이 켜져 있으면 메모리를 초과해서 써야 할 파드가 죽지 않고 하드디스크(Swap)로 숨어 들어가서 계속 살아남아 버린다. Kubelet 입장에서는 파드들이 진짜 메모리를 얼마나 쓰고 있는지 정확한 통제와 계산을 할 수 없게 됨.

### 2-2. 필수 커널 모듈 로드

컨테이너 레이어 관리(`overlay`)와 네트워크 패킷 필터링(`br_netfilter`) 모듈을 커널에 적재한다.

```Bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```
- `modprobe`: 리눅스 커널에 특정 모듈(여기서는 `overlay`와 `br_netfilter`)을 즉시 로드하는 명령어다.

### 2-3. IPv4 포워딩 및 iptables 활성화

서로 다른 노드에 위치한 파드들이 통신하려면 리눅스 커널이 네트워크 패킷을 올바르게 전달(포워딩)하도록 설정해야 한다.

```Bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```
- `--system`: `/etc/sysctl.d/` 등 시스템의 설정 디렉토리에 있는 모든 커널 파라미터 파일을 다시 읽어들여 즉시 적용하라는 옵션이다.

---

## 3. 컨테이너 런타임 (containerd) 설치

**(마스터, 워커 양쪽 노드 모두 실행)**

쿠버네티스가 Pod(컨테이너)를 실행할 때 사용할 엔진, 즉 컨테이너 런타임(Container Runtime)을 설치하는 과정이다. 과거에는 Docker를 직접 사용했지만, 현재는 더 가볍고 표준화된 `containerd`를 사용하는 것이 실무 표준이다. 파드를 실행할 엔진인 `containerd`를 설치하고, K8s와 자원 관리자(cgroup)를 일치시킨다.

```Bash
# 1. Docker 공식 GPG 키 및 저장소 추가
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 2. containerd 설치
sudo apt-get update
sudo apt-get install -y containerd.io

# 3. SystemdCgroup 설정 활성화
# containerd 기본 설정 파일을 생성하여 지정된 경로에 저장한다.
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# 설정 파일 내용 중 SystemdCgroup 옵션을 false에서 true로 바꾼다.
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# 4. 서비스 재시작 및 자동 실행 등록
sudo systemctl restart containerd
sudo systemctl enable containerd
```
- `-m 0755`: 디렉토리를 생성할 때 접근 권한을 755(소유자는 읽기/쓰기/실행, 나머지는 읽기/실행)로 설정하는 옵션이다.
- `-p`: 지정한 경로의 상위 디렉토리가 없다면 에러 없이 함께 생성하는 옵션이다.
- `--dearmor`: 다운로드한 텍스트 형태의 GPG 키를 apt 패키지 관리자가 인식할 수 있는 바이너리 형식으로 변환하는 옵션이다.
- `-o`: 변환된 데이터를 저장할 출력(Output) 파일의 경로를 지정하는 옵션이다.
- `$(dpkg --print-architecture)`: 현재 OS의 CPU 아키텍처(예: amd64)를 자동으로 확인하여 문자열로 치환하는 명령어다.
- `config default`: containerd의 기본 설정값을 터미널 화면에 출력하는 명령어다. 이를 `tee`를 통해 파일로 덮어쓴다.

설치가 완료되면 아래 명령어로 `containerd`가 정상적으로 실행 중인지 확인한다.

```Bash
systemctl status containerd
```

출력 결과에 `Active: active (running)`이라고 표시되면 성공.

---

## 4. K8s 필수 컴포넌트 설치

**(마스터, 워커 양쪽 노드 모두 실행)**

K8s 1.30 버전의 `kubeadm`, `kubelet`, `kubectl`을 설치하고 의도치 않은 자동 업데이트를 방지한다.
- **kubeadm:** K8s 클러스터를 초기화(init)하고 워커 노드를 연결(join)하는 구축 전용 관리 도구.
- **kubelet:** 모든 노드에 상주하며 컨테이너 런타임(containerd)과 통신하여 파드의 실행 상태를 관리하는 핵심 에이전트.
- **kubectl:** 완성된 클러스터에 API 요청을 보내 리소스를 조작하는 커맨드라인(CLI) 도구.

```Bash
# 1. 필수 패키지 설치
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# 2. 구버전 키 링 디렉토리가 없다면 생성
sudo mkdir -p /etc/apt/keyrings

# 3. K8s 1.30 버전용 퍼블릭 서명 키 다운로드 및 변환
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# 4. apt 패키지 매니저에 K8s 1.30 저장소 추가
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# 5. 저장소 업데이트 및 3개 패키지 설치
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# 6. 패키지 자동 업데이트 방지 (버전 고정)
sudo apt-mark hold kubelet kubeadm kubectl
```
- `apt-transport-https`: 패키지 관리자(`apt`)가 HTTPS 프로토콜을 사용하는 저장소(repository)에 안전하게 접근하여 패키지를 다운로드할 수 있도록 지원하는 패키지다.
- `apt-mark hold`: `apt-get upgrade` 명령어를 실행했을 때, 명시된 패키지(`kubelet`, `kubeadm`, `kubectl`)가 의도치 않게 최신 버전으로 자동 업데이트되는 것을 차단(잠금)하는 명령어다. K8s는 마스터와 워커 간의 버전 호환성에 매우 민감하므로 버전 고정이 필수적이다.

설치가 완료되면 양쪽 노드에서 동일한 버전이 설치되었는지 확인한다.

```Bash
kubeadm version
kubectl version --client
```

---

## 5. 클러스터 생성 및 네트워크(CNI) 구성

**(마스터 노드에서만 실행)**

### 5-1. 클러스터 초기화 (kubeadm init)

마스터 노드에서 아래 명령어를 실행하여 클러스터를 생성한다. `<마스터노드IP>` 부분에는 `ip addr`을 통해 확인한 마스터 VM의 IP(예: `192.168.56.10`)를 정확히 입력한다.

```Bash
# 1. 마스터 노드 IP 확인
ip addr

# 2. Kubeadm 클러스터 초기화
sudo kubeadm init --apiserver-advertise-address=<마스터노드IP> --pod-network-cidr=192.168.0.0/16
```
- `--apiserver-advertise-address`: K8s의 핵심인 API 서버가 워커 노드들의 통신을 받아들일 IP 주소를 명시적으로 지정하는 옵션이다. VirtualBox의 호스트 전용 어댑터 IP를 지정하여 통신 혼선을 막는다.
- `--pod-network-cidr=192.168.0.0/16`: 클러스터 내부에서 생성될 파드(Pod)들이 부여받을 IP 대역을 지정하는 옵션이다. 추후 설치할 Calico 네트워크 플러그인의 기본 요구값에 맞춰 설정한다.

성공하면 `Your Kubernetes control-plane has initialized successfully!`라는 메시지가 출력된다.

**성공했을 경우 `kubeadm join ...`으로 시작하는 긴 메세지가 출력된다. 이것은 워커 노드가 마스터에 합류하기 위한 보안 토큰과 Hash값이 포함된 비밀번호다. 이 줄을 반드시 복사해서 Windows 메모장 등에 안전하게 백업해 둔다.**

### 5-2. Kubeconfig 인증 설정

일반 계정에서 `kubectl` 명령어를 사용할 수 있도록 권한을 부여하기 위해 config에 반영한다.

```Bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
- `-i`: `cp` 명령어 사용 시, 대상 경로에 이미 같은 이름의 파일이 존재할 경우 덮어쓸지 확인 프롬프트를 띄우는 옵션이다.
- `$(id -u):$(id -g)`: 현재 로그인한 사용자의 고유 ID(UID)와 그룹 ID(GID)를 자동으로 가져와서, 복사한 설정 파일의 소유권을 root에서 현재 계정으로 변경하는 명령어다.

설정이 끝났으면 마스터 노드에서 아래 명령어로 클러스터 상태를 확인한다.

```Bash
kubectl get nodes
```

현재는 마스터 노드 1개만 보이며, 아직 네트워크 플러그인을 설치하지 않았기 때문에 상태가 `NotReady`로 표시되는 것이 정상이다.

### 5-3. Calico CNI (네트워크 플러그인) 설치

파드 간의 가상 네트워크 통신망을 구축한다. 이 작업 후 마스터 노드가 `Ready` 상태로 변경되는 것을 확인하면 된다.

```Bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

`created` 메세지가 여러 줄 노출된 이후 Pod들이 생성되고 네트워크 망이 구성되기 까지 1~2분 정도 소요된다.

```Bash
# 1. 노드 상태 확인 (Ready로 바뀌었는지 확인)
kubectl get nodes

# 2. 시스템 파드(Calico, CoreDNS 등) 상태 확인
kubectl get pods -n kube-system
```

만약 CoreDNS Pod가 Running으로 노출된다면 설치가 완료된 것이다.

---

## 6. 워커 노드 클러스터 합류 (Join)

**(워커 노드에서만 실행)**

마스터 노드 초기화(`kubeadm init`) 완료 시 출력된 `join` 이하 저장해두었던 Token 값과 Hash 값을 워커 노드 터미널에 입력한다.

```Bash
sudo kubeadm join 192.168.56.101:6443 --token <토큰값> --discovery-token-ca-cert-hash sha256:<해시값>
```

성공했을 경우 `This node has joined the cluster` 메세지가 노출된다.

> **상태 확인 (마스터 노드에서 실행):** `kubectl get nodes` 명령어로 `k8s-master`와 `k8s-worker1` 두 개가 모두 보이고, 모든 노드가 `Ready` 상태인지 확인.

---

## 7. 가동 테스트 (Nginx 배포 및 네트워크 접속)

**(마스터 노드에서만 실행)**

클러스터가 정상적으로 트래픽을 라우팅하는지 확인하기 위해 Nginx 파드를 배포하고 3가지 방식으로 접속을 테스트한다.

```Bash
# 1. Nginx 배포 및 스케일 아웃(파드 2개)
kubectl create deployment nginx-test --image=nginx:alpine
kubectl scale deployment nginx-test --replicas=2

# 2. NodePort 서비스 생성 (외부 접속 개방)
kubectl expose deployment nginx-test --type=NodePort --port=80

# 3. 정보 확인 (파드 IP, 서비스 IP, NodePort 번호 등)
kubectl get pods -o wide
kubectl get svc nginx-test
```

- `kubectl get pods -o wide` 출력 결과의 **`NODE`** 열을 확인했을 때, 마스터 노드는 기본적으로 Pod를 배정받지 않도록 보호(Taint)되어 있기 때문에, **두 개의 파드 모두 `worker-virtualbox`에 할당**되어 `Running` 상태로 떠 있어야 정상.

- `kubectl get svc nginx-test` 출력 결과의 `PORT(S)` 열을 보면 `80:3xxxx/TCP` 형태로 30000번대의 랜덤한 포트 번호(예: 31234)가 할당된 것을 확인. 이 5자리 포트 번호를 저장하거나 기억해둔다.

### 7.1 3가지 네트워크 접속 테스트 방법

```Bash
# 방법 A (외부 접속/NodePort): <노드IP>:<3xxxx포트>
# Worker VM의 ip addr 결과로 노출되는 IP를 입력하면 된다.
curl http://<WorkerVM의 IP>:31508

# 방법 B (내부 통신/Pod IP): <파드IP>:80
curl http://<Calico 네트워크가 부여한 Pod의 내부통신용 IP>:80

# 방법 C (로드밸런싱/ClusterIP): <서비스IP>:80
curl http://<로드밸런싱 서비스 전용 가상 IP>:80
```

모든 방식에서 `Welcome to nginx!`가 출력되면 클러스터 구축 성공.

---
