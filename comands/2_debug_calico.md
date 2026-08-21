`node1` всё ещё `NotReady`, и виден корень проблемы: под `calico-node-6hg7t` именно на **node1** завис на `Init:0/3` — то есть даже первый из трёх init-контейнеров Calico не смог отработать. На node2/node3 те же поды прошли нормально (`1/1 Running`), значит проблема локальная для node1, не в манифесте Calico.

## Диагностика — выполните на node1

```bash
kubectl describe pod calico-node-6hg7t -n kube-system
```

Смотрите в самый низ, в блок `Events` — там будет конкретная причина (например, `FailedMount`, `ImagePullBackOff`, `CrashLoopBackOff` у init-контейнера, или ошибка в `Warning`).

```bash
kubectl logs calico-node-6hg7t -n kube-system -c upgrade-ipam
kubectl logs calico-node-6hg7t -n kube-system -c install-cni
```

Это два первых init-контейнера Calico (`upgrade-ipam`, потом `install-cni`, потом `mount-bpffs`) — узнаем, на каком именно из них затык, и что он пишет в лог.

```bash
sudo systemctl status kubelet -l
sudo journalctl -u kubelet -n 80 --no-pager
```

Раз нода в целом `NotReady`, стоит проверить и сам kubelet на node1 — возможно, у него ещё остались последствия старого `kubeadm reset`/переинициализации (например, недочищенные файлы CNI-конфигурации в `/etc/cni/net.d/` с прошлой попытки).

```bash
ls -la /etc/cni/net.d/
sudo crictl ps -a | grep calico
```

Проверит, не осталось ли на node1 "мусора" от старой Calico-установки — конфликтующий файл CNI-конфига может блокировать `install-cni` init-контейнер, который как раз пытается записать новый.

Пришлите вывод этих команд (особенно `describe pod` и `journalctl -u kubelet`) — там почти наверняка будет явная строка с ошибкой, и я скажу точное исправление, а не буду гадать вслепую.

```




```

```





```

Нашли зацепку: событие `Pulling image "docker.io/calico/cni:v3.28.0"` было **11 минут назад**, и с тех пор — тишина. Ни `Pulled`, ни `Failed`. Это значит, что скачивание образа реально зависло именно на node1 — при этом на node2/node3 всё скачалось нормально. Значит, дело не в Calico и не в CNI-манифесте, а в сети/CRI-O конкретно на node1.

## Проверяем интернет и CRI-O на node1

```bash
ping -c 3 8.8.8.8
curl -Is https://registry-1.docker.io | head -5
cat /etc/resolv.conf
```

Первая команда проверит, есть ли вообще связность с интернетом (через NAT-адаптер `enp0s3`). Вторая — доходит ли конкретно до Docker Hub. Третья — не сломался ли DNS (если `nameserver` пустой или недоступный — apt/curl вроде бы у вас раньше работали, но за это время что-то могло измениться).

```bash
sudo systemctl status crio -l
sudo journalctl -u crio -n 100 --no-pager | tail -60
```

Смотрим, не упал ли сам CRI-O или не завис ли на попытке подключиться к registry — там обычно видна конкретная ошибка (`context deadline exceeded`, `TLS handshake timeout`, `no route to host` и т.д.).

## Поставить crictl (у вас его нет)

`crictl` не входит в пакет `cri-o`, ставится отдельно — пригодится для диагностики "снаружи" kubelet:

```bash
VERSION="v1.34.0"
curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz | sudo tar -C /usr/local/bin -xz
sudo crictl config runtime-endpoint unix:///var/run/crio/crio.sock
```

Дальше можно вручную проверить, реально ли CRI-O умеет тянуть образ:

```bash
sudo crictl pull docker.io/calico/cni:v3.28.0
```

Если эта команда тоже зависнет — проблема на 100% в сети/CRI-O на node1, а не в Kubernetes/Calico. Если команда сразу отработает — проблема была временной (например, просто перегруз при первой попытке), и достаточно будет пересоздать под.

## Если после диагностики окажется, что дело просто зависло

Можно принудительно пересоздать конкретный под, не трогая остальной кластер:

```bash
kubectl delete pod calico-node-6hg7t -n kube-system
```

DaemonSet автоматически создаст новый под на node1 — этого часто достаточно, если первая попытка скачивания просто "залипла" на старте CRI-O.

Пришлите вывод `ping`, `curl`, и особенно `journalctl -u crio` — там почти наверняка будет явная причина, чтобы не гадать вслепую.

```














```

Отлично — `crictl pull` отработал мгновенно (образ уже подтянулся, видимо, "зависшая" попытка CRI-O в итоге доскачала его в фоне, просто событие `Pulled` не долетело до `kubectl describe`). Раз вы уже удалили старый под — DaemonSet сейчас пересоздаст его на node1, и в этот раз образ уже локально закэширован, значит должно пройти быстро.

## Проверяем результат

```bash
kubectl get pods -n kube-system -o wide -w
```

Флаг `-w` (watch) покажет изменения статуса в реальном времени — не нужно гонять команду руками. Ждите, пока новый `calico-node-xxxxx` на node1 пройдёт через `Init:1/3` → `Init:2/3` → `Init:3/3` → `PodInitializing` → `Running`. `Ctrl+C`, когда увидите `1/1 Running`.

```bash
kubectl get nodes
```

После этого node1 должен перейти в `Ready` — Calico на всех трёх нодах наконец запущен.

## Финальная проверка всего кластера

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

Ожидаемый итог: все три ноды `Ready`, все поды в `kube-system` (calico-node×3, calico-kube-controllers, coredns×2, etcd×2, kube-apiserver×2, kube-controller-manager×2, kube-scheduler×2, kube-proxy×3) в статусе `Running`.

## Тест "живого" кластера

Когда всё зелёное, стоит запустить реальный тестовый под, чтобы убедиться, что поды действительно получают сеть и планировщик их распределяет по нодам:

```bash
kubectl run nginx-test --image=nginx --restart=Never
kubectl get pod nginx-test -o wide
kubectl exec nginx-test -- curl -s -o /dev/null -w "%{http_code}\n" localhost
```

Если получите `200` и увидите, что под получил IP из диапазона `10.244.x.x` — кластер полностью рабочий, можно приступать к следующим задачам (SSH-ключ для Ansible/Kubespray, деплой приложений и т.д.).

Пришлите вывод `kubectl get nodes` и `kubectl get pods -A`, когда watch остановится — зафиксируем итоговое состояние кластера.
