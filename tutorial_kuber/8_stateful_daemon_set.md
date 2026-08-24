# StatefulSet:

- надо для stateful нагрузки (бд, kafka, elasticsearch)
- дает стабильные имена (1,2,3) и свой диск (PVC) который не теряется при пересоздании.
- создаются и удаляются по порядку

# StatefulSet practice

```bash
kubectl apply -f 07-statefulset/web-stateful.yaml

kubectl get pods

kubectl get statefulset

kubectl delete pod web-stateful-1
# pod возвращается с тем же стабильным именем.
kubectl get pods -w

kubectl delete -f 07-statefulset/web-stateful.yaml
```

# DaemonSet

- запускает по 1 поду на каждой ноде
- надо для агентов (сбор логов, метрик, сеть)
- kube-proxy, calico - daemonset'ы (Подтверждение: `kubectl get ds -n kube-system`)

anti-afinity - просто разносит поды по нодам, не связано с темой daemonset

```bash
kubectl apply -f 08-daemonset/node-agent.yaml
kubectl get ds

# DaemonSet создаёт pod на каждой подходящей node.
kubectl get pods -o wide
```

```





```

# taints/tolerations

## Taint (изъян/пятно) — метка на **ноде**

Taint — это "отпугиватель" на ноде: "сюда обычные поды не ставить, если только они специально не готовы это терпеть". Именно с этим вы уже работали руками:

```bash
kubectl taint nodes node2 node-role.kubernetes.io/control-plane:NoSchedule-
```

Формат taint — три части: `ключ=значение:эффект`. У вас было `node-role.kubernetes.io/control-plane` (ключ, без значения) `:NoSchedule` (эффект). Смысл: "любой под, который не умеет терпеть этот конкретный taint — не размещать на этой ноде".

## Toleration (терпимость) — метка на **поде**

Toleration — это, наоборот, объявление в манифесте пода: "я умею мириться с таким-то taint'ом, можешь меня сюда ставить". Без соответствующей toleration под просто игнорирует такую ноду при планировании — как будто её не существует.

Вы это уже видели, только не связали с термином — вернитесь к самому первому `kubectl describe pod calico-node-6hg7t`, там был длинный блок:

```
Tolerations:  :NoSchedule op=Exists
              :NoExecute op=Exists
              CriticalAddonsOnly op=Exists
              node.kubernetes.io/disk-pressure:NoSchedule op=Exists
              ...
```

Это ровно то — Calico явно прописал в своём манифесте "я терплю **вообще любой** taint" (`op=Exists` без указания конкретного ключа означает "любой taint с этим эффектом"), поэтому его поды смогли встать даже на node1/node2, которые тогда были ещё с taint'ом control-plane.

## Почему это критично именно для DaemonSet

DaemonSet по определению должен запустить **ровно один под на каждой ноде кластера**, без исключений — так работают `kube-proxy` и `calico-node`, которые нужны абсолютно везде, включая control-plane ноды (без сетевого CNI-агента даже control-plane нода не смогла бы нормально работать). Если бы у DaemonSet'а не было универсальных toleration'ов — он бы попросту "не заметил" node1/node2 из-за их taint'а и запустил поды только на "чистых" worker-нодах (как раз то поведение, о котором вы спрашивали в самом начале вопроса про taints).

## Три эффекта taint — не только `NoSchedule`

| Эффект             | Что делает                                                                                                |
| ------------------ | --------------------------------------------------------------------------------------------------------- |
| `NoSchedule`       | Новые поды без toleration **не размещаются**. Уже работающие — не трогает                                 |
| `PreferNoSchedule` | "Мягкий" вариант — планировщик **старается** избегать ноды, но не запрещает жёстко                        |
| `NoExecute`        | Самый строгий — не только блокирует новые поды, но **выселяет** уже работающие, если у них нет toleration |

Именно `NoExecute` вы тоже видели в списке toleration'ов Calico (`:NoExecute op=Exists`) — это позволяет ему пережить, например, временную недоступность ноды (`node.kubernetes.io/not-ready:NoExecute`), не будучи принудительно убитым.

## Как это пишется в манифесте DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: my-agent
spec:
  selector:
    matchLabels:
      app: my-agent
  template:
    metadata:
      labels:
        app: my-agent
    spec:
      tolerations:
        - key: "node-role.kubernetes.io/control-plane"
          operator: "Exists"
          effect: "NoSchedule"
      containers:
        - name: my-agent
          image: busybox
          command: ["sleep", "infinity"]
```

Без блока `tolerations` этот DaemonSet встал бы **только** на worker-ноды (у вас — только node3, если снова навесите taint на node2). С этим блоком — встанет и на control-plane ноды тоже, ровно как Calico/kube-proxy.

## Проверить на практике прямо сейчас

```bash
kubectl get nodes -o json | jq '.items[] | {name:.metadata.name, taints:.spec.taints}'
```

Покажет актуальные taints на всех трёх нодах — полезно свериться перед тем, как разбираться, почему какой-то под "не хочет" вставать туда, куда вы ожидаете.
