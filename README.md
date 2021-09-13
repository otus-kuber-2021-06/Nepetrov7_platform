# Nepetrov7_platform
Nepetrov7 Platform repository
## домашнее задание 1. (преподготовка репозитория)
1. добавил в репозиторий файлы .github/auto_assign.yml .github/labeler.yml .github/workflows/auto-assign.yml .github/workflows/labeler.yml .github/workflows/run-tests.yml
1. дождался автопроверки, сделал merge-request

## домашнее задание 2 (Настройка локального окружения. Запуск первого контейнера.Работа с kubectl)
1. установил настроил kubectl
1. установил и запустил minikube
1. подключил дашборд для kubernetes `https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/`
1. создал Dockerfile и образ из него. выбрал centos7 с установленным на него nginx (в задании было разрешено использовать любой вариант)
1. добавил образ в dockerhub
1. написал манифест pod-a и применил его `web-pod.yaml`
1. с помощью Dockerfile приложения frontend был создан образ frontend и добавлен в dockerhub
1. создал манифест pod-a frontend. `kubectl run frontend --image nepetrov7/frontend --restart=Never --dry-run=client -o yaml > frontend-pod.yaml`
1. применил манифест, но он был в статусе Error. озучив describe пода - обнаружил, что сервис не может найти переменные
1. добавил переменные, необходимые для работы сервиса, запустил сервис
1. манифест с необходимыми переменными поместил в файл `frontend-pod-healthy.yaml`

## домашнее задание 3 (Kubernetes controllers. ReplicaSet, Deployment, DaemonSet)
1. был применен выданным манифест replicaset для frontend
1. в манифесте была ошибка, не было раздела selector, о чем была соотвествующая ошибка в describe rs
1. поправил манифест, запустил 3 реплики
1. изменил образ приложения в манифесте, применил. вопрос в ДЗ - почему поды не пересоздались? - ответ: потому что replicaset следит только за количеством запущенных подов, это инструмент не для обновления приложения в подах, а для слежения за количеством реплик.
1. создал 2 образа микросервиса paymentService и залил в докер хаб
1. сделал манифест для replicaset и deployments микросервиса paymentService
1. при смене версии образа в манифесте deployments и его применении - добавляется новый replicaset и запускает новые поды
1. по умолчанию применяется стратегия Rolling Update. принцип: создание нового пода и адаление старого (по одному)
1. сделал откат версии deployments (`kubectl rollout undo deployment paymentservice --to-revision=1`)
1. сделал задание со звездочкой (2 манифеста):
    - аналог blue-green: развертывание трех новых подов, затем удаление трех старых
    - Reverse Rolling Update: удаление одного старого пода, затем созданиие нового
1. сделал манифест deployments для frontend и добавил в него предоставленную readinessProbe
1. изучил причину нахождения пода в состоянии `0/1 Running` - readinessProbe не прошла. поправили манифест
1. при обновлении, если readinessProbe первого пода из deployments не прошла - то deployments не будет пытаться обновлять дальше
1. запустил DaemonSet для node-exporter
1. пробросил порты с пода (`kubectl port-forward <имя любого pod в DaemonSet> 9100:9100`)
1. проверил метрики ноды (`curl localhost:9100/metrics`)
1. задание со звездочкой: найти способ модернизировать DaemonSet таким образом, чтобы он запускался в том числе и на мастер нодах
    - в официальном репозитории мне попался сразу же "правильный" DaemonSet, который разворачивает node-exporter на всех нодах kubernetes
    - если сделать describe мастер ноды, то мы увидем там `Taints: node-role.kubernetes.io/master:NoSchedule` что запрещает разворачивать 
    - если в `spec.tolerations` указать `operator: "Exists"`, `effect: "NoSchedule"`, то поды будут размещаться и на мастер ноды тоже

# домашнее задание 4 (kubernetes-security)
1. task01:
    - Создать Service Account bob, дать ему роль admin в рамках всего кластера {`01_sa_bob_admin.yml`}
    - Создать Service Account dave без доступа к кластеру (`02_sa_dave.yml`)
1. task2:
    - Создать Namespace prometheus (`01_ns_prometheus.yml`)
    - Создать Service Account carol в этом Namespace (`02_sa_carol.yml`)
    - Дать всем Service Account в Namespace prometheus возможность делать get, list, watch в отношении Pods всего кластера (`03_clusterrole_and_binding.yml`)
1. task03:
    - Создать Namespace dev (`01_ns_dev.yml`)
    - Создать Service Account jane в Namespace dev (`02_sa_jane.yml`)
    - Дать jane роль admin в рамках Namespace dev (`03_jane_admin_dev.yml`)
    - Создать Service Account ken в Namespace dev (`04_sa_ken.yml`)
    - Дать ken роль view в рамках Namespace dev (`05_ken_view_dev.yml`)

# доманшее задание 5 (kubernetes-network)
1. добавили в `web-pod.yaml` readinessProbe:
```
readinessProbe:
httpGet:
    path: /index.html
    port: 80
```
1. запустили под, он перешел в состояние
```
NAME READY STATUS  RESTARTS AGE
web  0/1   Running 0        5m47s
```
в describe видим что `readinessProbe` не удалась, так как наш под слушает порт 8000, а мы установили пробу на 80 порт:
`Readiness probe failed: Get http://172.24.204.119:80/index.html: dial tcp 172.24.204.119:80: connect: connection refused`
1. установили `livenessProbe` вида:
```
livenessProbe:
tcpSocket: { port: 8000 }
```
проба прошла
1. вопрос для самопроверки:
- Почему следующая конфигурация валидна, но не имеет смысла?
```
livenessProbe:
  exec:
    command:
      - 'sh'
      - '-c'
      - 'ps aux | grep my_web_server_process'
```
    - Ответ: потому что она всегда выполнится и она ничего не проверяет. в этом выводе будет всегда как минимум сам `grep`
- Бывают ли ситуации, когда она все-таки имеет смысл?  
    - Ответ: в таком виде нет. если только убрать сам grep из вывода: `ps aux | grep my_web_server_process | grep -v grep`. это может иметь смысл на случай проверки "имеется ли в контейнере кокретный процесс".
1. создаем deployment для web-pod `web-deploy.yaml` просто добавив:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
```
1. удаляем старый под `kubectl delete pod/web --grace-period=0 --force`
1. деплоим новый и видим что под не запустился `Available False MinimumReplicasUnavailable` в блоке conditions
1. меняем в readinessProbe порт на 8000, увеличиваем количество реплик на 3
1. вставляем описание стратегии в spec
```
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 100%
```
    - нельзя ставить `maxUnavailable` и `maxSurge` со значением 0 одновременно.
1. создание Service ClusterIP `web-svc-cip.yaml`
- `ClusterIP` удобны в тех случаях, когда:
    - Нам не надо подключаться к конкретному поду сервиса
    - Нас устраивается случайное расределение подключений между подами
    - Нам нужна стабильная точка подключения к сервису, независимая от подов, нод и DNS-имен
1. деплоим его в кластер и смотрим его ip `kuebctl get service`
1. заходим на vm minikube `minikube ssh` далее `sudo -i`
1. делаем `curl  http://<CLUSTER-IP>/index.html` - работает
1. делаем `ping <CLUSTER-IP>` - пинга нет
1. обращаем внимание на то что ни в `arp -an` ни в `ip addr show` нет ClusterIP, а вот в `iptables --list -nv -t nat` ClusterIP есть
    - Нужное правило находится в цепочке `KUBE-SERVICES`
    - Затем мы переходим в цепочку `KUBE-SVC-...` - здесь находятся правила "балансировки" между цепочками `KUBE-SEP-...`
        - SVC - Service
    - В цепочках `KUBE-SEP-...` находятся конкретные правила перенаправления трафика (через DNAT)
        - SEP - Service Endpoint
    - подробнее: https://msazure.club/kubernetes-services-and-iptables/
1. включаем ipvs для kube-proxy "на живую" `kubectl -n kube-system edit configmaps kube-proxy`
    - затем в ключ `data.config.conf:|- mode: ""` вписываем значение `ipvs`
    - и в ключ `ipvs.strictARP:` указываем `true`
    - удалим под kube-proxy `kubectl -n kube-system delete pod --selector='k8s-app=kube-proxy'`
    - описание работы и настройки ipvs в k8s: `https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md`
    - Причины включения `strictARP`: `https://github.com/metallb/metallb/issues/153`
    - входим снова на ноду, проверяем: `iptables --list -nv -t nat`
        - видим что у цепочек 0 preference
        - kube-proxy настроил все по новому, но не удалил мусор
        - запуск kube-proxy --cleanup не помогает `kubectl -n kube-system exec kube-proxy-<pod> kube-proxy --cleanup <pod>`
    - полностью очистим все правила iptables
        - создадим файл `/tmp/iptables.cleanup` с содержимым
            ```
            *nat
            -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
            COMMIT
            *filter
            COMMIT
            *mangle
            COMMIT
            ```
        - применим конфигурацию: `iptables-restore /tmp/iptables.cleanup`
        - проверим результат: `iptables --list -nv -t nat`
    - лишние правила удалены, мы видим только актуальную конфигурацию, kube-proxy периодически делает полную синхронизацию правил в своих цепочках
    - так как на vm нет утилиты `ipvsadm` - запустим `toolbox` и установим в него `ipvsadm`
        - `dnf install ipvsadm -y && dnf clean all`
        - проверяем наш сервис: `ipvsadm --list -n` и находим там правила распределения нагрузки
    - теперь делаем снова ping clusterIP, и видим что он стал пинговаться, все потому что этот ip теперь есть на интерфейсе `kube-ipvs0`
        - проверить это можно так: `ip addr show kube-ipvs0`
1. устаналиваем MetalLB (для теста можно как есть, для запуска в продакшен среде нужно править манифесты)
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```
1. проверяем все ли создалось: `kubectl --namespace metallb-system get all`
```
NAME                            READY   STATUS    RESTARTS   AGE
pod/controller-fb659dc8-jltcq   1/1     Running   0          5m56s
pod/speaker-2gp4d               1/1     Running   0          5m56s

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
daemonset.apps/speaker   1         1         1       1            1           beta.kubernetes.io/os=linux   5m56s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   1/1     1            1           5m56s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-fb659dc8   1         1         1       5m56s
```
1. настраиваем балансировщик с помощью ConfigMap `metallb-config.yaml`
- В конфигурации мы настраиваем:
    - Режим L2 (анонс адресов балансировщиков с помощью ARP)
    - Создаем пул адресов 172.17.255.1 - 172.17.255.255 - они будут назначаться сервисам с типом `LoadBalancer`
1. применяем манифест
1. создаем манифест для LoadBalancer `web-svc-lb.yaml`
1. присвоеный ip можно увидеть `kubectl describe svc web-svc-lb`
1. теперь можно обращаться по нему curl-ом и каждый раз попадать на другой под
1. по умолчанию ipvs использует rr (Round-Robin) балансировку. доступные алгоритмы балансировки: https://github.com/kubernetes/kubernetes/blob/1cb3b5807ec37490b4582f22d991c043cc468195/pkg/proxy/apis/config/types.go#L185
1. Задание со звездочкой: DNS через MetalLB
- создаем манифест сервиса, который открывает доступ к CoreDNS снаружи кластера `coredns/dns-svc-metallb.yml`
    - не забываем, что dns работает по TCP и UDP, поэтому сервиса 2 на один и тот же IP
    - для проверки нужно с ноды кластера выполнить: `nslookup coredns-svc-tcp.kube-system.svc.cluster.local 172.17.255.10`
    - документация: https://metallb.universe.tf/usage/
1. деплоим ingress-nginx: `kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml`
1. создаем и деплоим `nginx-lb.yaml`
1. теперь можно делать curl на этот ip
1. подключение приложения web к ingress. создаем headless-сервис `web-svc-headless.yaml`
1. создаем ingress-прокси, создав манифест с ресурсом ingress `web-ingress.yaml `
- теперь можно обращаться к странице поду по адресу: `http://<LB_IP>/web/index.html`
- теперь балансировка выполняется посредством nginx, а не IPVS
1. Задание со звездочкой* |  Ingress для Dashboard
- сделать доступ к `kubernetes-dashboard` через ingress-прокси
- сервис должен быть доступен  через префикс /dashboard ).
- решение:
    - устанавливаем дашборд: `kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml`
    - пишем манифест и деплоим ингресс `./dashboard/dashboard-ingress.yaml`
1. Задание со звездочкой * | Canary для Ingress
- Реализуйте канареечное развертывание с помощью ingress-nginx
- Перенаправление части трафика на выделенную группу подов должно происходить по HTTP-заголовку
- решение:
    - пишем 2 манифеста ingress `.canary/ing-web-*.yaml`, в canary ingress добавляем следующие annotation:
        - `nginx.ingress.kubernetes.io/canary: "true"` - метка что этот ингресс является канареечным
        - `nginx.ingress.kubernetes.io/canary-weight: "20"` - тут указываем сколько процентов трафика быдет уходить на канареечные поды
        - не забываем, что без доменного имени это работать не будет.

# доманшее задание 6 (kubernetes-volumes)
- развернуть StatefulSet с minIO - локальным s3 хранилищем
    - деплооим 2 манифеста: `https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-02/Kuberenetes-volumes/minio-statefulset.yaml` и `https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-02/Kuberenetes-volumes/minio-headless-service.yaml`
- задание со звездочкой* переместить креды в secret и настроить конфигурацию на их пользование из секрета
    - решение: `minio-statefulset.yaml` и `minio-creds-secret.yml`
