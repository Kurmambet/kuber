ReplicaSet не поддерживает обновление, а deployment - обертка над ReplicaSet, которая позволяет обновлять запущеные поды

```bash
kubectl apply -f 03-deployment/web-deployment.yaml

# для проверки
kubectl get deploy
kubectl get rs
kubectl get pods
```

## Удалим 1 под

```bash
# ReplicaSet controller восстановил
kubectl delete pod web-5d4b5ccb95-5brdt
```

Rolling Update постепенно (не сразу убивает все поды) вводит новый манифест, создается новый ReplicaSet и в него перетикают поды из старого по шагам.

- maxUnavailable - сколько подов может быть недоступно во время обновления.
- maxSurge - сколько можно создать лишних реплик сверх replicas в моменте.

```bash
kubectl scale deployment web --replicas=10
kubectl get pods -l app=web
```

Применим новый манифест:

```bash
kubectl apply -f 03-deployment/web-deployment-v126.yaml
```

```bash
kubectl rollout status deployment/web

kubectl rollout history deployment/web

# откатиться к прошлой версии
kubectl rollout undo deployment/web


# версию nginx можно посмотреть так:
kubectl describe pod web-77b9b4cbb4-fhl5m | grep nginx:
```

### cordon

```bash
# нода не будет принимать больше новые поды
kubectl cordon node3

kubectl get nodes
# у ноды3 теперь SchedulingDisabled статус

# и обратно
kubectl uncordon node3
```

### pod disruption budgets

```bash
kubectl get deploy


kubectl delete pdb web-pdb

# pod disruption budgets - сколько минимум подов работает
kubectl create pdb web-pdb --selector='app=web' --min-available=2

kubectl describe pdb web-pdb

kubectl get pdb
```

### извлечь все поды с ноды и отправит их планировщику-шедуллеру, чтобы тот перенаправил их на другие ноды

```bash
kubectl drain node3
kubectl drain node3 --ignore-daemonsets --delete-emptydir-data
```
