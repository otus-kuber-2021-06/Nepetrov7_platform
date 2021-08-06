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
