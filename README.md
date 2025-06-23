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

Запускаем minikube в режими однонодового кластера k8s

```bash
minikube start --vm-driver=docker
```
