---
{"dg-publish":true,"permalink":"/einz-notes/guide/k8-s/k8-s-study-log-13/","title":"13: Terraform"}
---


# IaC(Infrastructure as Code) 및 Terraform

## 1. Terraform의 개념과 도입 의의

Terraform은 해시코프(HashiCorp)에서 개발한 오픈소스 IaC 도구이다. 단순한 생성 자동화 스크립트를 넘어, K8S 및 다양한 클라우드 리소스를 코드로 정의하고 제어하는 인프라 번역기 역할을 수행한다.

### 1.1. 명령형(Imperative)에서 선언형(Declarative)으로의 전환

- **명령형(kubectl 방식):** "네임스페이스를 생성하라"와 같이 구체적인 행동을 지시한다. 동일한 명령을 반복 실행하면 이미 리소스가 존재한다는 에러를 반환하며 작업이 중단된다.
```bash
# 만약 terraform-demo namespace가 이미 존재한다면 Error 발생 후 Stop
kubectl create namespace terraform-demo
```

- **선언형(Terraform 방식):** "해당 네임스페이스가 1개 존재해야 한다"라는 인프라의 최종 목표 상태(설계도)를 선언한다. 코드를 재실행해도 현재 클러스터 상태와 설계도를 비교하여, 차이가 없을 경우 아무 작업도 하지 않고(No changes) 우아하게 종료된다.
```bash
# tf 파일을 읽고 선언되어 있는 설계안과 자동으로 일치시켜줌
terraform apply
```


### 1.2. 상태(State) 관리를 통한 인프라 추적

Terraform은 코드 적용(`apply`)을 완료하면 작업 디렉토리에 `terraform.tfstate`라는 상태 파일을 생성한다.

- 이 파일에는 Terraform이 생성한 모든 리소스의 상세 정보와 고유 ID가 기록된다.

- 인프라 삭제(`destroy`) 시 사용자가 특정 리소스 이름을 일일이 지정할 필요가 없다. Terraform이 상태 파일을 읽어 자신이 생성한 리소스만 정확히 역추적하여 안전하게 삭제한다.


### 1.3. 플러그인(Provider) 아키텍처

Terraform 코어 엔진 자체는 인프라를 직접 제어하지 않는다. `kubernetes`, `aws`, `gcp` 등의 프로바이더(Provider) 플러그인을 다운로드하여, 사용자가 작성한 코드를 각 플랫폼이 이해할 수 있는 API 호출로 자동 변환해 준다.

### 1.4. 인프라의 완벽한 복제 및 협업 (IaC)

인프라 구축 과정이 명령어 히스토리가 아닌 텍스트 파일(`.tf`)로 영구 보존된다. 장애로 인해 클러스터가 파괴되어도 명령어 한 줄로 동일한 환경을 즉시 복구할 수 있다. 애플리케이션 소스 코드처럼 Git을 통해 인프라 변경 이력을 버전 관리하고 팀원과 협업할 수 있다.

## 2. Terraform 설치

우분투 환경에 해시코프 공식 저장소를 추가하고 Terraform 패키지를 설치한다.

```Bash
# 필수 패키지 설치
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl

# 해시코프 GPG 키 다운로드 및 추가
# -f: HTTP 오류 발생 시 조용히 실패 처리함
# -s: 진행 상태 표시줄을 숨김
# -S: -s 옵션과 함께 사용 시, 실패할 경우 에러 메시지를 출력함
# -L: 서버가 리다이렉트 응답을 보내면 해당 위치로 재요청함
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -

# 해시코프 공식 apt 저장소 추가
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"

# 테라폼 설치 및 버전 확인
sudo apt-get update && sudo apt-get install terraform
terraform -version
```

## 3. 쿠버네티스 리소스 배포 실습

Terraform 코드를 작성하여 K8S 네임스페이스, Nginx 파드(Deployment), 통신용 서비스를 일괄 생성하고 의존성을 관리한다.

### 3.1. 테라폼 구성 파일 작성

작업 디렉토리를 생성하고 `main.tf` 파일을 작성한다.

```Terraform
# main.tf
terraform {
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

# K8S 인증 정보 경로 지정
provider "kubernetes" {
  config_path = "~/.kube/config"
}

# 네임스페이스 생성
resource "kubernetes_namespace" "tf_demo" {
  metadata {
    name = "terraform-demo"
  }
}

# Nginx 웹 서버 배포 (파드 2개)
resource "kubernetes_deployment" "nginx" {
  metadata {
    name      = "nginx-deployment"
    # 위에서 생성한 네임스페이스 이름을 동적으로 참조하여 생성 순서(의존성)를 보장함
    namespace = kubernetes_namespace.tf_demo.metadata[0].name
  }
  spec {
    replicas = 2
    selector {
      match_labels = {
        app = "nginx"
      }
    }
    template {
      metadata {
        labels = {
          app = "nginx"
        }
      }
      spec {
        container {
          image = "nginx:1.25"
          name  = "nginx"
          port {
            container_port = 80
          }
        }
      }
    }
  }
}

# Nginx 접근용 서비스 개방
resource "kubernetes_service" "nginx" {
  metadata {
    name      = "nginx-service"
    namespace = kubernetes_namespace.tf_demo.metadata[0].name
  }
  spec {
    selector = {
      app = "nginx"
    }
    port {
      port        = 80
      target_port = 80
    }
    type = "ClusterIP"
  }
}
```

### 3.2. Terraform 생명주기 명령어 실행

작성된 코드를 바탕으로 인프라를 실제 클러스터에 배포한다.

```Bash
# 1. 작업 디렉토리 초기화 및 플러그인(Provider) 다운로드
terraform init

# 2. 실행 계획 확인 (어떤 리소스가 생성 및 변경되는지 시뮬레이션 결과 출력)
terraform plan

# 3. 인프라 배포 적용
# -auto-approve: 배포 진행 시 사용자에게 yes/no 프롬프트를 묻지 않고 즉시 변경 사항을 적용함
terraform apply -auto-approve
```

### 3.3. 배포 결과 확인 및 리소스 삭제

K8S 명령어로 상태를 확인하고, 테스트 종료 후 상태 파일(`terraform.tfstate`)을 기반으로 생성된 리소스를 일괄 철거한다.

```Bash
# K8S 클러스터에서 정상 배포 여부 확인
kubectl get all -n terraform-demo

# 테라폼으로 생성한 전체 인프라 일괄 삭제
# -auto-approve: 삭제 승인 프롬프트를 생략하고 즉시 삭제를 진행함
terraform destroy -auto-approve
```

---
