Задание:

Куберизировать приложение flask+redis: 
создать манифесты для запуска приложения flask и БД redis в кластере, и сервис сетевого доступа к ним.
Манифесты выложить в репо на гитхаб и в качестве ответа приложить ссылку.




Требования: 2CPU 4GB RAM 10GB HDD FREESPACE (можно меньше через отдельную настройку minicube) 

Установка необходимых пакетов

Скачиваем и инсталлируем kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Скачиваем и инсталлируем minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
```

```bash
sudo dpkg -i minikube_latest_amd64.deb
```

Запускаем minikube в режиме однонодового кластера k8s

```bash
minikube start --vm-driver=docker

😄  minikube v1.36.0 on Ubuntu 24.04 (vbox/amd64)
✨  Using the docker driver based on existing profile
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🚜  Pulling base image v0.0.47 ...
🔄  Restarting existing docker container for "minikube" ...
🐳  Preparing Kubernetes v1.33.1 on Docker 28.1.1 ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: default-storageclass, storage-provisioner
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default

```

Проверка minikube и k8s кластера на вм

```bash
minikube status                                                                  

minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

```bash
kubectl get nodes

NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   45m   v1.33.1

```

Cтруктура каталогов будет выглядеть следующим образом: Отдельный каталог для исходников приложения (из прошлой лабы) Отдельный каталог для манифестов k8s

```bash
.
├── flask
│   ├── app.py
│   ├── compose.yml
│   ├── dockerfile
│   └── requirements.txt
├── flask_k8s
│   ├── flask-service.yml
│   ├── flask.yml
│   ├── redis-service.yml
│   └── redis.yml
└── README.md


```

Производим сборку образа из исходников прямо внутри 

```bash
minikube minikube image build -t flask:v1 flask/
```

Загружаем готовый образ redis внутрь кластера

```bash
minikube image load redis:alpine
```

Проверяем, что образы стали доступны внутри кластера

```bash
minikube image ls  | grep docker                                                 
docker.io/library/redis:alpine
docker.io/library/flask:v1
```

Готовим описания (манифесты) 
flask_k8s/flask.yml

```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
  labels:
    app: flask-app
spec:
  replicas: 5
  selector:
    matchLabels:
      app: flask-app
      svc: front
  template:
    metadata:
      labels:
        app: flask-app
        svc: front
    spec:
      containers:
      - name: flask
        image: flask:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
        resources:
          limits:
            memory: "256Mi"
```


flask_k8s/flask-service.yml
```yml
---
apiVersion: v1
kind: Service
metadata:
  name: service-devops
  labels:
    app: flask-app
spec:
  type: LoadBalancer
  selector:
    app: flask-app
    svc: front
  ports:
  - port: 8000
    targetPort: 5000
  externalIPs:
  - 192.168.100.8
  - ```

flask_k8s/redis.yml
```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  labels:
    app: flask-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
        svc: db
    spec:
      containers:
      - name: redis
        image: redis:alpine
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 6379
```

flask_k8s/redis-service.yml
```yml
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    app: flask-app
spec:
  type: ClusterIP
  selector:
    app: flask-app
    svc: db
  ports:
  - port: 6379
    targetPort: 6379
```


применяем манифесты по отдельности файлами или весь каталог целиком

```bash
kubectl apply -f flask_k8s/
```


проверяем статус развертывания реплик
```bash
kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
flask-app-965787f44-58w8w   1/1     Running   0          75m
flask-app-965787f44-bxtb6   1/1     Running   0          75m
flask-app-965787f44-crgss   1/1     Running   0          75m
flask-app-965787f44-dktgb   1/1     Running   0          75m
flask-app-965787f44-dsx4m   1/1     Running   0          75m
redis-7587fd8fcf-pcwf5      1/1     Running   0          42m

```


проверяем сервисы
```bash
kubectl get services                                                                                                                                            
NAME             TYPE           CLUSTER-IP       EXTERNAL-IP               PORT(S)          AGE
kubernetes       ClusterIP      10.96.0.1        <none>                    443/TCP          3h25m
redis            ClusterIP      10.109.128.136   <none>                    6379/TCP         37m
service-devops   LoadBalancer   10.109.242.83    127.0.0.1,192.168.100.8   8000:30449/TCP   56m
```



Внутри вм в отдельном окне разрешаем проброс трафика вм внутрь minikube
```bash
minikube tunnel --bind-address 192.168.100.8 
```

Проверка через curl
```bash
curl 192.168.100.8:8000                                                                                                                                    Hello World! I have been seen 6 times. My name is: flask-app-965787f44-58w8w

curl 192.168.100.8:8000                                                                                                                                    
Hello World! I have been seen 7 times. My name is: flask-app-965787f44-dsx4m

curl 192.168.100.8:8000                                                                                                                                    
Hello World! I have been seen 8 times. My name is: flask-app-965787f44-dktg
```

Обновляем приложение на новую версию (Rolling Update)  


(Добавим к Hello World! VERSION 2)
```python
import redis 
from flask import Flask, make_response 
import socket 

app = Flask(__name__) 
cache = redis.Redis(host='redis', port=6379) 

def get_hit_count() -> int: 
    return int(cache.get('hits') or 0) 

def incr_hit_count() -> int: 
    return cache.incr('hits') 


@app.route('/metrics') 
def metrics():
    metrics = f'''
# HELP view_count Flask-Redis-App visit counter
# TYPE view_count counter
view_count{{service="Flask-Redis-App"}} {get_hit_count()}
''' # sic double quotes in label
    response = make_response(metrics, 200)
    response.mimetype = "text/plain" 
    return response 


@app.route('/') 
def hello(): 
    incr_hit_count() 
    count = get_hit_count() 
    return 'Hello World! VERSION 2 I have been seen {} times. My name is: {}\n'.format(count, socket.gethostname())
```

Пересобираем образ с новым тегом :v5 И делаем новый образ доступным в кластере 
```bash
minikube image build -t flask:v2 flask/
```

Обновляем поды на новую версию – патчим деплоймент # Можно обновить файл манифеста и применить его заново, # Можно пропатчить описание деплоймента прямо в БД k8s:

```bash
kbectl set image deployments/flask-app flask=flask:v2
```

Наблюдаем, как пересоздаются реплики подов на новую версию

```bash
kubectl get pods                                                                  
NAME                         READY   STATUS    RESTARTS   AGE
flask-app-5f77f99799-4xw9p   1/1     Running   0          92s
flask-app-5f77f99799-8z5gq   1/1     Running   0          89s
flask-app-5f77f99799-9vzjp   1/1     Running   0          92s
flask-app-5f77f99799-djklt   1/1     Running   0          92s
flask-app-5f77f99799-rr6vw   1/1     Running   0          89s
redis-7587fd8fcf-pcwf5       1/1     Running   0          67m

```

```bash
kubectl describe deployment flask-app                                            
Name:                   flask-app
Namespace:              default
CreationTimestamp:      Mon, 23 Jun 2025 08:20:50 +0000
Labels:                 app=flask-app
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               app=flask-app,svc=front
Replicas:               5 desired | 5 updated | 5 total | 5 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=flask-app
           svc=front
  Containers:
   flask:
    Image:      flask:v2
    Port:       5000/TCP
    Host Port:  0/TCP
    Limits:
      memory:      256Mi
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  flask-app-965787f44 (0/0 replicas created)
NewReplicaSet:   flask-app-5f77f99799 (5/5 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  3m13s  deployment-controller  Scaled up replica set flask-app-5f77f99799 from 0 to 2
  Normal  ScalingReplicaSet  3m13s  deployment-controller  Scaled down replica set flask-app-965787f44 from 5 to 4
  Normal  ScalingReplicaSet  3m13s  deployment-controller  Scaled up replica set flask-app-5f77f99799 from 2 to 3
  Normal  ScalingReplicaSet  3m10s  deployment-controller  Scaled down replica set flask-app-965787f44 from 4 to 3
  Normal  ScalingReplicaSet  3m10s  deployment-controller  Scaled up replica set flask-app-5f77f99799 from 3 to 4
  Normal  ScalingReplicaSet  3m10s  deployment-controller  Scaled down replica set flask-app-965787f44 from 3 to 2
  Normal  ScalingReplicaSet  3m10s  deployment-controller  Scaled up replica set flask-app-5f77f99799 from 4 to 5
  Normal  ScalingReplicaSet  3m10s  deployment-controller  Scaled down replica set flask-app-965787f44 from 2 to 1
  Normal  ScalingReplicaSet  3m8s   deployment-controller  Scaled down replica set flask-app-965787f44 from 1 to 
```

Проверка через curl
```bash
curl 192.168.100.8:8000                                                          
Hello World! VERSION 2 I have been seen 10 times. My name is: flask-app-5f77f99799-rr6vw

curl 192.168.100.8:8000                                                          
Hello World! VERSION 2 I have been seen 11 times. My name is: flask-app-5f77f99799-9vzjp

curl 192.168.100.8:8000                                                          
Hello World! VERSION 2 I have been seen 12 times. My name is: flask-app-5f77f99799-8z5gq

```
