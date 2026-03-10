---
{"dg-publish":true,"permalink":"/einz-notes/guide/k8-s/k8-s-study-log-8/","title":"8: Gitea, Jenkins, ArgoCD를 통한 CICD 구축"}
---


# 로컬 GitOps CI/CD 파이프라인 구축 총정리 (Minikube + Gitea + Jenkins + ArgoCD)

## 1. 코드 저장소 및 빌드 서버 구성 (Gitea & Jenkins)

### 1.1 Gitea (사내 Git 저장소) 컨테이너 실행

- 외부 GitHub 대신 로컬 환경에서 코드를 관리하기 위해 Gitea를 도커 컨테이너로 실행.

```bash
docker run -d --name gitea -p 3000:3000 -p 222:22 gitea/gitea:latest

# Jenkins가 도커 이미지를 빌드하려면 우분투 호스트의 도커 엔진에 접근할 권한이 필요하다. 또한, 기존에 ArgoCD 포트 포워딩으로 `8080` 포트를 사용 중이므로 포트 충돌을 피하기 위해 Jenkins는 `9090` 포트로 개방한다.
docker run -d --name jenkins -p 9090:8080 -p 50000:50000 -u root -v /var/run/docker.sock:/var/run/docker.sock -v jenkins-data:/var/jenkins_home jenkins/jenkins:lts
```

- 브라우저(`localhost:3000`)로 접속해 초기 설정을 마치고 `k8s-app` 이라는 저장소를 생성하여 K8S 배포용 YAML 파일(`my-node-app.yaml` 등)을 푸시.

### 1.2 Gitea (로컬 Git 저장소) 컨테이너 실행, Jenkins (CI 빌드 서버) 실행 및 재시작 설정

- 컨테이너 이미지 빌드와 파이프라인 구동을 위해 Jenkins를 실행. (포트 9090 사용)
- 가상 머신(VirtualBox) 재부팅 시 컨테이너 이름 충돌(`Conflict`) 에러를 방지하고 자동 실행되도록 재시작 정책을 적용.

```bash
# 기존에 멈춘 컨테이너를 깨울 때
docker start gitea
docker start jenkins

# 재부팅 시 자동 실행되도록 정책 업데이트
docker update --restart unless-stopped gitea
docker update --restart unless-stopped jenkins
```

**VirtualBox의 Port forwarding 설정 추가 필요** : 호스트 3000 => 게스트 3000, 호스트 9090 => 게스트 9090

---

## 2. Gitea (로컬 Git 서버) 구축

### 2.1 Gitea (로컬 Git 서버) 초기 설정

Gitea 웹 화면(`http://localhost:3000`)으로 이동하여 Gitea 초기 설정 진행.

1. **데이터베이스 설정 (Database Settings):** 가장 상단의 Database Type을 **"SQLite3"**로 선택. (로컬 테스트 환경이므로 별도의 외부 DB 연동 없이 가장 가볍게 세팅하기 위함)

2. **일반 설정 (General Settings):** * Site Title은 "My Local Git" 등 원하는 이름으로 지정.
	- Server Domain, Gitea Base URL 등 나머지 네트워크 설정은 일단 기본값 그대로.

3. **관리자 계정 생성 (Optional Settings):** 화면 맨 아래쪽의 **"Administrator Account Settings"** 탭을 클릭하여 펼친 후, 사용할 관리자 ID(Username), 암호(Password), 이메일을 입력. (이 계정이 향후 로컬 깃허브의 마스터 계정)

4. **설치 진행:** 화면 맨 아래의 **"Install Gitea"** 버튼을 클릭.

### 2.2 Gitea 저장소 생성 및 로컬 코드 Push

#### 2.2.1 원격 저장소(Repository) 생성.

1. Gitea 웹 화면 우측 상단의 **`+`** 아이콘을 누르고 **`새 저장소 (New Repository)`** 를 클릭.
2. **저장소 이름(Repository Name):** `k8s-app` 이라고 입력.
3. **가시성(Visibility):** 실습 편의(Jenkins와 ArgoCD의 접근성)를 위해 **`Public(공개)`** 상태 그대로.
4. 하단의 **`저장소 생성 (Create Repository)`** 버튼을 클릭.
5. 화면에 나타난 HTTP 저장소 주소를 복사하거나 메모 및 따로 저장해두기. _(예: `http://localhost:3000/본인계정명/k8s-app.git`)_

#### 2.2.2 터미널에서 Git 초기화 및 커밋

- 지금까지 작업했던 `~/k8s-study` 폴더를 Git 로컬 저장소로 만들고 파일들을 기록해야함. 우분투 터미널을 열고 아래 명령어를 순서대로 실행.

```bash
# 작업 폴더로 이동
cd ~/k8s-study

# Git 초기 환경 설정 (최초 1회만 필요, 본인 정보로 임의 입력해도 무방함)
git config --global user.name "admin"
git config --global user.email "admin@example.com"

# 해당 폴더를 Git 저장소로 초기화
git init

# 폴더 내 모든 파일(YAML, Node.js 소스코드 등)을 스테이징 영역에 추가
git add .

# 첫 번째 버전으로 확정(Commit)
git commit -m "Initial commit for CI/CD pipeline"
```

#### 2.2.3 Gitea 서버로 Push

명령어의 `계정명` 부분을 앞서 Gitea에서 만든 본인의 관리자 ID로 변경하여 실행.

```bash
# Gitea 원격 저장소 주소 연결
git remote add origin http://localhost:3000/계정명/k8s-app.git

# 코드를 Gitea 서버로 밀어넣기 (Push)
git push -u origin master

# 명령어 실행 시 터미널에서 Gitea의 Username과 Password를 물어볼 수 있음. 가입했던 정보를 입력하면 됨.
```

---

## 3. Jenkins 파이프라인(CI) 구축

### 3.1 Jenkins 초기 설정

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

- 비밀번호가 출력되면 해당 값을 Jenkins 웹 화면에 입력하여 잠금을 해제.

1. **플러그인 설치:** "Customize Jenkins" 화면에서 왼쪽의 **"Install suggested plugins"** 버튼을 클릭. (기본적으로 필요한 Git, 파이프라인 관련 플러그인들이 자동으로 설치되며 수 분 정도 소요됨)

2. **관리자 계정 생성:** 플러그인 설치가 완료되면 "Create First Admin User" 화면이 나타남. 향후 접속에 사용할 계정명(Username), 암호(Password), 이름, 이메일을 입력하고 **"Save and Continue"** 를 클릭.

3. **인스턴스 설정:** "Instance Configuration" 화면이 나타나면 Jenkins URL은 기본값(예: `http://localhost:9090/`) 그대로 두고 **"Save and Finish"** 를 클릭.

4. **시작:** "Start using Jenkins" 버튼을 누르면 Jenkins 메인 대시보드 화면으로 진입함.

### 3.2 Jenkins 파이프라인(빌드 자동화) 생성

Jenkins에서 Gitea의 코드를 가져와서 자동으로 빌드하는 '작업 지시서(Pipeline)'를 작성.

#### 3.2.1 Jenkins에서 새로운 Item(작업) 생성

1. Jenkins 웹 대시보드(`http://localhost:9090`) 좌측 상단의 **"New Item (새로운 Item)"** 을 클릭.

2. 작업 이름(Enter an item name)에 `k8s-cicd-pipeline`이라고 입력.
  
3. 목록에서 **"Pipeline (파이프라인)"** 을 선택하고, 맨 아래의 **"OK"** 버튼을 클릭.
  

#### 3.2.2 파이프라인 스크립트 작성

설정 화면이 열리면 맨 아래의 **"Pipeline"** 섹션으로 스크롤 내린다. `Definition`이 `Pipeline script`로 되어 있는 빈 텍스트 박스에 아래의 코드를 복사하여 붙여넣기.

_(주의: 스크립트 내의 `본인계정명` 부분을 앞서 Gitea에서 생성한 본인의 ID로 반드시 변경 필요.)_

```Groovy
pipeline {
    agent any

    environment {
        DOCKERHUB_ID = '본인계정명'
        GITEA_USERNAME = '본인계정명'
	    // 고정 버전이 아닌 빌드 번호를 태그로 사용 (예: 1.4, 1.5...)
	    IMAGE_TAG = "${DOCKERHUB_ID}/my-node-app:1.${BUILD_NUMBER}"
    }

    stages {
        // 무한 루프 방지용 스테이지 추가
        stage('0. Check Trigger Source') {
            steps {
                script {
                    // 마지막 커밋 메시지를 가져와서 확인
                    def commitMsg = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
                    
                    if (commitMsg.contains('Jenkins Auto Build')) {
                        echo "! 젠킨스가 자동 생성한 커밋입니다. 무한 루프 방지를 위해 파이프라인을 종료합니다."
                        currentBuild.result = 'SUCCESS'
                        error("무한 루프 방지용 의도적 종료 (실패 아님)")
                    }
                }
            }
        }

        stage('1. Checkout Code from Gitea') {
            steps {
                git url: "http://172.17.0.1:3000/${GITEA_USERNAME}/k8s-app.git", branch: 'master'
            }
        }
        
        stage('2. Build Docker Image') {
            steps {
                dir('node-app') {
                    sh "docker build -t ${IMAGE_TAG} ."
                }
            }
        }

        stage('3. Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'DOCKER_PW', usernameVariable: 'DOCKER_ID')]) {
                    sh "echo \$DOCKER_PW | docker login -u \$DOCKER_ID --password-stdin"
                    sh "docker push ${IMAGE_TAG}"
                    sh "docker logout"
                }
            }
        }

        stage('4. Update Manifest & Push to Gitea') {
            steps {
                // Jenkins 금고에서 Gitea 계정 정보를 꺼내온다.
                withCredentials([usernamePassword(credentialsId: 'gitea-creds', passwordVariable: 'GITEA_PW', usernameVariable: 'GITEA_ID')]) {
                    sh """
                    # 1. my-node-app.yaml 파일 안의 image 주소를 새 버전으로 찾아 바꾸기 (sed 명령어)
                    sed -i "s|image: .*my-node-app.*|image: ${IMAGE_TAG}|g" my-node-app.yaml
                    
                    # 2. 변경된 사항을 Git으로 커밋
                    git config user.email "jenkins@local-cicd.com"
                    git config user.name "Jenkins CI"
                    git add my-node-app.yaml
                    git commit -m "Update image tag to ${IMAGE_TAG} by Jenkins Auto Build"
                    
                    # 3. Gitea 서버로 Push (주소에 인증 정보 포함)
                    git push http://\$GITEA_ID:\$GITEA_PW@172.17.0.1:3000/\$GITEA_ID/k8s-app.git HEAD:master
                    """
                }
            }
        }
    }
}
```

작성을 완료했다면 화면 맨 아래의 **"Save (저장)"** 버튼을 클릭.

#### 3.2.3 수동 빌드(Build) 테스트 및 확인

방금 만든 파이프라인이 정상적으로 Gitea에 접속해서 코드를 가져오고, 도커 이미지를 구워내는지 테스트.

1. `k8s-cicd-pipeline` 화면 좌측 메뉴에서 **"Build Now (지금 빌드)"** 를 클릭.
2. 하단의 'Build History'에 `#1` 이라는 번호가 깜빡이며 작업이 시작됨.
3. 해당 번호(`#1`)를 클릭하고 **"Console Output (콘솔 출력)"** 메뉴로 진입.

#### 문제 해결: DOCKERHUB_ID

현재 실습에서 이미지 저장소(Repository)로 Docker Hub를 사용할 예정이기 때문에 Docker Hub 계정 및 토큰이 필요함.

#### 문제 해결: Jenkins 컨테이너에 Docker CLI 설치

`docker: not found` 에러가 발생할 경우 터미널을 열고 아래 명령어 두 줄을 순서대로 실행 (실행 중인 Jenkins 컨테이너 내부로 관리자 권한(`-u root`)을 가지고 들어가 패키지를 설치하는 명령)

```Bash
# 1. 패키지 목록 업데이트
docker exec -u root jenkins apt-get update

# 2. 도커 클라이언트 설치
docker exec -u root jenkins apt-get install -y docker.io

# 혹은 순수 Docker CLI 바이너리만 다운로드 해도 됨
docker exec -it -u root jenkins bash

# 1. 공식 정적 바이너리 다운로드
curl -fsSLO https://download.docker.com/linux/static/stable/x86_64/docker-24.0.9.tgz

# 2. 압축 해제 및 CLI 실행 파일만 /usr/bin으로 이동
tar xzvf docker-24.0.9.tgz
mv docker/docker /usr/bin/
rm -rf docker docker-24.0.9.tgz

# 3. 설치 확인
docker --version
```

소켓 권한 문제 해결

- Jenkins는 내부적으로 `jenkins`라는 일반 사용자로 구동되는데, 호스트에서 끌고 온 `docker.sock`은 `root` 소유라서 권한 거부(Permission Denied) 에러가 날 수 있다. 아래 명령어로 소켓 접근 권한을 열어주고 빠져나온다.

```bash
chmod 666 /var/run/docker.sock
exit
```

### 3.3 Jenkins Credential (자격 증명) 등록

- Jenkins가 Gitea 소스를 가져오고 Docker Hub에 이미지를 푸시할 수 있도록 보안 자격 증명을 등록해야함.

- 파이프라인 스크립트에 비밀번호를 평문으로 적는 것은 보안상 절대 금물이므로, Jenkins의 전용 금고에 자격 증명을 등록.

1. Jenkins 메인 대시보드 좌측 메뉴에서 **Manage Jenkins (Jenkins 관리)** 를 클릭.

2. Security 섹션에서 **Credentials (자격 증명)** 를 클릭.

3. 표 형태의 목록에서 `(global)` 링크를 클릭한 뒤, 우측 상단의 **+ Add credentials (자격 증명 추가)** 버튼을 누른다.

4. 아래와 같이 정보를 입력한다.
	- **Kind:** `Username with password` (기본값 유지)
    - **Scope:** `Global` (기본값 유지)
    - **Username:** 본인의 Docker Hub ID (이메일 주소가 아닌 ID)
    - **Password:** 본인의 Docker Hub 비밀번호 (또는 발급받은 Access Token)
    - **ID:** `dockerhub-creds` (이름을 정확히 이렇게 입력해야 나중에 파이프라인 코드에서 불러올 수 있다)
    - **Description:** Docker Hub Credentials (선택 사항)

5. 하단의 **Create** 버튼을 클릭하여 저장한다.

### 3.4 Jenkins에 Gitea Credential 등록

1. Jenkins 대시보드 좌측 메뉴에서 **Manage Jenkins (Jenkins 관리)** -> **Credentials (자격 증명)** 이동.

2. `(global)` 클릭 후 **+ Add credentials** 클릭.

3. 정보 입력:
    - **Kind:** `Username with password`
    - **Username:** 본인의 Gitea ID (예: 본인계정명)
    - **Password:** 본인의 Gitea 비밀번호
    - **ID:** `gitea-creds` (파이프라인에서 이 이름표로 꺼내 쓸 예정)
4. **Create** 버튼을 눌러 저장.

---

## 4. ArgoCD(CD) 설치 및 포트 포워딩 자동화

### 4.1 ArgoCD 설치 및 접속

- Minikube 클러스터 내에 `argocd` 네임스페이스를 생성하고 설치 매니페스트를 적용.

```bash
# 1. 네임스페이스 생성
kubectl create namespace argocd

# 2. 공식 배포 설계도(YAML) 적용
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. 파드 정상 구동 대기
kubectl get pods -n argocd -w

# 초기 관리자(admin) 비밀번호 추출 및 Base64 디코딩
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
````

### 4.2 포트 포워딩 자동화 (Systemd 서비스 등록)

- 매번 VM을 켤 때마다 포트 포워딩을 수동으로 여는 불편함을 없애기 위해 리눅스 백그라운드 서비스로 등록.

```Bash
sudo nano /etc/systemd/system/argocd-pf.service
```

```Ini, TOML
[Unit]
Description=ArgoCD Port Forward Auto Start
After=network.target

[Service]
Type=simple
User=본인계정명
Environment=KUBECONFIG=/home/본인계정명/.kube/config
ExecStart=/usr/local/bin/kubectl port-forward --address 0.0.0.0 svc/argocd-server -n argocd 8080:443
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```Bash
# 서비스 적용 및 실행
sudo systemctl daemon-reload
sudo systemctl enable argocd-pf
sudo systemctl start argocd-pf
sudo systemctl status argocd-pf
```

### 4.3 ArgoCD에 Application (my-node-app) 등록

- ArgoCD UI에 접속해 새 앱을 생성.

1. GENERAL (일반 설정)
- **Application Name:** `my-node-app`
- **Project Name:** `default`  
- **Sync Policy:** `Automatic` (체크)
    - _하위 옵션인 `Prune Resources`와 `Self Heal`도 체크

1. SOURCE (Git 저장소 연결)
- **Repository URL:** `http://host.minikube.internal:3000/본인계정명/k8s-app.git`
    - _(만약 이 주소로 연결 에러가 나면, 터미널에서 `ip a`를 쳐서 나오는 실제 IP, 예를 들어 `http://192.168.x.x:3000/...` 형식으로 대체)_
- **Revision:** `master`
- **Path:** `.` (`my-node-app.yaml`이 존재하는 위치)

1. DESTINATION (배포할 K8S 클러스터 위치)
- **Cluster URL:** `https://kubernetes.default.svc` (내부 클러스터 기본값)
- **Namespace:** `default` (기본 네임스페이스)

---

## 5. 오류 대처

1. App Health `Degraded` 에러 (파드 배포 실패)
- **증상:** `ErrImageNeverPull` 또는 `ImagePullBackOff` 발생
- **원인:** 로컬 테스트용으로 설정해 둔 `imagePullPolicy: Never` 때문에 외부 Docker Hub에서 이미지를 받아오지 않음. 또는 Docker Hub 저장소가 Private임.
- **해결:** Gitea에서 `my-node-app.yaml` 파일을 열고 `imagePullPolicy: Always`로 변경(또는 삭제). Docker Hub 저장소를 Public으로 전환.

2. App Health 무한 `Progressing` 에러 (빙글빙글 도는 현상)
- **증상:** `nginx-service`가 며칠을 기다려도 완료되지 않음.
- **원인:** 서비스 타입이 `LoadBalancer`로 되어 있으나, Minikube 환경에는 클라우드 외부 IP를 발급해 줄 장비가 없어 영원히 대기(`Pending`)함.
- **해결:** Gitea에서 해당 YAML 파일을 열어 `type: ClusterIP` 또는 `type: NodePort`로 수정.

3. K8S 문법 에러 (`Forbidden: may not be used when type is 'ClusterIP'`)
- **원인:** 타입을 `ClusterIP`로 바꿨음에도 외부 개방용 설정인 `nodePort: 3xxxx` 줄이 그대로 남아 있어 K8S 문법 검사에서 오류 발생.
- **해결:** YAML 파일의 `ports` 하위에 있던 `nodePort` 줄을 완전히 삭제 후 Commit.  

4. argocd-repo-server 뻗을 경우
- **원인:** VirtualBox에 할당된 리소스 이상으로 자원 사용할 경우 argocd-repo-server pod가 가끔 다운되는 경우가 발생.
- **해결:** `kubectl delete pod -l app.kubernetes.io/name=argocd-repo-server -n argocd` 입력 후 `kubectl get pods -n argocd -w` 로 재기동 확인

---

## 6. Webhook 추가

개발자가 코드를 푸시하기만 하면 전체 파이프라인이 자동으로 굴러가도록 하려면 웹훅(Webhook)을 설정해 주어야 함.

### 6.1 Jenkins API 토큰 발급

1. Jenkins 우측 상단의 본인 **계정명(예: admin 또는 본인 ID)**을 클릭.
2. 좌측 메뉴에서 **Security**를 클릭.  
3. 중간쯤에 있는 **API Token** 섹션에서 **Add new Token** 버튼 클릭.
4. Name에 `gitea-token`이라고 적고 **Generate**를 클릭.
5. **화면에 나타난 토큰을 복사해서 메모 및 저장한 후 Save 버튼을 클릭.** (이 창을 벗어나면 다시 확인 불가)

### 6.2 파이프라인 빌드 Trigger 설정

1. 다시 `k8s-cicd-pipeline` -> **Configure** 진입. 
2. 중간에 있는 **Triggers** 섹션까지 스크롤 다운.  
3. **Trigger builds remotely (e.g., from scripts) (빌드를 원격으로 유발)** 체크박스에 체크.
4. 아래에 생기는 **Authentication Token** 칸에 `my-secret` 입력.
5. **Save**를 눌러 저장.

### 6.3 Gitea에 웹훅(Webhook) 추가

1. Gitea(`http://localhost:3000`)에 접속해서 본인의 `k8s-app` 저장소로 진입.
2. 우측 상단의 **Settings (설정)** -> 좌측 메뉴의 **Webhooks (웹훅)**을 클릭.
3. **Add Webhook** 파란색 버튼을 누르고 **Gitea**를 선택.
4. 아래 양식에 맞춰서 정확하게 입력.
    - **Target URL (대상 URL):** `http://본인젠킨스ID:메모저장해둔APITOKEN@172.17.0.1:9090/job/k8s-cicd-pipeline/build?token=my-secret` _(예시: `http://admin:11a4b5c...f78d@172.17.0.1:9090/job/k8s-cicd-pipeline/build?token=my-secret`)_
    - **HTTP Method:** `POST`
    - **POST Content Type:** `application/json`
    - **Trigger On:** `Push Events` (기본값)
5. 맨 아래 **Add Webhook** 버튼을 눌러 저장.
6. 추가된 웹훅 리스트를 클릭하고 맨 아래로 내려가서 **Test Push Event** 버튼을 눌러서 테스트.

**내부망 웹훅 차단 에러 해결 (ALLOWED_HOST_LIST)**

- **증상:** Gitea에서 Test Delivery 시 `webhook can only call allowed HTTP servers` 에러 발생.
- **원인:** Gitea의 보안 정책상 로컬망(172.17.0.x 등)으로 웹훅을 쏘는 것을 해킹으로 간주해 차단함.
- **해결:** Gitea 설정 파일(`app.ini`)에 예외 설정을 추가하고 재시작.

```Bash
# 1. 설정 파일에 [webhook] 섹션 추가
docker exec -u root gitea sh -c "echo '' >> /data/gitea/conf/app.ini"
docker exec -u root gitea sh -c "echo '[webhook]' >> /data/gitea/conf/app.ini"

# 2. 모든 IP(*)로의 웹훅 전송을 허용하도록 설정 추가
docker exec -u root gitea sh -c "echo 'ALLOWED_HOST_LIST = *' >> /data/gitea/conf/app.ini"

# 3. 변경 사항 적용을 위해 컨테이너 재시작
docker restart gitea
```

---

### 완성된 최종 플로우

이제 개발자가 로컬 PC에서 코드를 수정하고 **`git push`** 만 하면 -> **Gitea** 가 웹훅으로 **Jenkins** 를 찌르고 -> Jenkins가 이미지를 빌드해 **Docker Hub** 에 올린 뒤 YAML 버전을 수정해 다시 Gitea에 푸시하고 -> **ArgoCD** 가 이를 감지하여 K8S 클러스터 파드를 새 버전으로 자동 교체(Rolling Update)