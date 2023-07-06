# Реализация xiu кластера для развёртывания в production среде kubernetes

Xiu - a simple, high performance and secure live media server in pure Rust (RTMP[cluster]/HTTP-FLV/HLS).
https://github.com/harlanc/xiu.git

- Масштабирование кластера и load-balancing rtmp.
- ingress-конфиг для httpflv и hls endpoints на поддомены.
- HPA
- SSL для HLS и HTTPFLV
- liveness/readiness

---

Для реализации нам понадобится написать несколько yaml файлов с конфигурациями, которые в дальшейшем развернут сервис и необходимые компоненты в kubernetes.
Мы хотим развернуть сервис в продакшен среде. Для начала создадим отдельный неймспейс xiu-prod-namespace для сервиса.

**xiu-namespace.yaml**
```
apiVersion: v1
kind: Namespace
metadata:
  name: xiu-prod-namespace
  labels:
    app: xiu
    environment: prod
```
В лейблах указываем app и environment, которые в дальнейшем могут помочь нам получать информацию и выполнять действия с нашим развёртыванием с помощью селекторов.

Конфигурацию xiu мы положим в configMap. configMap далее пропишем в deployment, будем подключать его в директорию `/etc/xiu`.

**xiu-configmap.yaml**
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: xiu-node-configmap
  namespace: xiu-prod-namespace
  labels:
    app: xiu
    environment: prod
data:
  config_rtmp.toml: |
    [rtmp]
    enabled = true
    port = 1935

    [hls]
    enabled = true
    port = 8080
   
    [httpflv]
    enabled = true
    port = 8081
   
    [log]
```

Далее пишем deployment, который развернёт наше приложение.
Здесь же сделаем дополнительно:
- опишем Readiness и Liveness Probes, которые соответственно будут определять, когда приложение в поде начнёт отвечать на запросы и что оно продолжает работать после запуска.
- подключим конфигмап
- выберем стратегию rollingUpdate, чтобы поды уходили и создавались по одному
  
Судя по описанию xiu, каких-либо встроенных способов определения работоспособности нет. Из идей, мы можем, например:
- Смотреть на доступность портов. Тут есть момент, что Readiness\Liveness по tcpSocket может проверять доступность только одного порта. Можно сделать кастомную exec проверку или проверки с другого пода.
- Смотреть на логи xiu.
- Сделать другие кастомные проверки под exec или проверки httpGet (поднять страницу со статусом на поде, например).

**xiu-deployment.yaml**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: xiu-prod-deployment
  namespace: xiu-prod-namespace
  labels:
    app: xiu
    environment: prod
spec:
  replicas: 1
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  selector:
    matchLabels:
      app: xiu
      environment: prod
  template:
    metadata:
      labels:
        app: xiu
        environment: prod
    spec:
      containers:
      - name: xiu-prod-pod
        image: bgelov/1687346100-977d03e7f0746077d90baa216bbf61c2:1.0.0
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 1935
        - containerPort: 8080
        - containerPort: 8081
        # https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
        readinessProbe:
          tcpSocket:
            port: 8080
          initialDelaySeconds: 5
        livenessProbe:
          tcpSocket:
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 30
        volumeMounts:
        - name: config
          mountPath: "/etc/xiu"
          readOnly: true
      volumes:
        - name: config
          configMap:
            name: xiu-node-configmap
            items:
            - key: "config_rtmp.toml"
              path: "config_rtmp.toml"
```

Для возможности взаимодействия с нашими подами внутри кластера пишем service.

**xiu-service.yaml**
```
apiVersion: v1
kind: Service
metadata:
  name: xiu-prod-service
  namespace: xiu-prod-namespace
  labels:
    app: xiu
    environment: prod
spec:
  selector:
    app: xiu-prod-pod
  ports:
    - name: hls
      port: 8080
      targetPort: 8080
    - name: httpflv
      port: 8081
      targetPort: 8081
    - name: rtmp
      port: 1935
      targetPort: 1935
```

Чтобы с нашим приложением могли взаимодействовать внешние клиенты, напишем ingress.
- SSL для HLS и HTTPFLV описываем тут же. Добавляем сертификаты в секреты и прописываем их на поддомены.
- После создания ingress смотрим external ip на ingress и прописываем его в A запись поддоменов.
- Добавляем балансировщик Application Load Balancer Ingress controller

**xiu-ingress.yaml**
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: xiu-prod-ingress
  namespace: xiu-prod-namespace
  labels:
    app: xiu
    environment: prod
  annotations:
    # Дополнительные настройки для Application Load Balancer Ingress controller
    # https://cloud.yandex.ru/docs/managed-kubernetes/tutorials/new-kubernetes-project
    ingress.alb.yc.io/subnets: xxxxxxxxxxxxxxxxxxxx
    ingress.alb.yc.io/external-ipv4-address: auto
    ingress.alb.yc.io/group-name: xiu-prod-ingress
spec:
  tls:
    - hosts:
        - hls.vestan.ip03.ru
      secretName: yc-certmgr-cert-id-xxxxxxxxxxxxxxxxxxxx
    - hosts:
        - httpflv.vestan.ip03.ru
      secretName: yc-certmgr-cert-id-xxxxxxxxxxxxxxxxxxxx
  rules:
  - host: hls.vestan.ip03.ru
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: xiu-prod-service
            port:
              name: hls
  - host: httpflv.vestan.ip03.ru
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: xiu-prod-service
            port: 
              name: httpflv
```

Так же добавим HorizontalPodAutoscaler. Но применим его позже, чтобы протестировать scale вручную.

**xiu-autoscaller.yaml**
```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: xiu-prod-autoscaling
  namespace: xiu-prod-namespace
  labels:
    app: xiu
    environment: prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: xiu-prod-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 1
```


# Деплой в облачный kubernetes
Для тестирования развернём приложение в Managed Service for Kubernetes от Yandex Cloud.

Проверяем, что мы подключены к нужному контексту.
```
kubectl config current-context
```

Подключаюсь к кластеру, перехожу в каталог с yaml файлами и выполняю последовательно apply:
```
kubectl apply -f xiu-namespace.yaml

# сразу переключаюсь на новый неймспейс
kubectl config set-context --current --namespace=xiu-prod-namespace

kubectl apply -f xiu-configmap.yaml
kubectl apply -f xiu-deployment.yaml
kubectl apply -f xiu-service.yaml
kubectl apply -f xiu-ingress.yaml
```

## Работа масштабирования

Изначально в делплое у нас установлен `replicas: 1`, т.е. развернётся только один под. Проверяем командой
```
kubectl get pods

NAME                                   READY   STATUS    RESTARTS   AGE
xiu-prod-deployment-79c786c57d-znkqg   1/1     Running   0          27m
```

Для увеличения до 3 подов выполняем команду:
```
kubectl scale deployment xiu-prod-deployment --replicas=3

NAME                                   READY   STATUS    RESTARTS   AGE
xiu-prod-deployment-79c786c57d-54495   1/1     Running   0          5s
xiu-prod-deployment-79c786c57d-m8vhs   1/1     Running   0          5s
xiu-prod-deployment-79c786c57d-znkqg   1/1     Running   0          28m
```

Чтобы уменьшить количество подов до 2, вводим команду 
```
kubectl scale deployment xiu-prod-deployment --replicas=2

NAME                                   READY   STATUS        RESTARTS   AGE
xiu-prod-deployment-79c786c57d-54495   1/1     Running       0          57s
xiu-prod-deployment-79c786c57d-m8vhs   1/1     Terminating   0          57s
xiu-prod-deployment-79c786c57d-znkqg   1/1     Running       0          29m
```

И в итоге у нас остаётся только 2 пода
```
NAME                                   READY   STATUS    RESTARTS   AGE
xiu-prod-deployment-79c786c57d-54495   1/1     Running   0          96s
xiu-prod-deployment-79c786c57d-znkqg   1/1     Running   0          30m
```

Возвращаем снова до одного и тестируем HPA.
```
kubectl scale deployment xiu-prod-deployment --replicas=1
```

## HPA
Теперь попробуем реализовать автомасштабирование, Horizontal Pod Autoscaler. Для ,быстрого теста напишем условие, что если у нас будет замечена нагрузка по памяти больше 1 мегабайта, то автоматически увеличить количество подов.

Проверяем количество подов
```
kubectl get pods --namespace xiu-prod-namespace
NAME                                        READY   STATUS    RESTARTS   AGE
xiu-prod-deployment-7f57f5f5f6-b74m7        1/1     Running   0          131m
```

Применяем HPA

```
kubectl apply -f xiu-autoscaller.yaml
```

**xiu-autoscaller.yaml**
```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: xiu-prod-autoscaling
  namespace: xiu-prod-namespace
  labels:
    app: xiu
    environment: prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: xiu-prod-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 1
```

И получаем увеличение количества подов

```
NAME                                        READY   STATUS    RESTARTS   AGE
xiu-prod-deployment-7f57f5f5f6-8jgg6        1/1     Running   0          39s
xiu-prod-deployment-7f57f5f5f6-9458t        0/1     Pending   0          24s
xiu-prod-deployment-7f57f5f5f6-b74m7        1/1     Running   0          131m
xiu-prod-deployment-7f57f5f5f6-btp5x        0/1     Pending   0          39s
xiu-prod-deployment-7f57f5f5f6-gtxhb        0/1     Pending   0          39s
```
