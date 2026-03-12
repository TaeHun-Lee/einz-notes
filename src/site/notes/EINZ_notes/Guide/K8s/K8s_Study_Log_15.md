---
{"dg-publish":true,"permalink":"/einz-notes/guide/k8s/k8s-study-log-15/","title":"15: Minikube에서 Kubeadm으로: LoadBalancer 및 Ingress 구축"}
---


# Minikube에서 Kubeadm으로: LoadBalancer 및 Ingress 구축

## 1. MetalLB (온프레미스 LoadBalancer)

클라우드 환경(AWS, GCP 등)이 아닌 온프레미스 서버에서 `LoadBalancer` 타입의 서비스를 사용할 수 있도록 외부 IP를 가상으로 할당하는 MetalLB를 설치한다.

### 1-1. kube-proxy strictARP 활성화

MetalLB가 네트워크 ARP 요청을 정상적으로 처리할 수 있도록 K8s 프록시 설정을 변경한다.

```Bash
# kube-proxy 설정 파일을 yaml 형태로 출력하고, sed로 값을 치환한 뒤 다시 클러스터에 적용함
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```
- `-n` (`--namespace`): 명령을 실행할 특정 네임스페이스를 지정하는 옵션.
- `-o` (`--output`): 출력 형식을 지정하는 옵션 (여기서는 yaml 형식으로 출력).
- `-e` (`--expression`): `sed` 명령어에서 실행할 문자열 치환 스크립트를 직접 지정하는 옵션.
- `-f` (`--filename`): 리소스를 생성하거나 업데이트할 때 사용할 파일 또는 URL을 지정하는 옵션. (`-`는 파이프라인을 통해 넘어온 표준 입력을 의미함).

**네트워크 ARP 요청이란?**

ARP(Address Resolution Protocol, 주소 결정 프로토콜)는 논리적인 통신 주소인 **IP 주소를 물리적인 하드웨어 주소인 MAC 주소로 변환**해 주는 네트워크 통신 규약이다.
- **IP 주소:** 네트워크상의 논리적 위치
- **MAC 주소:** 랜(LAN) 카드 기기 자체에 부여된 고유한 물리적 식별 번호

**MetalLB 환경에서 strictARP 옵션이 필요한 이유**

MetalLB는 물리적인 랜 카드가 없는 가상의 IP(`192.168.56.200`)를 LoadBalancer용으로 생성한다. 외부 통신 장비(Router)가 Nginx에 접속하기 위해 "192.168.56.200 누구인가?"라고 ARP 요청을 보낼 때, K8s 클러스터 내의 MetalLB 에이전트가 이를 가로채어 대신 ARP 응답을 보낸다.

이렇게 가상의 IP에 대해 K8s 노드가 정확하게 응답하고 트래픽을 끌고 오도록 제어하기 위해, `kube-proxy`의 `strictARP` 옵션을 켜서 네트워크 패킷 처리 규칙을 엄격하게 맞춘 것이다.

### 1-2. MetalLB 매니페스트 배포

```Bash
# MetalLB 네이티브 버전을 클러스터에 배포함
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml
```

### 1-3. IP 할당 풀(Pool) 및 L2 네트워크 광고 설정

MetalLB가 서비스에 부여할 사내망 IP 대역을 선언한다.

```Bash
# IPAddressPool 및 L2Advertisement 사용자 정의 리소스(CRD)를 생성함
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.56.200-192.168.56.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
EOF
```

**마찬가지로 `nano metallb.yaml` 생성하여 `kubectl apply -f metallb.yaml` 해도 가능**

---

## 2. NGINX Ingress Controller 구축

단일 로드밸런서 IP를 통해 들어오는 트래픽을 도메인 이름(Host)에 따라 서로 다른 파드로 분배하는 L7 라우터를 구축한다.

### 2-1. Ingress Controller 설치

```Bash
# 클라우드 환경용 NGINX Ingress Controller 매니페스트를 배포함
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
```

### 2-2. 백엔드 테스트용 웹 서버 배포

동작 확인을 위해 Nginx와 Apache 서버를 배포하고 클러스터 내부 서비스(ClusterIP)로 묶는다.

```Bash
# Nginx 디플로이먼트 생성 및 내부 포트 80 개방
kubectl create deployment web-nginx --image=nginx:alpine
kubectl expose deployment web-nginx --port=80

# Apache 디플로이먼트 생성 및 내부 포트 80 개방
kubectl create deployment web-apache --image=httpd:alpine
kubectl expose deployment web-apache --port=80
```
- `--image`: 파드 내부에 실행할 컨테이너의 이미지를 지정하는 옵션.
- `--port`: 서비스가 노출할 클러스터 내부 포트 번호를 지정하는 옵션.

### 2-3. Ingress 라우팅 규칙(Rules) 정의

접속 도메인(`nginx.local`, `apache.local`)에 따라 트래픽을 각각의 서비스로 전달하는 규칙을 생성한다.

```Bash
# Ingress 리소스를 클러스터에 생성함
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: "nginx.local"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-nginx
            port:
              number: 80
  - host: "apache.local"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-apache
            port:
              number: 80
EOF
```

**마찬가지로 `nano ingress.yaml` 생성하여 `kubectl apply -f ingress.yaml` 해도 가능**

### 3-4. 접속 테스트

할당받은 Ingress Controller의 `EXTERNAL-IP`로 HTTP 요청을 보내되, 헤더에 호스트 이름을 지정하여 라우팅을 검증한다.

```Bash
# HTTP 요청 헤더에 nginx.local 도메인을 명시하여 통신함 (Welcome to nginx! 출력 확인)
curl -H "Host: nginx.local" http://192.168.56.200

# HTTP 요청 헤더에 apache.local 도메인을 명시하여 통신함 (It works! 출력 확인)
curl -H "Host: apache.local" http://192.168.56.200
```
- `-H` (`--header`): HTTP 요청 시 서버로 보낼 사용자 정의 헤더 데이터를 추가하는 옵션.

**검증** : 결과로 `Welcome to nginx!`
**검증** : 결과로 `It works!`

---
