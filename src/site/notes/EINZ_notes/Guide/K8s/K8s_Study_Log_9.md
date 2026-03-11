---
{"dg-publish":true,"permalink":"/einz-notes/guide/k8s/k8s-study-log-9/","title":"9: Helm & Ingress"}
---


# Helm & Ingress

기존 하드코딩된 Kubernetes 매니페스트 파일을 Helm 기반의 템플릿으로 마이그레이션하고, Ingress를 도입하여 로컬 도메인 라우팅을 구성한 과정 요약.

## 1. Helm Chart Migration

단일 YAML 파일을 환경별로 재사용 가능한 Helm Chart 구조로 변경한다.

### 1.1 Helm Installation

```Bash
# Helm 설치 script 다운로드
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

# 실행 권한 부여
chmod 700 get_helm.sh

# Script 실행
./get_helm.sh
```

### 1.2 Chart Creation & Template Configuration

기존 작업 디렉토리에서 Helm 차트를 생성하고 불필요한 기본 템플릿을 삭제한다.

```Bash
# Helm chart 생성
helm create my-node-app-chart

# Default template들 삭제
rm -rf my-node-app-chart/templates/*
```

#### `my-node-app-chart/templates/deployment.yaml`

Node.js 앱이 3000번 포트를 사용하므로 `containerPort`를 3000으로 설정한다.

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: node-app
          # Use image repository and tag from values.yaml
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 3000
```

#### `my-node-app-chart/templates/service.yaml`

`targetPort`를 컨테이너 포트와 동일한 3000으로 설정한다.

```YAML
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-service
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: 80
      targetPort: 3000
  selector:
    app: {{ .Release.Name }}
```

#### `my-node-app-chart/values.yaml`

템플릿에 주입될 실제 변수 값들을 정의한다.

```YAML
# Number of pod replicas
replicaCount: 2

# Container image information
image:
  repository: dlrytnwkd/my-node-app
  tag: "1.243"

# Service configuration
service:
  type: ClusterIP
```

## 2. Jenkins Pipeline Integration & Fixes

Helm 구조에 맞게 Jenkins 파이프라인을 수정하고, 발생했던 무한 루프 및 문법 오류를 해결한다.

### 2.1 Infinite Loop Prevention & Pipeline Script Update

Gitea Webhook이 Jenkins 커밋에 의해 다시 트리거되는 무한 루프를 방지하기 위해, 코드를 Clone한 직후 커밋 메시지를 검증하는 로직을 추가한다. 또한 매니페스트 업데이트 대상을 `values.yaml`로 변경한다.

```Groovy
pipeline {
    agent any
    environment {
        DOCKERHUB_ID = '본인도커허브ID'
        IMAGE_TAG = "${DOCKERHUB_ID}/my-node-app:1.${BUILD_NUMBER}"
    }
    stages {
        stage('1. Checkout Code from Gitea') {
            steps {
                git url: "http://172.17.0.1:3000/${GITEA_USERNAME}/k8s-app.git", branch: 'master'
            }
        }
        
        // 무한 루프 방지용 스테이지 추가
        stage('2. Check Trigger Source') {
            steps {
                script {
                    // 마지막 커밋 메시지를 가져와서 확인
                    def commitMsg = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
                    echo "commitMsg : ${commitMsg}"
                    if (commitMsg.contains('Jenkins Auto Build')) {
                        echo "! 젠킨스가 자동 생성한 커밋입니다. 무한 루프 방지를 위해 파이프라인을 종료합니다."
                        currentBuild.result = 'SUCCESS'
                        error("무한 루프 방지용 의도적 종료 (실패 아님)")
                    }
                }
            }
        }
        stage('3. Build Docker Image') {
            steps {
                dir('node-app') {
                    sh "docker build -t ${IMAGE_TAG} ."
                }
            }
        }

        stage('4. Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'DOCKER_PW', usernameVariable: 'DOCKER_ID')]) {
                    sh "echo \$DOCKER_PW | docker login -u \$DOCKER_ID --password-stdin"
                    sh "docker push ${IMAGE_TAG}"
                    sh "docker logout"
                }
            }
        }

        stage('5. Update Manifest & Push to Gitea') {
            steps {
                // Jenkins 금고에서 Gitea 계정 정보를 꺼내온다.
                withCredentials([usernamePassword(credentialsId: 'gitea-creds', passwordVariable: 'GITEA_PW', usernameVariable: 'GITEA_ID')]) {
                    sh """
                    # 1. my-node-app-chart 폴더 안의 values.yaml에서 tag 값을 젠킨스 빌드 번호로 변경
                    sed -i 's/tag: .*/tag: \"1.${BUILD_NUMBER}\"/g' my-node-app-chart/values.yaml
                    
                    # 2. 변경된 사항을 Git으로 커밋
                    git config user.email "jenkins@local-cicd.com"
                    git config user.name "Jenkins CI"
                    git add my-node-app-chart/values.yaml
                    git commit -m 'Update Helm values tag to 1.${BUILD_NUMBER} by Jenkins Auto Build'
                    
                    # 3. Gitea 서버로 Push (주소에 인증 정보 포함)
                    git push http://\$GITEA_ID:\$GITEA_PW@172.17.0.1:3000/\$GITEA_ID/k8s-app.git HEAD:master
                    """
                }
            }
        }
    }
}
```

## 3. ArgoCD Configuration Update

ArgoCD가 기존 단일 YAML 대신 Helm Chart 디렉토리를 바라보도록 수정한다.

- ArgoCD Web UI 접속.

- Application 설정에서 `Path`를 `my-node-app-chart`로 변경.

- (Troubleshooting) `argocd-repo-server` 파드가 재시작 중일 때 발생하는 `connection refused` 에러는 파드가 `1/1 Running` 상태가 될 때까지 대기 후 다시 저장하여 해결.


## 4. Ingress Configuration & Network Routing

K8s 클러스터 내부의 서비스를 외부 도메인(`my-node-app.local`)을 통해 접근할 수 있도록 Ingress를 구성한다.

### 4.1 Ingress Template & Values Update

#### `my-node-app-chart/templates/ingress.yaml`

```YAML
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
spec:
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ .Release.Name }}-service
                port:
                  number: 80
{{- end -}}
```

#### `my-node-app-chart/values.yaml` (추가)

```YAML
# Ingress configuration
ingress:
  enabled: true
  host: my-node-app.local
```

변경 사항을 Gitea에 Push하여 ArgoCD가 Ingress 리소스를 클러스터에 배포하도록 유도한다.

### 4.2 Network Routing Fixes (Windows ↔ Ubuntu ↔ Minikube)

로컬 환경의 중첩 네트워크 문제 및 권한 문제를 우회하기 위한 설정.

#### 4.2.1 Windows `hosts` 파일 수정

Windows 시스템이 `my-node-app.local` 도메인을 로컬 호스트로 인식하도록 설정한다.

- 경로: `C:\Windows\System32\drivers\etc\hosts`

- 추가 내용: `127.0.0.1 my-node-app.local`
 
#### 4.2.2 VirtualBox Port Forwarding 우회 설정

Windows OS가 80번 포트를 점유하여 발생하는 `Connection Refused` 에러를 방지하기 위해 대체 포트(8088)를 사용한다.

- VirtualBox Network Settings -> Port Forwarding

- Host Port: `8088`, Guest Port: `8088`


#### 4.2.3 Minikube Ingress Port Forwarding (권한 이슈 해결)

Ubuntu 터미널에서 Ingress Controller로 트래픽을 넘긴다. 이때 `sudo` 권한 실행 시 root 계정의 kubeconfig를 참조하여 발생하는 인증 에러(`certificate signed by unknown authority`)를 방지하기 위해 사용자 계정의 kubeconfig 경로를 명시한다.

```Bash
# Enable Ingress addon in Minikube
minikube addons enable ingress

# Forward traffic from Ubuntu (8088) to Minikube Ingress (80)
sudo kubectl --kubeconfig=/home/xtra/.kube/config port-forward --address 0.0.0.0 -n ingress-nginx svc/ingress-nginx-controller 8088:80
```

### 4.3 확인

모든 설정 완료 후 Windows 브라우저에서 `http://my-node-app.local:8088`로 접속하여 정상 동작(502 에러 포트 매칭 해결 포함)을 확인한다.

---
