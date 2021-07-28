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
