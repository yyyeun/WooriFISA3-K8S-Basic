# WooriFISA3-K8S-Basic
![image](https://github.com/user-attachments/assets/af9f615f-44d0-40fa-a834-eb284a5da5be)

<br>

## 개요


## 기술


## 1. SpringBoot Application Container화
### 1. JAR 파일 준비
Run As > Run Configuration > Gradle Task
![image](https://github.com/user-attachments/assets/e52009cf-d3fb-4ab5-981a-65d46dd4389b)

bootJar Task를 추가하고 Working Directory를 현재 프로젝트로 지정한 후, Task를 실행합니다.
![image](https://github.com/user-attachments/assets/0d5caca6-a9d5-4d3d-81bf-37184d638faf)
![image](https://github.com/user-attachments/assets/c4de2853-be49-495b-a6c4-064daa395c02)


### 2. Dockerfile 작성 및 Image Build
```bash
# 1. 빌드 단계
FROM openjdk:17-slim
WORKDIR /app
COPY SpringApp-0.0.1-SNAPSHOT.jar app.jar

# 2. 실행 단계
FROM openjdk:17-slim
WORKDIR /app
COPY --from=0 /app/app.jar app.jar

# JVM 최적화 옵션 추가
ENTRYPOINT ["java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

```bash
$ docker build -t bootappfork8s:1.0 .

$ docker images
REPOSITORY                    TAG         IMAGE ID       CREATED         SIZE
bootappfork8s                 1.0         094a56682e9c   6 seconds ago   428MB
```

### 3. Docker Hub에 이미지 push
```bash
$ docker login

$ docker tag bootappfork8s:1.0 yyyeun/bootappfork8s:1.0

$ docker push yyyeun/bootappfork8s:1.0
```
![image](https://github.com/user-attachments/assets/5c4de912-c61c-429f-bfe0-875935a95644)


<br>

## 2. Implement Kubernetes Architecture
### 1. Minikube cluster 실행
```bash
# Minikube 클러스터 시작
$ minikube start

$ kubectl get all
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   40s
```

### 2. Deployment, Service YAML 파일 작성
```yaml
# deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spring-boot
  template:
    metadata:
      labels:
        app: spring-boot
    spec:
      containers:
      - name: spring-boot-container
        image: yyyeun/bootappfork8s:1.0
        ports:
        - containerPort: 8999
```

```yaml
# service.yml
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-service
spec:
  type: LoadBalancer
  selector:
    app: spring-boot
  ports:
    - protocol: TCP
      port: 8999
      targetPort: 8999
```

```bash
$ kubectl apply -f deployment.yml 
deployment.apps/spring-boot-app created

$ kubectl apply -f service.yml 
service/spring-boot-service created

$ kubectl get all
NAME                                  READY   STATUS    RESTARTS   AGE
pod/spring-boot-app-88f889dbb-2sh6x   1/1     Running   0          8s
pod/spring-boot-app-88f889dbb-qn6xw   1/1     Running   0          8s
pod/spring-boot-app-88f889dbb-wh27h   1/1     Running   0          8s

NAME                          TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/kubernetes            ClusterIP      10.96.0.1       <none>        443/TCP          56m
service/spring-boot-service   LoadBalancer   10.104.146.73   <pending>     8999:31401/TCP   3s

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/spring-boot-app   3/3     3            3           8s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/spring-boot-app-88f889dbb   3         3         3       8s
```

<details>
<summary>참고: Command 버전</summary>

```bash
# 동일한 app을 3개의 container로 구성해서 생성 및 배포
# pod, deployment, replicaset 생성
$ kubectl create deployment bootapp --image=yyyeun/bootappf
ork8s:1.0 --replicas=3

# 외부 로드밸런서를 통해 특정 포트으로 App 배포
# service 생성
$ kubectl expose deployment bootapp --type=LoadBalancer --port=8999
service/bootapp exposed
```

</details><br>

![image](https://github.com/user-attachments/assets/1014a85d-f6e5-46b6-9821-7c29ca336d33)


### 3. 터널링을 통해 외부 IP 부여
```bash
# Minikube에서 외부 IP 제공받기 위한 터널링
$ minikube tunnel

$ minikube ip

$ kubectl get svc
NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
kubernetes            ClusterIP      10.96.0.1       <none>          443/TCP          63m
spring-boot-service   LoadBalancer   10.104.146.73   10.104.146.73   8999:31401/TCP   7m2s
```

포트 포워딩 <br>
![image](https://github.com/user-attachments/assets/d04344cd-fc31-42f8-bbe8-38ba5cd59d58)

![image](https://github.com/user-attachments/assets/a09256d5-3df1-47ab-b860-3c3764753734)

![image](https://github.com/user-attachments/assets/6299be29-c7e3-48ae-9206-0a842e6f799c)

![image](https://github.com/user-attachments/assets/2552795d-fa4d-4c2f-95b2-bcdf393bf73e)

![image](https://github.com/user-attachments/assets/f6b14a08-1c7d-41f9-98e0-a3f117c7320d)

<br>

## Trouble Shooting
### Trouble 1. 로컬 이미지 접근 불가
Pod가 컨테이너 이미지를 가져오지 못해 `ErrImagePull`, `ImagePullBackOff` 등의 상태를 보임

![image](https://github.com/user-attachments/assets/799d1cbc-f303-454a-88b3-ebaba67f7ffb)

문제는 Kubernetes가 로컬 Docker 이미지(bootappfork8s:1.0)를 찾을 수 없다는 점입니다. Kubernetes 클러스터는 노드에서 이미지를 가져올 때, 기본적으로 외부 레지스트리(예: Docker Hub)에서 이미지를 찾습니다. 하지만 로컬에만 있는 이미지를 Kubernetes는 접근할 수 없기 때문에 ErrImagePull이나 ImagePullBackOff 같은 에러가 발생할 수 있습니다.

### Solution
Docker Hub에 이미지 push
```bash
$ docker login

$ docker tag bootappfork8s:1.0 yyyeun/bootappfork8s:1.0

$ docker push yyyeun/bootappfork8s:1.0

$ kubectl create deployment bootapp --image=yyyeun/bootappf
ork8s:1.0 --replicas=3
```

![image](https://github.com/user-attachments/assets/93230a75-4137-4285-9cb1-b0c9c08fd2b5)

1. Docker Registry에 이미지 푸시하기
Docker Hub나 개인 레지스트리(Docker Registry)에 이미지를 푸시한 후 Kubernetes에서 해당 이미지를 사용할 수 있도록 해야 합니다.

Docker Hub에 로그인합니다:

bash
코드 복사
docker login
이미지를 Docker Hub에 푸시할 수 있도록 태그를 다시 지정합니다:

bash
코드 복사
docker tag bootappfork8s:1.0 <your-dockerhub-username>/bootappfork8s:1.0
이미지를 Docker Hub에 푸시합니다:

bash
코드 복사
docker push <your-dockerhub-username>/bootappfork8s:1.0
Kubernetes 배포를 다음과 같이 수정합니다:

bash
코드 복사
kubectl set image deployment/bootapp bootapp=<your-dockerhub-username>/bootappfork8s:1.0
2. Minikube나 Docker Desktop을 사용하는 경우 (로컬 클러스터)
Minikube나 Docker Desktop을 사용하는 경우, 로컬 클러스터에서 로컬 이미지를 바로 사용할 수 있습니다.

Minikube의 Docker 데몬을 사용하도록 설정합니다:

bash
코드 복사
eval $(minikube docker-env)
이미지를 다시 빌드하여 Minikube의 Docker 데몬에 추가합니다:

bash
코드 복사
docker build -t bootappfork8s:1.0 .
배포를 다시 생성합니다:

bash
코드 복사
kubectl create deployment bootapp --image=bootappfork8s:1.0 --replicas=3
이 방법을 사용하면 로컬 클러스터에서 직접 로컬 이미지를 사용할 수 있습니다.

<hr>

### Trouble 2. localhost 접근 불가
```bash
$ curl 10.109.30.69:8999/test
<h2>Kubernetes Architecture 구현</h2>

$ curl http://127.0.0.1:8999/test
curl: (7) Failed to connect to 127.0.0.1 port 8999 after 0 ms: Connection refused
```

```bash
$ docker ps

CONTAINER ID   IMAGE                                 COMMAND                  CREATED       STATUS       PORTS                                                                                                                                  NAMES
d0aae0b3187b   gcr.io/k8s-minikube/kicbase:v0.0.45   "/usr/local/bin/entr…"   5 hours ago   Up 5 hours   127.0.0.1:32778->22/tcp, 127.0.0.1:32779->2376/tcp, 127.0.0.1:32780->5000/tcp, 127.0.0.1:32781->8443/tcp, 127.0.0.1:32782->32443/tcp   minikube
```
