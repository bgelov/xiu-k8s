# Задача 2
Реализация кластера для разверстки ее в production среде kubernetes.

Успешным результатом является:
- Возможность масштабирования кластера и load-balancing rtmp.
- Написан ingress-конфиг для httpflv и hls ендпоинтов на поддомены.

Срок выполнения 3 дня.

Решение предоставить в виде архива petrov-stage2-1685933169-1685933170.tar.xz, где timestamp - время получения и сдачи задания. И предоставить writeup с описанием решения WRITEUP.md в архиве.

## Требования к реализации
- Продемонстрировать работоспособность масштабирования. Сдеплоить в 1 экземпляре и с помощью scale увеличить значение до 3 и уменьшить до 2.
- Продемонстрировать работу балансировки нагрузки на rtmp-кластер.

Также, будет плюсом:
- HPA
- SSL для HLS и HTTPFLV
- liveness/readiness

---

# Решение

Для реализации решения нам понадобится написать несколько yaml файлов с конфигурациями, которые в дальшейшем развернут сервис и необходимые компоненты в kubernetes.

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

```

Далее пишем deployment, который развернёт наше приложение.
Здесь же опишем Readiness и Liveness Probes, которые соответственно будут определять, когда приложение в поде начнёт отвечать на запросы и что оно продолжает работать после запуска.

Судя по описанию xiu, каких-либо встроенных способов определения работоспособности нет. Из идей, мы можем, например:
- Смотреть на доступность портов. Тут есть момент, что Readiness\Liveness по tcpSocket может проверять доступность только одного порта.
- Смотреть на логи xiu.
- Сделать кастомные проверки под exec или проверки под httpGet, отдающие какой-либо статус.

**xiu-deployment.yaml**
```

```

Для возможности взаимодействия с нашими подами пишем service.

**xiu-service.yaml**
```

```

Чтобы с нашим приложением могли взаимодействовать внешние клиенты, напишем ingress.

**xiu-ingress.yaml**
```

```

Далее так же напишем HorizontalPodAutoscaler.

**xiu-autoscaller.yaml**
```

```


# Деплой в облачный kubernetes
Для тестирования развернё приложение в Managed Service for Kubernetes от Yandex Cloud.

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



kubectl get cm --namespace xiu-prod-namespace


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



## HPA
Теперь попробуем реализовать автомасштабирование, Horizontal Pod Autoscaler. Если у нас будет замечена нагрузка CPU больше заданной, то у нас автоматически увеличится количество подов.
```
kubectl apply -f xiu-namespace.yaml
```


, памяти или количество соединений.



Если мы будем использовать в такой конфигурации, то как я понимаю процесс, подать трансляцию сможет любой желающий. Следовательно, нам надо разграничить возможность просмотра трансляции и возможности трансляции.





# SSL для HLS и HTTPFLV
Воспользуемся сервисом Let's Encrypt.
Установим Cert-Manager
https://cert-manager.io/docs/installation/




# Упаковка решения

```
ARCHIVE_NAME="belov-stage1-$(date --utc -d "2023-06-21 11:15" +%s)-$(date --utc +%s)"
tar -cJf "${ARCHIVE_NAME}.tar.xz" xiu-compose xiu-image WRITEUP.md
```
