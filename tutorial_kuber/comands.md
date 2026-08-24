# Docker

```bash
sudo usermod -aG docker $USER
newgrp docker

docker rm -f $(docker ps -aq)      # удалить ВСЕ контейнеры (и запущенные, и остановленные)
docker rmi image:tag
docker container prune      # удалить ВСЕ остановленные контейнеры
docker image prune -a       # удалить ВСЕ образы, не используемые ни одним контейнером
docker system prune -a --volumes   # полная зачистка: контейнеры, образы, сети, volumes
docker ps -a                # посмотреть все контейнеры (включая остановленные)
docker images                # посмотреть все образы
```

## Где вообще выполнять kubectl

`kubectl` — это просто клиент, который читает файл `~/.kube/config` и по нему стучится в API-сервер по сети (в вашем случае `192.168.56.101:6443`). Это значит, что **не обязательно** заходить по SSH на node1 каждый раз. Проще скопировать конфиг к себе на хост (Linux Mint) и управлять кластером прямо оттуда, без единого SSH-подключения:

```bash
# на node1
cat ~/.kube/config
```

Скопируйте вывод и сохраните на хосте в `~/.kube/config` (или используйте `scp -i ~/.ssh/bm_k8s node1@192.168.56.101:~/.kube/config ~/.kube/config`). После этого на **хосте** просто ставите `kubectl` (`sudo snap install kubectl --classic` или через apt-репозиторий из тех же `pkgs.k8s.io`) — и все команды ниже выполняются с вашей машины Mint, а не изнутри VM.

Единственное, что реально требует захода именно на конкретную ноду по SSH — это системные операции (перезапуск kubelet, просмотр `journalctl`, `crictl`) и, собственно, сам `kubeadm init`/`join`, которые вы уже сделали.

## Сопоставление Docker → kubectl

| Docker                   | kubectl-аналог                                                  | Что делает                                                                                        |
| ------------------------ | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `docker ps`              | `kubectl get pods`                                              | Список запущенных подов (аналог контейнеров)                                                      |
| `docker images`          | `kubectl get pods -o wide` / нет прямого аналога                | В k8s образы не просматриваются централизованно — только через `crictl images` на конкретной ноде |
| `docker run`             | `kubectl run <name> --image=<img>`                              | Быстрый запуск одного пода (для теста, как вы уже делали с nginx)                                 |
| `docker exec -it`        | `kubectl exec -it <pod> -- <cmd>`                               | Зайти внутрь контейнера/пода                                                                      |
| `docker logs -f`         | `kubectl logs -f <pod>`                                         | Смотреть логи в реальном времени                                                                  |
| `docker build`           | нет аналога                                                     | Образы всё ещё собираются отдельно (`docker build`/`buildah`), кластер их только запускает        |
| `docker stop/rm`         | `kubectl delete pod <name>`                                     | Удалить под (но обычно удаляют не под, а весь Deployment — под пересоздастся сам)                 |
| `docker-compose up -d`   | `kubectl apply -f manifest.yaml`                                | Применить YAML-манифест — аналог "поднять всё описанное в файле"                                  |
| `docker-compose down`    | `kubectl delete -f manifest.yaml`                               | Убрать всё, что было создано этим манифестом                                                      |
| `docker-compose down -v` | `kubectl delete -f manifest.yaml` + `kubectl delete pvc <name>` | В k8s volume'ы (PVC) не удаляются автоматически вместе с ресурсами — их надо чистить отдельно     |
| `docker-compose restart` | `kubectl rollout restart deployment/<name>`                     | Перезапустить все поды деплоймента без изменения манифеста                                        |

## Минимальный набор для старта

### Смотреть, что происходит

```bash
kubectl get nodes                    # статус нод (Ready/NotReady)
kubectl get pods -A                  # все поды во всех неймспейсах
kubectl get pods                     # поды в текущем неймспейсе (default)
kubectl get all -n kube-system       # всё системное хозяйство кластера
kubectl describe pod <имя>           # подробности + события (первое, что смотреть при проблемах)
kubectl logs <имя>                   # логи пода
kubectl logs -f <имя>                # логи в реальном времени, как docker logs -f
kubectl logs <имя> -c <контейнер>    # если в поде несколько контейнеров (как у Calico)
```

### Заходить внутрь и разбираться

```bash
kubectl exec -it <имя> -- bash       # зайти в контейнер, как docker exec -it
kubectl top pod                      # потребление CPU/RAM подами (нужен metrics-server, по умолчанию не стоит)
kubectl top node                     # то же самое по нодам
```

### Запускать и убирать

```bash
kubectl run test --image=nginx --restart=Never   # разовый под для проверки (как вы делали)
kubectl delete pod test                            # удалить его
kubectl apply -f deployment.yaml                   # применить манифест (основной рабочий способ)
kubectl delete -f deployment.yaml                  # удалить то, что создал этот манифест
kubectl delete deployment <имя>                    # удалить конкретный деплоймент вручную
```

### Копать глубже при проблемах (то, чем мы весь вечер занимались)

```bash
kubectl get events --sort-by='.lastTimestamp'      # последние события по всему кластеру
kubectl get pods -o wide                           # + на какой ноде и с каким IP запущен под
kubectl rollout status deployment/<имя>            # дождаться, пока деплоймент выкатится
kubectl rollout restart deployment/<имя>           # аналог docker-compose restart
kubectl rollout undo deployment/<имя>               # откатить последний деплой назад
```

## Важное отличие от Docker в самой философии

В Docker вы обычно управляете **контейнерами напрямую** (`run`, `stop`, `rm`). В Kubernetes вы почти никогда не трогаете поды напрямую — вы описываете **желаемое состояние** в YAML-манифесте (Deployment/Service) и отправляете его через `kubectl apply`, а дальше control-plane (тот самый scheduler/controller-manager, которые вы разворачивали на node1/node2) сам следит, чтобы реальность соответствовала описанию. Удалить под руками (`kubectl delete pod`) — это скорее "перезапустить" его: если под управляется Deployment'ом, он пересоздастся автоматически (вы это только что видели с `calico-node-pb7jr`).

Если хотите, следующим шагом могу показать, как выглядит простой `Deployment` + `Service` манифест — это будет практическим мостиком от «одиночных команд» к «описательному» стилю Kubernetes.

```bash
kubectl cluster-info

ps aux | grep -E 'kube-apiserver|etcd|kube-sheduler|kube-controller-manager'

ps aux | grep kubelet

kubectl describe node # вся инфа о нодах

kubectl get ns  # namespaces

kubectl auth can-i --list  # какие у меня права

kubectl config current-context  # контекст = пользователь + кластер + namespace

kubectl get pods -n kube-system  # поды в namespace/пространстве имен kube-system
```
