---
{"dg-publish":true,"permalink":"/einz-notes/guide/k8-s/k8-s-study-log-4/","title":"4: ConfigMap, Secret, Ingress"}
---


# 설정 분리와 L7 라우팅

## 1. 설정과 코드의 분리 (ConfigMap & Secret)

애플리케이션 소스 코드에 환경 변수나 비밀번호를 하드코딩하지 않고, K8S 객체를 통해 외부에서 안전하게 주입하는 실무 표준 아키텍처 실습.

### 1.1 소스 코드 수정 및 이미지 재빌드
코드는 유지한 채 환경만 바꾸기 위해 `process.env`를 사용하도록 코드를 수정함.

```bash
# 1. server.js 수정 (환경 변수 읽기)
cat <<EOF > server.js
const express = require('express');
const os = require('os');
const app = express();
const port = 3000;

const greeting = process.env.GREETING_MESSAGE || '기본 인사말입니다.';
const dbPassword = process.env.DB_PASSWORD || '비밀번호 없음';

app.get('/', (req, res) => {
  res.send(\`
    <h1>\${greeting}</h1>
    <p>응답하는 파드 이름: \${os.hostname()}</p>
    <p>주입된 DB 비밀번호: \${dbPassword}</p>
  \`);
});

app.listen(port, () => {
  console.log(\`App running on http://localhost:\${port}\`);
});
EOF

# 2. Minikube 도커 엔진 연동 후 v1.2로 이미지 빌드
eval $(minikube docker-env)
docker build -t my-node-app:1.2 .
````

### 1.2 통합 배포 설계도(yaml) 작성 및 적용

ConfigMap(일반 설정)과 Secret(민감 정보)을 정의하고, Deployment에서 이를 환경 변수(env)로 끌어다 쓰도록 설정함.

```yaml
# my-node-app.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  GREETING_MESSAGE: "Greeting Message injected from ConfigMap!"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  DB_PASSWORD: "super-secret-password-123"
---
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
        env:
        - name: GREETING_MESSAGE
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: GREETING_MESSAGE
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD
---
apiVersion: v1
kind: Service
metadata:
  name: node-app-service
spec:
  type: NodePort
  selector:
    app: node-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
```


```bash
# 클러스터 적용 및 확인
kubectl apply -f my-node-app.yaml
```

---

## 2. Ingress를 활용한 URL 경로 기반 라우팅

하나의 진입점(포트)으로 들어온 트래픽을 URL 경로(`/apple`, `/banana`)에 따라 서로 다른 서비스로 분배하는 L7 라우팅 실습.

### 2.1 Ingress Controller 활성화

K8S 내부에 Nginx 인그레스 컨트롤러 파드를 시스템 네임스페이스(`ingress-nginx`)에 배포함.

```bash
minikube addons enable ingress
```

### 2.2 백엔드 테스트 앱 배포

접속 시 텍스트만 뱉어내는 가벼운 이미지(`hashicorp/http-echo`)를 활용해 Apple과 Banana 앱을 배포함.

```bash
# fruits-app.yaml 작성 후 배포
kubectl apply -f fruits-app.yaml
```

### 2.3 Ingress 라우팅 규칙 작성 및 적용

`main-ingress.yaml`을 통해 컨트롤러가 참고할 길 안내 규칙을 K8S에 전달함.

```yaml
# main-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /apple
        pathType: Prefix
        backend:
          service:
            name: apple-service
            port:
              number: 5678
      - path: /banana
        pathType: Prefix
        backend:
          service:
            name: banana-service
            port:
              number: 5678
```

```bash
# 규칙 적용
kubectl apply -f main-ingress.yaml
```

### 2.4 Ingress controller 포트 포워딩 및 검증

개별 서비스가 아닌, 트래픽을 통제하는 Ingress 컨트롤러 자체를 외부로 노출시켜 라우팅을 검증함.

```bash
# ingress-nginx 네임스페이스의 컨트롤러 서비스를 30001번 포트로 연결
kubectl port-forward --address 0.0.0.0 -n ingress-nginx service/ingress-nginx-controller 30001:80
```

- **검증:** `http://localhost:30001/apple` -> apple 출력
- **검증:** `http://localhost:30001/banana` -> banana 출력

---
