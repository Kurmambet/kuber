# Runbook: под завис в Pending / ContainerCreating / ImagePullBackOff

Универсальная инструкция на основе реальных инцидентов (Calico, curl, ingress-nginx).
Все команды с хоста, если не указано иное.

## 1. Диагностика — прежде чем что-либо чинить

```bash
kubectl get pods -A -o wide                     # общий обзор, ищем не-Running
kubectl describe pod <имя> -n <namespace>        # СНАЧАЛА смотрим блок Events внизу
```

Что искать в Events:
- `Pulling image "..."` без последующего `Pulled` за разумное время (>2 мин) → завис pull
- `FailedMount ... secret "X" not found` → под ждёт ресурс, которого нет (например, вебхук-сертификат)
- `Insufficient cpu/memory` → нода не может выделить ресурсы
- `Back-off restarting failed container` → контейнер падает после старта, не проблема pull

## 2. Если завис именно pull образа

Узнать на какую ноду шедулер поставил под:
```bash
kubectl get pod <имя> -n <namespace> -o wide     # колонка NODE
```

Проверить время на этой ноде в первую очередь (частая скрытая причина — TLS-ошибка
из-за рассинхрона часов, а не реальная проблема сети):
```bash
ssh <node> "date && timedatectl status"
```
Если время неверное — см. раздел 5.

Проверить сам интернет с ноды:
```bash
ssh <node> "ping -c 3 8.8.8.8 && curl -Is https://registry-1.docker.io | head -5"
```

Ключевой шаг — вытянуть образ вручную через crictl (синхронно, с реальной ошибкой,
в отличие от вывода kubelet, который просто пишет "Pulling" и молчит):
```bash
ssh <node> "sudo crictl pull <точный_образ_с_тегом_или_digest>"
```
Если команда сама зависла — Ctrl+C и повторить 2-3 раза (частый обрыв соединения
именно на больших образах, не обязательно блокировка).

Если crictl не установлен на ноде:
```bash
VERSION="v1.34.0"
ssh <node> "curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/\$VERSION/crictl-\$VERSION-linux-amd64.tar.gz | sudo tar -C /usr/local/bin -xz && sudo crictl config runtime-endpoint unix:///var/run/crio/crio.sock"
```

После успешного ручного pull — пересоздать конкретный под, чтобы он подхватил
уже закэшированный образ:
```bash
kubectl delete pod <имя> -n <namespace>
```

## 3. Если завис именно FailedMount / отсутствующий Secret (например, ingress-nginx-admission)

```bash
kubectl get jobs -n <namespace>                  # Job'ы, которые должны создать секрет
kubectl get pods -n <namespace> -l job-name=<имя_job>
kubectl logs -n <namespace> -l job-name=<имя_job>
kubectl get secret <имя_секрета> -n <namespace>  # появился ли секрет
```
Если Job сам застрял на pull — см. раздел 2 для образа этого Job'а.
Как только секрет появился — под с FailedMount обычно сам переходит в Running
в течение минуты; если нет:
```bash
kubectl delete pod <имя_пода> -n <namespace>
```

## 4. Принудительная зачистка зависших объектов (Terminating дольше 2-3 минут)

Обычное удаление пода:
```bash
kubectl delete pod <ИМЯ> -n <NAMESPACE> --grace-period=0 --force
```
(имя пода и `-n namespace` — ОТДЕЛЬНЫЕ аргументы, не путать порядок)

Если под не убирается даже так — снять finalizer напрямую:
```bash
kubectl patch pod <ИМЯ> -n <NAMESPACE> -p '{"metadata":{"finalizers":[]}}' --type=merge
```

Если завис целый Namespace в Terminating:
```bash
kubectl get namespace <NAMESPACE> -o json | jq '.status'   # причина в conditions
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -n 1 kubectl get --show-kind --ignore-not-found -n <NAMESPACE>  # что реально осталось
```
Хирургическое снятие finalizer у namespace (когда обычный delete не работает):
```bash
kubectl get namespace <NAMESPACE> -o json | jq '.spec.finalizers = []' > ns-no-fin.json
kubectl replace --raw "/api/v1/namespaces/<NAMESPACE>/finalize" -f ns-no-fin.json
```
⚠️ После этого приёма namespace исчезает из API, но объекты внутри него могут
остаться "осиротевшими" в etcd — их придётся удалять вручную по одному:
```bash
kubectl delete pod <имя> -n <NAMESPACE> --grace-period=0 --force --ignore-not-found
kubectl delete service <имя> -n <NAMESPACE> --ignore-not-found
kubectl delete deployment <имя> -n <NAMESPACE> --ignore-not-found
kubectl delete replicaset --all -n <NAMESPACE> --ignore-not-found
kubectl get all -n <NAMESPACE>                    # должно быть пусто
```

Не забыть cluster-scoped объекты, которые НЕ удаляются вместе с namespace
(например, вебхуки — иначе они продолжат ловить запросы в пустоту):
```bash
kubectl get validatingwebhookconfigurations
kubectl delete validatingwebhookconfigurations <имя>
kubectl get mutatingwebhookconfigurations
kubectl delete mutatingwebhookconfigurations <имя>
```

Проверить "мёртвые" контейнеры прямо в CRI-O на ноде (иногда k8s думает что пода
нет, а контейнер физически жив):
```bash
ssh <node> "sudo crictl ps -a | grep <имя_приложения>"
ssh <node> "sudo crictl rm -f <container_id>"
```

## 5. Рассинхрон времени после долгого простоя VM (делать при каждом старте лабы)

```bash
# на КАЖДОЙ ноде (node1, node2, node3):
sudo systemctl enable --now chrony
sudo chronyc makestep
date
timedatectl status                                # ок, если время верное, даже если
                                                   # "System clock synchronized: no" —
                                                   # это просто ещё не устоялось
```
Если после скачка времени сеть Calico начала выдавать `Unauthorized`
(`error getting ClusterInformation: connection is unauthorized`):
```bash
kubectl delete pods -n kube-system -l k8s-app=calico-node
kubectl get pods -n kube-system -o wide -w        # ждём все calico-node снова Running
```

## 6. Полная переустановка ingress-nginx с нуля (когда проще снести, чем чинить)

```bash
# 1. Убедиться, что тяжёлый образ уже в кэше на нужной ноде — тянем вручную заранее
ssh node3 "sudo crictl pull registry.k8s.io/ingress-nginx/controller:<тег_или_digest>"

# 2. Снести всё
kubectl delete namespace ingress-nginx
kubectl get namespace ingress-nginx -w            # дождаться NotFound (или см. раздел 4,
                                                   # если зависло)
kubectl delete validatingwebhookconfigurations ingress-nginx-admission --ignore-not-found

# 3. Поставить заново
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.13.0/deploy/static/provider/baremetal/deploy.yaml
kubectl get pods -n ingress-nginx -w              # НЕ прерывать Ctrl+C раньше времени!
kubectl -n ingress-nginx rollout status deploy/ingress-nginx-controller

# 4. (опционально) Отдать контроллеру реальный сетевой стек ноды,
#    чтобы порты 80/443 были доступны напрямую по IP ноды, без NodePort-обёртки:
kubectl -n ingress-nginx patch deploy ingress-nginx-controller \
  -p '{"spec":{"template":{"spec":{"hostNetwork":true,"dnsPolicy":"ClusterFirstWithHostNet"}}}}'
kubectl -n ingress-nginx rollout status deploy/ingress-nginx-controller

# 5. Проверка
kubectl get ingressclass
kubectl get pods -n ingress-nginx -o wide
```

## Золотое правило по итогам этой сессии

Прежде чем разбираться со сложными симптомами (webhook, RBAC, сеть) — **всегда
сначала** проверить: 1) время на нодах (раздел 5), 2) реально ли долетел образ
через `crictl pull` вручную (раздел 2). В 80% инцидентов этого чата причина
сводилась именно к этим двум пунктам.
