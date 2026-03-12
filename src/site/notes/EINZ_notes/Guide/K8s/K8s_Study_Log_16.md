---
{"dg-publish":true,"permalink":"/einz-notes/guide/k8s/k8s-study-log-16/","title":"16: Minikube에서 Kubeadm으로: 영구 스토리지(NFS) 구축"}
---


# Minikube에서 Kubeadm으로: 영구 스토리지(NFS) 구축

## 1. 다중 랜카드 환경의 Kubelet IP 통신 오류 해결

VirtualBox 환경에서 K8s 마스터 노드가 워커 노드의 파드에 접속 시도 시 `pod does not exist` 에러가 발생하는 현상을 해결한다.

**원인:** K8s 클러스터 내의 모든 노드가 VirtualBox의 첫 번째 랜카드인 NAT IP(`10.0.2.15`)를 기본 `INTERNAL-IP`로 등록하여 노드 간 IP 충돌 및 라우팅 오류가 발생함.

### 1-1. 마스터 노드 Kubelet IP 강제 지정 (마스터 노드에서 실행)

```Bash
# kubelet 환경 변수 파일에 마스터 노드의 호스트 전용 IP를 명시함
echo 'KUBELET_EXTRA_ARGS="--node-ip=192.168.56.101"' | sudo tee /etc/default/kubelet

# systemd 설정 파일을 다시 읽어들임
sudo systemctl daemon-reload

# kubelet 데몬을 재시작하여 변경된 IP를 적용함
sudo systemctl restart kubelet
```

### 1-2. 워커 노드 Kubelet IP 강제 지정 (워커 노드에서 실행)

```Bash
# kubelet 환경 변수 파일에 워커 노드의 호스트 전용 IP(예: 102)를 명시함
echo 'KUBELET_EXTRA_ARGS="--node-ip=192.168.56.102"' | sudo tee /etc/default/kubelet

# systemd 설정 파일을 다시 읽어들임
sudo systemctl daemon-reload

# kubelet 데몬을 재시작하여 변경된 IP를 적용함
sudo systemctl restart kubelet
```

### 1-3. 상태 확인 (마스터 노드에서 실행)

```Bash
# 노드의 INTERNAL-IP가 192.168.56.x 대역으로 정상 분리되었는지 확인함
kubectl get nodes -o wide
```
- `-o wide` (`--output wide`): 기본 출력 정보 외에 내부 IP, 외부 IP, OS 이미지 등 추가적인 상세 정보를 포함하여 출력하는 옵션.

---

## 2. NFS (Network File System) 서버 구축

파드가 삭제되거나 워커 노드가 재부팅되어도 데이터가 날아가지 않도록, 마스터 노드의 디스크 일부를 공용 네트워크 폴더로 구성한다.

### 2-1. NFS 서버 설치 및 폴더 권한 설정 (마스터 노드에서 실행)

```Bash
# 1. NFS 커널 서버 패키지를 설치함
sudo apt-get update
sudo apt-get install -y nfs-kernel-server

# 2. 데이터를 저장할 디렉토리를 생성함
sudo mkdir -p /srv/nfs

# 3. K8s의 다양한 파드가 접근할 수 있도록 디렉토리 소유자를 권한 없음 상태로 변경함
sudo chown nobody:nogroup /srv/nfs

# 4. 모든 사용자가 읽기, 쓰기, 실행을 할 수 있도록 권한을 개방함
sudo chmod 777 /srv/nfs
```
- `-y` (`--yes`): 설치 도중 나타나는 확인 프롬프트에 자동으로 '예'를 입력하는 옵션.
- `-p` (`--parents`): 생성하려는 디렉토리의 상위 경로가 존재하지 않으면 상위 경로까지 한 번에 생성하는 옵션.

### 2-2. NFS 외부 접근 허용 (마스터 노드에서 실행)

```Bash
# 1. NFS 설정 파일에 공유 폴더 경로와 권한 옵션을 추가함
echo "/srv/nfs *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

# 2. 변경된 NFS 공유 설정을 시스템에 적용함
sudo exportfs -a

# 3. NFS 서버를 재시작함
sudo systemctl restart nfs-kernel-server
```
- `-a` (`--append`): `tee` 명령어 사용 시, 기존 파일의 내용을 덮어쓰지 않고 파일의 맨 끝에 내용을 추가하는 옵션.
- `-a` (`--all`): `exportfs` 명령어 사용 시, `/etc/exports` 파일에 정의된 모든 디렉토리를 공유 목록에 즉시 등록하는 옵션.
- `rw`: 읽기 및 쓰기 권한 허용.
- `sync`: 메모리가 아닌 물리적 디스크에 데이터가 완전히 기록될 때까지 대기하여 데이터 유실을 방지.
- `no_root_squash`: 원격 접속 클라이언트의 root 사용자를 서버의 root 사용자와 동일한 권한으로 인정함.

### 2-3. NFS 클라이언트 설치 (워커 노드에서 실행)

```Bash
# 워커 노드에서 마스터 노드의 공유 폴더를 마운트하기 위해 필수 패키지를 설치함
sudo apt-get update
sudo apt-get install -y nfs-common
```

---

## 3. Helm 설치 및 StorageClass 설정

K8s 클러스터 내에서 영구 볼륨(PV)을 동적으로 생성해 주는 프로비저너를 설치한다.

### 3-1. Helm 패키지 매니저 설치 (마스터 노드에서 실행)

```Bash
# 1. Helm 설치 스크립트를 다운로드함
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

# 2. 스크립트 실행 권한을 부여함
chmod 700 get_helm.sh

# 3. 스크립트를 실행하여 Helm을 설치함
./get_helm.sh
```
- `-f` (`--fail`): 서버 오류(400 이상) 발생 시 내용을 출력하지 않고 조용히 실패 처리하는 옵션.
- `-s` (`--silent`): 진행률 표시줄을 숨기는 옵션.
- `-S` (`--show-error`): `-s`와 함께 쓰여 진행률은 숨기되, 에러 발생 시 에러 메시지는 출력하는 옵션.
- `-L` (`--location`): 서버가 리다이렉션(30X)을 지시하면 해당 새 위치로 따라가서 다운로드하는 옵션.
- `-o` (`--output`): 다운로드한 내용을 지정한 파일 이름으로 저장하는 옵션.

### 3-2. NFS 프로비저너 배포 (마스터 노드에서 실행)

```Bash
# 1. NFS 외부 프로비저너 Helm 저장소를 추가함
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

# 2. 저장소 정보를 최신 상태로 갱신함
helm repo update

# 3. 마스터 노드의 IP와 공유 경로를 지정하여 프로비저너를 설치함
helm install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=192.168.56.101 \
  --set nfs.path=/srv/nfs \
  --set storageClass.name=nfs-client \
  --set storageClass.defaultClass=true
```
- `--set`: Helm 차트 내부에 사전 정의된 설정(values)을 사용자가 원하는 값으로 덮어쓰는 옵션.

---

## 4. 데이터 영구 보존 검증 (마스터 노드에서 실행)

PVC를 통해 스토리지를 요청하고 파드를 배포하여 데이터가 물리적 디스크에 저장되는지 확인한다.

### 4-1. PVC 및 테스트 파드 생성

```Bash
# 매니페스트 파일을 작성하여 K8s 클러스터에 적용함
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: test-container
    image: busybox
    command: [ "sleep", "3600" ]
    volumeMounts:
    - name: nfs-volume
      mountPath: /data
  volumes:
  - name: nfs-volume
    persistentVolumeClaim:
      claimName: test-pvc
EOF
```
- `-f` (`--filename`): 생성할 리소스의 설정 파일 경로를 지정하는 옵션. `-`는 파이프라인(`|`)을 통해 전달받은 표준 입력을 파일 대신 읽어들임을 의미함.

### 4-2. 파드 내부에 데이터 기록 및 호스트 확인

```Bash
# 1. 실행 중인 테스트 파드 내부에 쉘 명령을 내려 텍스트 파일을 생성함
kubectl exec test-pod -- sh -c "echo 'Hello NFS Storage!' > /data/hello.txt"

# 2. 마스터 노드의 실제 물리적 공유 디렉토리에 파일이 생성되었는지 확인 및 출력함 (*는 자동 매칭)
cat /srv/nfs/default-test-pvc-*/hello.txt
```
- `exec`: 실행 중인 컨테이너 내부에 접속하거나 외부에서 특정 명령을 전달하여 실행하게 하는 명령어.
- `--`: K8s `kubectl` 명령어 옵션의 끝을 나타내며, 이후 작성되는 내용은 순수하게 컨테이너 내부 리눅스 환경으로 전달할 명령어임을 명시함.
- `-c` (`--command`): `sh` 쉘 환경에서 명령어를 실행할 때, 뒤따라오는 문자열 데이터 전체를 단일 명령어로 인식하여 즉시 실행하도록 지시하는 옵션.

### 4-3. Worker 노드에서 파드 내부에 데이터 기록 및 호스트 확인

```Bash
# 운영체제가 인식하는 모든 마운트 정보 중 nfs가 들어간 줄만 출력
mount | grep nfs
```

결과물로 출력되는 `192.168.56.101:/srv/nfs/default-test-pvc-pvc... on /var/lib/kubelet/pods/<마운트 Pod ID>/volumes/kubernetes.io~nfs/pvc-<마운트 PVC ID>...`
에서 마운트된 Pod Id와 PVC Id를 추출해서 출력해본다.

```Bash
# 복사한 경로로 cat 명령어 실행
sudo cat /var/lib/kubelet/pods/<마운트 Pod ID>/volumes/kubernetes.io~nfs/pvc-<마운트 PVC ID>/hello.txt
```

**검증** : `Hello NFS Storage!` 출력

---
