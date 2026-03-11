---
{"dg-publish":true,"permalink":"/einz-notes/guide/k8s/k8-s-study-log-12/","title":"12: HTTPS 인증서 자동화 구축 (cert-manager)"}
---


# HTTPS 인증서 자동화 구축 (cert-manager)

Kubernetes 환경에서 HTTPS 통신을 위한 TLS 인증서 발급 및 갱신을 자동화하는 과정 요약. 로컬 가상 환경 특성상 외부 공인 인증 기관을 거칠 수 없으므로 자체 서명(Self-Signed) 방식을 사용했다.

## 1. cert-manager 설치

Helm을 사용하여 Jetstack의 cert-manager를 클러스터에 배포한다. 가상 머신 환경의 리소스 제한으로 인한 타임아웃 에러를 방지하기 위해 대기 시간을 연장한다.

```Bash
# Jetstack 저장소 추가 및 업데이트
helm repo add jetstack https://charts.jetstack.io
helm repo update

# cert-manager 설치
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true \
  --timeout 30m
```
- `--set installCRDs=true`: cert-manager 작동에 필수적인 K8S 사용자 정의 리소스(CRD)를 함께 설치하는 옵션.
- `--timeout 30m`: K8S API 서버 부하로 인해 설치가 지연될 때 발생하는 시간 초과 에러를 막기 위해, 대기 시간을 기본 5분에서 30분으로 늘리는 옵션이다.


## 2. ClusterIssuer (인증서 발급소) 생성

클러스터 전역에서 동작하며 자체 서명된 인증서를 찍어낼 수 있는 발급소 리소스를 생성한다.

```Bash
# yaml 문서를 작성하여 K8S 클러스터에 즉시 적용
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: local-selfsigned-issuer
spec:
  selfSigned: {}
EOF
```

## 3. Helm 차트 Ingress 연동 설정

Ingress 리소스가 앞서 만든 인증서 발급소에 TLS 인증서를 요청하도록 Helm 차트를 수정한다.

**values.yaml 수정**

Ingress를 활성화하고, 인증서 발급소를 지정하는 어노테이션(annotations) 및 TLS 시크릿 정보를 추가한다.

```YAML
ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "local-selfsigned-issuer"
  hosts:
    - host: my-node-app.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls:
    - secretName: my-node-app-tls
      hosts:
        - my-node-app.local
```

**templates/ingress.yaml 수정**

Helm 배포 시 `values.yaml`에 추가한 `annotations`와 `tls` 블록을 K8S 매니페스트에 동적으로 렌더링할 수 있도록 템플릿 구조를 변경한다.

```YAML
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
  annotations:
    {{- toYaml .Values.ingress.annotations | nindent 4 }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  tls:
    {{- toYaml .Values.ingress.tls | nindent 4 }}
# 하위 rules 설정은 기존과 동일하게 유지
```

수정이 완료되면 Git에 푸시한다. ArgoCD가 변경 사항을 감지하고 배포 파이프라인(Argo Rollouts)을 통해 K8S에 적용한다.

## 4. 발급 상태 및 HTTPS 통신 확인

인증서가 정상 발급되었는지 확인하고, 암호화된 통신(TLS 핸드쉐이크)을 테스트한다.

```Bash
# 1. Ingress의 PORTS 항목에 443 포트가 추가되었는지 확인
kubectl get ingress

# 2. cert-manager가 생성한 인증서의 READY 상태가 True인지 확인
kubectl get certificate
```

인증서 발급이 완료되면 HTTPS 접속 테스트를 진행한다.

```Bash
# HTTPS 통신 테스트 및 인증서 발급자 상세 정보 확인
curl -kv https://192.168.49.2 -H "Host: my-node-app.local"
```
- `-k` (또는 `--insecure`): 브라우저나 curl이 해당 인증서를 신뢰할 수 없는 자체 서명 인증서로 판단하더라도, 보안 경고를 무시하고 강제로 통신을 진행하라는 옵션이다.
- `-v` (또는 `--verbose`): 클라이언트와 서버 간의 TLS 핸드쉐이크 과정, 인증서 상세 정보(발급자 등), 요청 및 응답 헤더 등 통신의 전체 로그를 화면에 자세히 출력하라는 옵션이다.

---
