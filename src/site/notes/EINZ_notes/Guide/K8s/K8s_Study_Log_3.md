---
{"dg-publish":true,"permalink":"/einz-notes/guide/k8s/k8s-study-log-3/","title":"3: 스토리지(PV&PVC), 리소스, 커스텀 앱 배포"}
---


# 스토리지(PV&PVC), 리소스, 커스텀 앱 배포

## 1: 데이터 영속성 보장 (Persistent Volume & PVC)

컨테이너(Pod)가 삭제되거나 재생성 되어도 내부 데이터가 날아가지 않도록, K8S 클러스터 노드(Minikube)의 실제 디스크 공간을 연결(Mount)하는 실습.

### 1.1 Minikube 내부에 실제 파일 생성

K8S 외곽의 가상 머신(Minikube) 내부에 직접 들어가서 보존될 데이터를 생성한다.

```bash
# 1. Minikube 노드 내부로 접속
minikube ssh

# 2. 저장소로 사용할 디렉토리 생성
sudo mkdir -p /data/nginx-html

# 3. 영구 보존될 HTML 파일 생성 (Bash 히스토리 에러 방지를 위해 반드시 작은따옴표 '' 사용)
echo '<h1>Data survived!</h1>' | sudo tee /data/nginx-html/index.html

# 4. 파일 생성 확인 후 빠져나오기
ls -la /data/nginx-html
exit
```

### 1.2 PV 및 PVC가 포함된 통합 설계도 작성

`nginx-storage.yaml` 파일을 생성하여 PV, PVC, Deployment, Service를 모두 정의한다.

**주의:** K8S의 동적 프로비저닝(자동으로 빈 폴더를 생성하는 기능)을 막기 위해 `storageClassName: ""` 및 `volumeName`을 명시적으로 지정해야 한다.

```yaml
# nginx-storage.yaml
# 1. PV (실제 저장소 정의)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/nginx-html"
---
# 2. PVC (저장소 사용 요청)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  storageClassName: ""      # K8S의 자동 빈 폴더 생성 방지
  volumeName: nginx-pv      # 직접 만든 PV(nginx-pv)를 정확히 지목
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
# 3. Deployment & 4. Service (이전 Nginx 실습과 동일하며, volumes 설정만 추가)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-storage-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-storage
  template:
    metadata:
      labels:
        app: nginx-storage
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: html-volume
      volumes:
      - name: html-volume
        persistentVolumeClaim:
          claimName: nginx-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-storage-service
spec:
  type: NodePort
  selector:
    app: nginx-storage
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

### 1.3 배포 및 데이터 보존(Pod 파괴) 테스트

```bash
# 배포 적용
kubectl apply -f nginx-storage.yaml

# 파드 파괴 및 자동 재생성 대기
kubectl delete pod [파드_이름]
kubectl get pods -w

# 포트 포워딩 후 윈도우 브라우저(localhost:8888)에서 "Data survived!" 출력 확인
kubectl port-forward --address 0.0.0.0 service/nginx-storage-service 8888:80
```

---

## 2: 파드 리소스 제한 (Resource Quota)

특정 파드가 호스트(가상 머신)의 CPU나 메모리를 과도하게 점유하여 시스템이 다운되는 것을 방지하기 위한 자원 통제 설정.

### 2.1 설계도에 리소스 제한 추가

`nginx-storage.yaml`의 `containers` 하위에 `resources` 블록을 추가한다.

```yaml
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

### 2.2 적용 및 검증

```bash
# 변경된 YAML 적용
kubectl apply -f nginx-storage.yaml

# 정상 적용 확인 (출력 내용 중 Limits와 Requests 항목 확인)
kubectl describe pod [파드_이름]
```

---

## 3: 커스텀 앱(Node.js) 이미지 빌드 및 K8S 배포

공식 이미지가 아닌, 직접 작성한 코드를 Docker 컨테이너로 감싸서 배포하는 CI/CD 기초 실습.

### 3.1 소스 코드 및 Dockerfile 작성

```bash
mkdir -p ~/k8s-study/node-app && cd ~/k8s-study/node-app

# server.js 작성
cat <<EOF > server.js
const express = require('express');
const os = require('os');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.send(\`<h1>Hello, this is custom K8S App!</h1><p>Answering pod name: \${os.hostname()}</p>\`);
});

app.listen(port, () => {
  console.log(\`App running on http://localhost:\${port}\`);
});
EOF

# package.json 작성
cat <<EOF > package.json
{
  "name": "my-node-app",
  "version": "1.0.0",
  "main": "server.js",
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF

# Dockerfile 작성
cat <<EOF > Dockerfile
FROM node:18-alpine
WORKDIR /usr/src/app
COPY package.json ./
RUN npm install
COPY server.js ./
EXPOSE 3000
CMD [ "node", "server.js" ]
EOF
```

**참고** : `cat <<EOF >` 대신 `nano`로 생성해도 됨.

### 3.2 Minikube 도커 연동 및 빌드

가상 머신 외부가 아닌 Minikube 내부의 도커 엔진에 이미지를 저장해야 K8S가 이미지를 찾을 수 있다.

```bash
# 현재 터미널을 Minikube 도커 엔진과 연동
eval $(minikube docker-env)

# 커스텀 이미지 빌드
docker build -t my-node-app:1.0 .
```

### 3.3 배포 및 로드 밸런싱 테스트

`my-node-app.yaml` 파일을 작성하고 로컬 이미지를 바라보도록 `imagePullPolicy: Never`를 명시해야 한다.

```yaml
# my-node-app.yaml
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
        image: my-node-app:1.0
        imagePullPolicy: Never
        ports:
        - containerPort: 3000
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
# K8S 클러스터에 적용
kubectl apply -f my-node-app.yaml

# K8S 내부 서비스 URL을 통한 로드 밸런싱 검증
export SVC_URL=$(minikube service node-app-service --url)
for i in {1..5}; do curl $SVC_URL; echo ""; sleep 0.5; done
# (출력되는 파드 이름이 2개의 파드 사이에서 번갈아 가며 바뀌는 것을 확인)
```

---
