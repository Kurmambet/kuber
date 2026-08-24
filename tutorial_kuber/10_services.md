с помощью сервисов организуется стабильная точка доступа (статичный IP) к группе подов

сервис дает постоянное имя и виртуальный IP а трафик раскидывает по живым

сервис работает через kubeproxy

Типы:

- `Cluster IP` - работает только внутри кластера. 1 сервис ходит в другой сервис по имени. создает DNS запись типа svc cluster local
- `NodePort` - открывает порт на каждой ноде и делает эти ноды доступными извне. обычно поверх этого идет балансировзие или ingress.
- `LoadBalancer` - выдаем внешний ip. В облаке по умолчанию. чтобы это работало в on prem кластере надо чтото вроде Metal LB
- `Headless` - нужен когда у нас 1 виртуальный ip, а нам надо получить адреса конкретных подом. часто надо для StatefulSet
- `ExternalName` - не ведет к подам, созхдает dns cname на внешний сервис

kubeproxy - работает с сервисами. Дает виртуальный ip для группы подов через iptables/ipvs.

cni - делает сетевую доступность между подами, выдает им ip, чтобы траффик доходил от 1 пода к другому.

у сервиса есть селектор по имени/label подов

# ClusterIP:

```bash
kubectl apply -f 03-deployment/web-deployment.yaml
kubectl get deploy

kubectl apply -f 04-service/web-clusterip.yaml
# будет выбиать по селектору app:web, а у деплоймента этот селектор labels: app: web


kubectl get svc
# kubectl delete svc web-stateful

kubectl get endpoints  # deprecated

kubectl get endpointslices

kubectl port-forward svc/web 8080:80

curl localhost:8080


kubectl describe svc web

# временный контейнер дл dns
kubectl run dns-test --rm -it --image=busybox:1.36 --restart=Never -- sh
# внутри
nslookup web
# 10.103.175.112 резолвится в наш виртуальный ip
nslookup web.default.svc.cluster.local
```

ExternalTrafficPolisy - есть 2 парамера:

- cluster (по умолчанию) - траффик на любую ноду балансируется на под на любой ноде
- local - траффикк идет только на под, на той же ноде, которая приняла пакет. если на ней нет подов, то дропаются пакеты. зато source ip клиента сохраняется и нет лишнего хопа

# NodePort:

```bash
kubectl apply -f web-nodeport.yaml
kubectl get svc web-nodeport
minikube service web-nodeport --url
```

# Broken selector:

```bash
kubectl apply -f web-service-broken-selector.yaml
kubectl get endpoints
kubectl apply -f web-clusterip.yaml
kubectl get endpoints
```
