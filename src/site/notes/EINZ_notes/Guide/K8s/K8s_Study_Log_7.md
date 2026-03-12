---
{"dg-publish":true,"permalink":"/einz-notes/guide/k8s/k8s-study-log-7/","title":"7: 상태 유지형 애플리케이션 (StatefulSet)"}
---


# 상태 유지형 애플리케이션 (StatefulSet)

## Deployment vs StatefulSet의 스토리지 구조 차이

두 객체 모두 파드를 관리하지만, 스토리지를 다루는 철학이 완전히 다름. 데이터베이스와 같이 데이터의 독립성과 영속성이 보장되어야 하는 환경에서는 반드시 StatefulSet을 사용해야 한다.

* **Deployment + PVC (무상태 앱의 스토리지 공유):**
  * 사용자가 직접 1개의 PVC를 생성하고 Deployment에 연결.
  * 파드가 여러 개로 늘어나면(Scale-out), 모든 파드가 동일한 1개의 PVC(디스크)를 공유해서 바라봄.
  * 용도: 웹 서버의 정적 이미지 폴더 등 모든 파드가 동일한 데이터를 읽어야 할 때.

* **StatefulSet + PVC (상태 유지 앱의 독립 스토리지):**
  * 사용자가 직접 PVC를 만들지 않고, `volumeClaimTemplates`(PVC 생성 틀)을 선언.
  * K8S가 파드를 생성할 때마다 이 틀을 이용해 파드 전용 PVC를 1:1로 자동 생성하여 매칭함 (예: 파드-0은 PVC-0, 파드-1은 PVC-1).
  * 용도: MySQL, Redis 등 각 파드가 고유한 마스터/슬레이브 데이터를 독립적으로 가져야 할 때.

---

## 1. StatefulSet과 영구 볼륨(PV/PVC) 배포 실습

### 1.1 Headless Service 및 StatefulSet 설계도 작성

StatefulSet은 파드의 고정된 네트워크 신원을 보장하기 위해 IP가 할당되지 않는 Headless Service(`clusterIP: None`)를 동반해야 한다.

```yaml
# stateful-app.yaml
# 1. Headless Service
apiVersion: v1
kind: Service
metadata:
  name: nginx-stateful-svc
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
# 2. StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-stateful
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx-stateful-svc"
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www-data
          mountPath: /usr/share/nginx/html
  # 3. PVC 붕어빵 틀 (각 파드마다 1GB 독립 스토리지 할당)
  volumeClaimTemplates:
  - metadata:
      name: www-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
````

### 1.2 클러스터 적용 및 생성 규칙 관찰

```bash
# 설계도 적용
kubectl apply -f stateful-app.yaml

# 파드 생성 실시간 관찰
kubectl get pods -w
```

- **검증:** 파드가 무작위 해시값이 아닌 `web-stateful-0`, `web-stateful-1`과 같이 순차적이고 고정된 이름으로 생성된다. 0번 파드가 완전히 실행(Running)된 이후에 1번 파드가 순차적으로 생성됨을 확인.

### 1.3 영구 볼륨(PVC) 1:1 자동 생성 확인

```bash
kubectl get pvc
```

- **검증:** `www-data-web-stateful-0` 및 `www-data-web-stateful-1`이라는 이름으로 각 파드에 종속된 1GB 스토리지가 Bound(연결) 상태로 생성된다.

---

## 2. 고의적 장애 발생 및 데이터 영속성 테스트

파드가 파괴되고 재생성되어도 연결된 PVC를 통해 데이터가 보존되는지 검증해본다.

### 2.1 0번 파드 스토리지에 고유 데이터 기록

```bash
kubectl exec -it web-stateful-0 -- sh -c 'echo "This data must survive a pod restart!" > /usr/share/nginx/html/index.html'
```

### 2.2 파드 강제 삭제 (장애 시뮬레이션)

```bash
kubectl delete pod web-stateful-0
```

### 2.3 자동 복구 및 데이터 보존 확인

```bash
# 파드 재생성 대기 (StatefulSet 컨트롤러가 즉시 동일한 이름으로 파드를 다시 살려냄)
kubectl get pods

# 재생성된 파드 내부에 접속하여 데이터 확인
kubectl exec -it web-stateful-0 -- cat /usr/share/nginx/html/index.html
```

- **검증:** 파드가 삭제되고 완전히 새로운 컨테이너로 재생성 되었음에도, K8S가 기존 PVC(`www-data-web-stateful-0`)를 정확히 다시 마운트하여 **"This data must survive a pod restart!"** 메시지가 유실 없이 그대로 출력된다.

---
