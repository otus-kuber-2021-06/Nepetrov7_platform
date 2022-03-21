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

# домашнее задание 7 (kubernetes-templating)
1. Создаем кластер на google cloud
1. добавляем репозиторий `google-cloud-sdk`
    ```
    cat /etc/yum.repos.d/google-cloud-sdk.repo
    [google-cloud-sdk]
    name=Google Cloud SDK
    baseurl=https://packages.cloud.google.com/yum/repos/cloud-sdk-el8-x86_64
    enabled=1
    gpgcheck=1
    repo_gpgcheck=0
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    ```
1. ставим из него пакет: `sudo yum install -y google-cloud-sdk`
1. настраиваем kubectl на локальной машине: `sudo gcloud container clusters get-credentials k8s --zone europe-west3-a --project infra-327408`
1. устанавливаем helm 3
1. добавляем репозиторий с stable чартами `helm repo add stable https://charts.helm.sh/stable`
1. деплоим nginx-ingress `helm upgrade --install nginx-ingress stable/nginx-ingress --wait --namespace=nginx-ingress --version=1.41.3`
1. добавляем репозиторий с cert-manager `helm repo add jetstack https://charts.jetstack.io`
1. ставим CDR: `kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.16.1/cert-manager.crds.yaml`
1. создаем ns `kubectl create ns cert-manager`
1. ставим cert-manager `helm upgrade --install cert-manager jetstack/cert-manager --wait --namespace=cert-manager --version=0.16.1`
1. создаем и деплоим ClusterIssuer `cert-manager/clusterissuer.yml`
## задание со *. установить и разобраться в использовании chartmuseum
1. создаем namespase `kubectl create ns chartmuseum`
1. создаем `chartmuseum/values.yaml`
1. деплоим chartmuseum в kubernetes `helm upgrade --install chartmuseum stable/chartmuseum --wait --namespace=chartmuseum --version=2.13.2 -f ./chartmuseum/values.yaml`
1. проверяем, что chartmuseum установлен `export POD_NAME=$(kubectl get pods --namespace chartmuseum -l "app=chartmuseum" -l "release=chartmuseum" -o jsonpath="{.items[0].metadata.name}") ; kubectl port-forward $POD_NAME 8080:8080 --namespace chartmuseum` `curl http://127.0.0.1:8080/` или `https://chartmuseum.34.141.76.119.nip.io/`
1. ставим gofish `curl -fsSL https://raw.githubusercontent.com/fishworks/gofish/main/scripts/install.sh | bash`
1. `gofish init` `gofish install gofish` `gofish upgrade gofish`
1. ставим chartmuseum `gofish install chartmuseum`
1. создаем сервис аккаунт по инструкции: `https://cloud.google.com/docs/authentication/getting-started`
1. создаем переменную среды с путем до ключа `export GOOGLE_APPLICATION_CREDENTIALS="/home/user/Downloads/[FILE_NAME].json"`
1. Создаем google storage bucket: `https://cloud.google.com/storage/docs/creating-buckets`
1. 
    ```
    chartmuseum --debug --port=8080   --storage="google"   --storage-google-bucket="infra-bucket777"   --storage-google-prefix=""
    2021-10-13T12:37:31.651+0300    DEBUG   Fetching chart list from storage        {"repo": ""}
    2021-10-13T12:37:31.795+0300    DEBUG   No change detected between cache and storage    {"repo": ""}
    2021-10-13T12:37:31.795+0300    INFO    Starting ChartMuseum    {"host": "0.0.0.0", "port": 8080}
    2021-10-13T12:37:31.795+0300    DEBUG   Starting internal event listener
    ```
1. ставим плагин `https://github.com/chartmuseum/helm-push`
1. добавляем репозиторий `helm repo add chartmuseum https://chartmuseum.34.159.243.248.nip.io`
1. скачиваем чарт для теста: `helm fetch --untar kubernetes-dashboard/kubernetes-dashboard`
1. пушим тестовый чарт в репозиторий `helm cm-push kubernetes-dashboard/ chartmuseum`
curl --data-binary "@kubernetes-dashboard-5.0.4.tgz" http://localhost:8080/api/charts

1. обновляем индекс репозитория `helm repo update`
1. устанавливаем с нашего репозитория дашборд `helm install kubernetes-dashboard chartmuseum/kubernetes-dashboard --namespace=kubernetes-dashboard --create-namespace`
1. complete!
## harbor
1. добавляем репозиторий `helm repo add harbor https://helm.goharbor.io`
1. скачиваем чарт `helm fetch --untar harbor/harbor --version=1.8.0`
1. редактируем `values.yml` в соотвествии с ТЗ
1. ТЗ:
    - Должен быть включен ingress и настроен host harbor.<IPадрес>.nip.io
    - Должен быть включен TLS и выписан валидный сертификат
    - Скопируйте используемый файл values.yaml в директорию kubernetes-templating/harbor/
1. устанавливаем herbor `helm upgrade --install harbor harbor/harbor --version=1.1.2 --namespace=harbor --create-namespace --values kubernetes-templating/harbor/values.yaml`

## создаем свой helm chart
- `helm create hipster-shop`
- удаляем все содержимое templates и values.yml
- и из содержимого файла `https://github.com/express42/otus-platform-snippets/blob/master/Module-04/05-Templating/manifests/all-hipster-shop.yaml` создаем чарты hipster-shop и frontend
- добавляем `frontend` как зависимость для чарта `hipster-shop`, для этого прописываем в файл `./hipster-shop/Chart.yaml` следующие строки:
    ```
    dependencies:
    - name: frontend
        version: 0.1.0
        repository: "file://../frontend"
    ```
- а затем обновляем зависимости `helm dep update kubernetes-templating/hipster-shop`
- после этого появится файл `kubernetes-templating/hipster-shop/charts/frontend-0.1.0.tgz`
- теперь можно деплоить чарт `hipster-shop` и автоматически задеплоится чарт `frontend`
- меняем NodePort с помощью ключа `--set` не меняя его в values.yaml `helm upgrade --install hipster-shop kubernetes-templating/hipster-shop --namespace=hipster-shop --set frontend.service.NodePort=31234`
## задение со * добавить в зависимости redis, чтобы он деплоился вместе с релизом hipster-shop
- добавляем репозиторий bitnami `helm repo add bitnami https://charts.bitnami.com/bitnami`
- добавляем в `Chart.yml` в раздел `dependencies` строки:
    ```
    - name: redis
        version: latest
        repository: https://charts.bitnami.com/bitnami
    ```
## Научиться работать с helm-secrets | Необязательное задание.
- ставим зависимости: `yum install sops gnupg2 gnu-getopt`
- ставим плагин `helm plugin install https://github.com/futuresimple/helm-secrets`
- гененируем gnupg key `gpg --full-generate-key`
- проверяем наличие ключа `gpg -k`
- создаем файл `kubernetes-tamplating/frontend/secret.yaml` и помещаем в него `visibleKey: hiddenValue`
- шифруем файл secret.yaml `sops -e -i --pgp <$ID> secrets.yaml` указывая id своего ключа
- проверяемм содержимое файла, и видим ключ и его зашифрованное значение. в таком виде серкрет можно коммитить в гит
- расшифровывать можно любой из команд: `helm secrets view secrets.yaml` или `sops -d secrets.yaml`
- чтобы поместить значение этого секрета в kubernetes:
    - создаем файл `kubernetestemplating/frontend/templates/secret.yaml` с содержимым:
    ```
apiVersion: v1
kind: Secret
metadata:
  name: secret
type: Opaque
data:
  visibleKey: {{ .Values.visibleKey | b64enc | quote }}
    ```
    - при деплое передаем файл `secrets.yaml` как файл `values`, плагин `helm-secrets` расшифрует его, сложит значение во временный файл `secrets.yaml.dec` значение перменной вставит в шаблон секрета, а после деплоя удалит его.
    - деплоим: `helm secrets upgrade --install hipster-shop kubernetes-templating/frontend --namespace hipster-shop -f kubernetes-templating/frontend/values.yaml -f kubernetes-templating/frontend/secrets.yaml`
- залить чарты в публичный harbor
    - `helm package  kubernetes-templating/hipster-shop/`, полученные архив загружаем в harbor
    - добавляем файл kubernetes-templating/repo.sh для добавления преподавателем публичного репозитория себе.

## kubecfg
- выносим service и deployment от сервисов paymentservice и shippingservice в папку `kubernetes-templating/kubecfg`
- деплоим снова наш чарт и видим, что при добавлении в козину товаров - сайт выдает ошибку
- устанавливаем kubecfg
- Kubecfg предполагает хранение манифестов в файлах формата .jsonnet. возьмем такой файл с официального репозитория: `https://github.com/bitnami/kubecfg/blob/master/examples/guestbook.jsonnet`
- Для начала в файле мы должны указать libsonnet библиотеку, которую будем использовать для генерации манифестов. `https://github.com/bitnami-labs/kube-libsonnet/`
- получился файл `kubernetes-templating/kubecfg/services.jsonnet`
- деплоим `kubecfg update kubernetes-templating/kubecfg/services.jsonnet --namespace hipster-shop`

## Kustomize
- выносим еще один сервис в отдельную директорию `kubernetes-templating/kustomize/cartservice` и разносим по разным файлам.
- пишем файл `kubernetes-templating/kustomize/cartservice/kustomization.yaml`
- по тз нужно, чтобы деплой сервиса происходил по команде `kubectl apply -k kubernetes-templating/kustomize/overrides/<Название окружения>/` в разные нейспейсы с разными тегами и префиксом в названии. для этого создаем файлы в директории `kubernetes-templating/kustomize/overrides/*/kustomization.yaml`, в которых ссылаемся на файл, описанный в предыдущем пункте.

# домашнее задание 8 (kubernetes-operator)
- определяем customResource CustomResourceDefinition-ом 
- создаем описание кастом ресурса kubernetes-operators/deploy/crd.yml и сам ресурс kubernetes-operators/deploy/cr.yml
- определяем `openAPIV3Schema` для того чтобы был фиксированный тип значений всех полей у кастом ресурса
- прописываем обязательные ключи, чтобы без их определения задеплоить ресур нельзя было, поле `required`

1. написание контроллера для обработки двух типов событий следующими действиями:
    - При создании объекта типа `kind: mySQL`, он будет:
        - Cоздавать PersistentVolume, PersistentVolumeClaim, Deployment, Service для mysql
        - Создавать PersistentVolume, PersistentVolumeClaim для бэкапов базы данных, если их еще нет.
        - Пытаться восстановиться из бэкапа
    - При удалении объекта типа `kind: mySQL`, он будет:
        - Удалять все успешно завершенные backup-job и restore-job
        - Удалять PersistentVolume, PersistentVolumeClaim, Deployment, Service для mysql
1. выполнение:
    - для этого создаем темплейты всех этих сущностей в `kubernetes-operators/build/templates/*`
    - описываем python приложение в `kubernetes-operators/build/mysql-operator.py`
    - не забываем передать в backup pvc все параметры, в том числе и `storage_size`
    - пишем dockerfile и заливаем его на `hub.docker.com`
    ```
[user@localhost deploy]$ k get pvc
NAME                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
backup-mysql-instance-pvc   Bound    pvc-a475fd09-d3ca-406a-8ae0-bc9bed15bf8d   1Gi        RWO            standard       6s
mysql-instance-pvc          Bound    pvc-9691af6c-daad-4fff-81da-cc96a6353a63   1Gi        RWO            standard       6s
    ```
    - выполняем проверку
        - `kubectl exec -it $MYSQLPOD -- mysql -u root -potuspassword -e "CREATE TABLE test ( id smallint unsigned not null auto_increment, name varchar(20) not null, constraint pk_example primary key (id) );" otus-database`
        - `kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) VALUES (null, 'some data' );" otus-database`
        - `kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) VALUES (null, 'some data-2' );" otus-database`
        - `kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database`
        ```
+----+-------------+
| id | name        |
+----+-------------+
|  1 | some data   |
|  2 | some data-2 |
+----+-------------+
        ```
        - `kubectl delete mysqls.otus.homework mysql-instance` - удаляем
        - `export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")`
        - `kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database`
        ```
+----+-------------+
| id | name        |
+----+-------------+
|  1 | some data   |
|  2 | some data-2 |
+----+-------------+
        ```
```
[user@localhost deploy]$ export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
bectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database[user@localhost deploy]$ kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-------------+
| id | name        |
+----+-------------+
|  1 | some data   |
|  2 | some data-2 |
+----+-------------+
[user@localhost deploy]$ kubectl get jobs
NAME                         COMPLETIONS   DURATION   AGE
backup-mysql-instance-job    1/1           3s         4m48s
restore-mysql-instance-job   1/1           48s        4m16s
```

# kubernetes-monitoring
### задание: создать кастомный образ nginx, рядом развернуть nginx-exporter для сбора и преобразования метрик для prometheus.
- в оф доке берем параметр для конфига nginx: `https://nginx.org/ru/docs/http/ngx_http_stub_status_module.html`
- этот параметр указывает путь по которому будут доступны метрики сервера
- билдим и заливаем образ на dockerhub
- получается 2 файла в директории `kubernetes-monitoring/build`
- далее пишем `kubernetes-monitoring/deploy/deployment.yaml`
- указываем в поде 2 контейнера, в контейнер nginx-prometheus-exporter передаем перменную окружения `SCRAPE_URI` непосредственно с url откуда метрики собирать.
- пишем сервис для подов `service.yaml`
- деплоим prometheus `helm upgrade --install prometheus prometheus-community/kube-prometheus-stack`
- деплоим ServiceMonitor `kubernetes-monitoring/deploy/servicemonitor.yaml`
- `kubectl port-forward prometheus-operator-grafana-5454fd5fbf-hc4l8 3000`переходим на https://localhost:3000 и настраиваем дашборд с source `http://prometheus-operator-prometheus:9090`
- экспортированный json файл сохранил в `kubernetes-monitoring/grafana/dashboard.json` в этой же директории скриншот.

# kubernetes-logging
- разворачиваем новый кластер в google-cloud
    - default-pool - c одной нодой
    - infra-pool - с тремя нодами
    - навешиваем taint на default-pool `node-role=infra:NoSchedule`
- деплоим hipster-shop:
    - `kubectl create ns microservices-demo`
    - `kubectl apply -f https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-02/Logging/microservices-demo-without-resources.yaml -n microservices-demo`
- проверяем что все поды поднялись на ноде default-pool `k -n microservices-demo get po -o wide`
    - видим что не поднялся pod adservice:v0.1.3 (нет такого образа)
    - смотрим по ссылке, поднимаем версию до v.0.3.4
    - пробрасываем порт до пода с frontend и проверяем что он работает как положено.
- ставим ElasticSearch, Fluent Bit, Kibana без какой-либо преднастройки
    - `helm repo add elastic https://helm.elastic.co`
    - `kubectl create ns observability`
    - `helm upgrade --install elasticsearch elastic/elasticsearch --namespace observability`
    - `helm upgrade --install kibana elastic/kibana --namespace observability`
    - `helm upgrade --install fluent-bit stable/fluent-bit --namespace observability`
    - обнаруживаем что все сервисы пытаются запуститься на ноде из default-pool
    - пишем values для elasticserch `kubernetes-logging/elasticsearch.values.yaml`
        - происываем туда toleration на наш taint `node-role=infra:NoSchedule` и прописываем `nodeSelector` чтобы он запускался только на нодах пула infra (можно использовать `nodeAffinity` для более гибкой настройки)
    - переустанавливаем с новыми values `helm upgrade --install elasticsearch elastic/elasticsearch --namespace observability -f kubernetes-logging/elasticsearch.values.yaml`
- деплоим nginx ingress
    - `helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx`
    - `helm upgrade --install nginx-ingress ingress-nginx/ingress-nginx --create-namespace --namespace nginx-ingress --values kubernetes-logging/nginx-ingress.values.yaml`
- делаем доступ для kibana через ingress
    - `helm -n observability upgrade --install kibana elastic/kibana -f kubernetes-logging/kibana.values.yaml`
- удостоверяемся что все работает:
    - проверяем логи fluent-bit: `k -n observability logs fluent-bit-nc8xn --tail 3`
'''
    [2022/02/24 09:13:02] [ warn] net_tcp_fd_connect: getaddrinfo(host='fluentd'): Name or service not known
    [2022/02/24 09:13:02] [error] [out_fw] no upstream connections available
'''
    - деплоим c указанием имени сервиса elasticsearch-master `helm upgrade --install fluent-bit stable/fluent-bit --namespace observability --values kubernetes-logging/fluent-bit.values.yaml`
    - удостоверяемся в том что ошибок в логах нет
    - смотрим что в ElasticSearch появились индексы
    - удостоверяемся что в логахз fuentbit тоже нет ошибок
- устанавливаем prometheus-exporter
    - `helm upgrade --install prometheus-operator  prometheus-community/prometheus-operator --namespace=observability --values kubernetes-logging/prometheus.values.yaml`
    - `helm upgrade --install elasticsearch-exporter stable/elasticsearch-exporter --set es.uri=http://elasticsearch-master:9200 --set serviceMonitor.enabled=true --namespace=observability`
- настройки grafana
    - входим в графану через `k -n observability port-forward prometheus-operator-grafana-6b9b46b6c-fdtsp 3000` или по url `grafana.34.121.149.124.nip.io`
    - логин выполняем по паролю `k -n observability get secrets prometheus-operator-grafana -o jsonpath={.data.admin-password} | base64 --decode`
    - импортируем дашборд `https://grafana.com/grafana/dashboards/4358`
- тестируем механизм:
    - делаем drain node `k drain gke-cluster-2-infra-pool-96e38e8c-6j78 --ignore-daemonsets`
    - проверяем и видим что в дашборде отобразилось 2 нодыи при это кластер остался полностью работоспособен
    - пробуем задрейнить еще одну ноду `k drain gke-cluster-2-infra-pool-96e38e8c-ktb0 --ignore-daemonsets --delete-emptydir-data`
    - видим сообщение об ошибке `error when evicting pods/"elasticsearch-master-2" -n "observability" (will retry after 5s): Cannot evict pod as it would violate the pod's disruption budget.`
    - `operator-prometheus-0 и elasticsearch-master-1` в статусе `Pending`
    - kibana потеряла подключение к кластеру
    - Метрики Prometheus перестали собираться, так как у сервиса, к которому подключается exporter, пропали все endpoint
- приходим к выводу, что чтобы не потерять мониторинг кластера - нужно знать о выходе из строя нод (желательно на этапе выхода из  строя первой ноды)
    - для решения подобных проблем можно добавить алерт `ElasticsearchTooFewNodesRunning` из источника: `https://github.com/prometheus-community/elasticsearch_exporter/blob/master/examples/prometheus/elasticsearch.rules`

```
ALERT ElasticsearchTooFewNodesRunning
  IF elasticsearch_cluster_health_number_of_nodes < 3
  FOR 5m
  LABELS {severity="critical"}
  ANNOTATIONS {description="There are only {{$value}} < 3 ElasticSearch nodes running", summary="ElasticSearch running on less than 3 nodes"}
```

- делаем uncordon нод
- следует обратить внимание на метрики
    - `unassigned_shards` - количество shard, для которых не нашлось подходящей ноды, их наличие сигнализирует о проблемах
    - `jvm_memory_usage` - высокая загрузка (в процентах от выделенной памяти) может привести к замедлению работы кластера
    - `number_of_pending_tasks` - количество задач, ожидающих выполнения. Значение метрики, отличное от нуля, может сигнализировать о наличии проблем внутри кластера
    - хорошая статья о метриках: `https://habr.com/ru/company/yoomoney/blog/358550/`
- решаем проблему с логами nginx-ingress
    - докидываем парсер для fluent-bitи снова деплоим его в кластер
- решаем сложность анализа логов nginx-ingress в kibana.
    - исходя из документации `https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#log-format-escape-json` и `https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#log-format-upstream` докидываем конфиг в values:

```

  config:
    name: nginx-config

    log-format-escape-json: "true"
    log-format-upstream: '{"remote_addr": "$proxy_protocol_addr", "x-forward-for": "$proxy_add_x_forwarded_for", "request_id": "$req_id", "remote_user": "$remote_user", "bytes_sent": $bytes_sent, "request_time": $request_time, "status":$status, "vhost": "$host", "request_proto": "$server_protocol", "path": "$uri", "request_query": "$args", "request_length": $request_length, "duration": $request_time,"method": "$request_method", "http_referrer": "$http_referer", "http_user_agent": "$http_user_agent" }'
```
    - деплоим еще раз хелмом. поды не требуют рестарта, так как nginx-ingress-controller сам перечитывает конфиги благодаря аргументу, указанному в команде запуска контейнера: `--configmap=$(POD_NAMESPACE)/nginx-ingress-ingress-nginx-controller`
- создаем новую визуализацию с типом TSVB
    - настроим ее для отображения количества запросов к nginx-ingress, выставив фильтр `kubernetes.labels.app_kubernetes_io/instance: nginx-ingress`
    - сделаем дашборд для отображения запросов к nginx-ingress со статусами:
        - 200-299
        - 300-399
        - 400-499
        - 500+
    - выгрузил в `kubernetes-logging/export.ndjson`
- деплоим loki
    - удаляем все с ns
    - `helm repo add grafana https://grafana.github.io/helm-charts`
    - `helm repo update`
    - докидываем в values prometheus `additionalDataSources`
    - деплоим loki
    - `helm upgrade --install loki grafana/loki-stack -n observability --create-namespace  --values values.loki.yaml`
    - смотрим логи nginx ingress с source Loki,через `Explore`
        - `{app_kubernetes_io_name="ingress-nginx"}`
    - создаем дашборд, в котором одновременно выведем метрики ingress-nginx и его логи
        - убеждаемся что вместе с ingress-nginx устанавливается serviceMonitor и prometheus его видит
        - добавляем дашборд и закидываем в него переменные из официального источника `https://github.com/kubernetes/ingress-nginx/blob/master/deploy/grafana/dashboards/nginx.json`
        - сосдаем новую панель и добавляем query для ingress-nginx

```
sum(rate(nginx_ingress_controller_requests{controller_pod=~"$controller",controller_class=~"$controller_class",namespace=~"$namespace",ingress=~"$ingress",status!~"[4-5].*"}[1m])) by (ingress) / sum(rate(nginx_ingress_controller_requests{controller_pod=~"$controller",controller_class=~"$controller_class",namespace=~"$namespace",ingress=~"$ingress"}[1m])) by(ingress)
```

        - аналогичным образом добавляем панель с количеством запросов к ingress-nginx в секунду
        - добаляем панель с логами
        - выгружаем в `ingress-nginx.json`
