# Домашнее задание к занятию "`Kubernetes. Часть 1`" - `Горелов Николай`


Это задание для самостоятельной отработки навыков и не предполагает обратной связи от преподавателя. Его выполнение не влияет на завершение модуля. Но мы рекомендуем его выполнить, чтобы закрепить полученные знания.

### Задание 1

**Выполните действия:**

1. Запустите Kubernetes локально, используя k3s или minikube на свой выбор.
1. Добейтесь стабильной работы всех системных контейнеров.

---

### Решение 1

1. Установка `kubectl`:

``` bash
curl -LO "https://dl.k8s.io/release/v1.27.0/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

2. Установка `minikube`:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

3. Установка `docker`:

```bash
sudo apt install docker -y
```

4.  Добавляем пользователя в группу docker:

```bash
sudo usermod -aG docker nimda
```

![](HW-files/1.JPG)

---

### Задание 2

Есть файл с деплоем:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  selector:
    matchLabels:
      app: redis
  replicas: 1
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: master
        image: bitnami/redis
        env:
         - name: REDIS_PASSWORD
           value: password123
        ports:
        - containerPort: 6379
```


**Выполните действия:**

1. Измените файл с учётом условий:

 * redis должен запускаться без пароля;
 * создайте Service, который будет направлять трафик на этот Deployment;
 * версия образа redis должна быть зафиксирована на 6.0.13.

2. Запустите Deployment в своём кластере и добейтесь его стабильной работы.

---

### Решение 2

1.  Создадим файл redis-deployment.yml с учётом условий задания:

``` yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  selector:
    matchLabels:
      app: redis
  replicas: 1
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: master
        image: bitnami/redis:6.0.13  # Зафиксированная версия образа
        ports:
        - containerPort: 6379
        env:
        - name: ALLOW_EMPTY_PASSWORD
          value: "yes"  # Разрешение на пустой пароль
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  selector:
    app: redis
  ports:
    - protocol: TCP
      port: 6379
      targetPort: 6379
  type: ClusterIP

```
2. Запустим и проверим состояние:

``` bash
nimda@vm2:~/kubernetes_part1$ kubectl apply -f redis-deployment.yml
```

![](HW-files/2.JPG)

---

### Задание 3

**Выполните действия:**

1. Напишите команды kubectl для контейнера из предыдущего задания:

 - выполнения команды ps aux внутри контейнера;
 - просмотра логов контейнера за последние 5 минут;
 - удаления контейнера;
 - проброса порта локальной машины в контейнер для отладки.

---

### Решение 3

Команды kubectl:

* Выполнение `ps aux` внутри контейнера:

``` bash
kubectl exec -it deployment/redis -c master -- ps aux
```

* Просмотр логов контейнера за последние 5 минут:

``` bash
kubectl logs -l app=redis --since=5m
```

* Удаление контейнера (точнее, Pod'а, управляемого Deployment'ом):

``` bash
kubectl delete pod -l app=redis
```
(Deployment автоматически создаст новый Pod)

* Проброс порта локальной машины в контейнер (port-forwarding):

``` bash
kubectl port-forward deployment/redis 6379:6379
```

(Теперь можно получить доступ к Redis на локальном порту 6379)

---

### Задание 4

Есть конфигурация nginx:

```
location / {
    add_header Content-Type text/plain;
    return 200 'Hello from k8s';
}
```

**Выполните действия:**

Напишите yaml-файлы для развёртки nginx, в которых будут присутствовать:

 - ConfigMap с конфигом nginx;
 - Deployment, который бы подключал этот configmap;
 - Ingress, который будет направлять запросы по префиксу /test на наш сервис.

 ---

### Решение 4

YAML-файлы для развёртывания Nginx:

configmap.yaml:

``` yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    location / {
        add_header Content-Type text/plain;
        return 200 'Hello from k8s';
    }
```


deployment.yaml:

``` yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/conf.d
      volumes:
      - name: nginx-config-volume
        configMap:
          name: nginx-config
```

ingress.yaml:

``` yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - http:
      paths:
      - path: /test
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Применение YAML-файлов:

``` bash
kubectl apply -f configmap.yaml
kubectl apply -f deployment.yaml
kubectl apply -f ingress.yaml
```

Теперь можно получить доступ к Nginx через Ingress. Для Minikube нужно узнать IP адрес ingress контроллера:

``` bash
minikube ip
```

Затем открыть в браузере `[minikube_ip]/test`. Должно отобразиться "Hello from k8s".

---