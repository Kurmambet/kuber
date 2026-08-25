## раскатим приложение

```bash
kubectl apply -f app/k8s/deployment.yaml  # не сможет скачать mage из docker registry

```

Хороший улов — вы правы, автор курса ориентировался на кластер с Docker/containerd, а у вас CRI-O, поэтому команда `podman load` из варианта B тоже не сработает "из коробки" — `podman` у вас не установлен. Разберём рабочий путь именно под вашу связку CRI-O + Docker только на хосте.

## Почему вариант A (через registry) — правильный выбор для вас

Вариант A из инструкции создан именно для случая "нет общего docker daemon между машиной со сборкой и кластером" — то есть ровно ваша ситуация. Просто вместо публичного Docker Hub поднимем **свой** локальный registry на хосте (благо Docker Engine там есть) — это даже ближе к тому, как это делается в реальном проде, чем вариант B.

## Шаг 1. Поднять локальный registry — на хосте

```bash
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

Это официальный лёгкий образ Docker Registry — простое хранилище образов, слушает порт `5000`. Раз он висит на `-p 5000:5000`, он доступен не только с localhost, но и по IP хоста в host-only сети — `192.168.56.1` (тот адрес, что вы сами задавали при создании `vboxnet0`).

## Шаг 2. Собрать и запушить образ — на хосте

```bash
cd /путь/к/ultimate-devops-guide/app
docker build -t scott-pilgrim-quotes:1.0.0 .
docker tag scott-pilgrim-quotes:1.0.0 192.168.56.1:5000/scott-pilgrim-quotes:1.0.0
docker push 192.168.56.1:5000/scott-pilgrim-quotes:1.0.0
```

Всё верно диагностировали в предыдущем шаге для CRI-O, но забыли про **тот же самый** барьер на стороне самого Docker-клиента/демона на хосте — он тоже по умолчанию требует HTTPS для любого registry, кроме `localhost`. Ошибка `server gave HTTP response to HTTPS client` — это именно демон Docker, а не CRI-O, отказывается пушить по простому HTTP.

## Настроить Docker разрешать этот registry как insecure

**На хосте:**

```bash
sudo mkdir -p /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "insecure-registries": ["192.168.56.1:5000"]
}
EOF
sudo systemctl restart docker
```

Это аналог того, что вы уже делали для CRI-O через `/etc/containers/registries.conf.d/`, только формат конфига другой, потому что Docker и CRI-O — разные движки со своими настройками, несмотря на схожую задачу.

Если `/etc/docker/daemon.json` уже существует и там что-то есть — не перезаписывайте файл целиком, а добавьте ключ `insecure-registries` в существующий JSON вручную (`sudo nano /etc/docker/daemon.json`), чтобы не потерять другие настройки.

## Проверка после перезапуска

```bash
docker push 192.168.56.1:5000/scott-pilgrim-quotes:1.0.0
```

Теперь должно пройти без ошибки — увидите нормальный прогресс загрузки слоёв (`Pushed`).

## Заодно проверим, что registry реально принял образ

```bash
curl http://192.168.56.1:5000/v2/_catalog
```

Должно вернуть `{"repositories":["scott-pilgrim-quotes"]}` — подтверждение, что образ физически лежит в вашем локальном registry и доступен по сети для всех трёх VM.

## Важная деталь на будущее — IP хоста подтверждён

Кстати, ваш `ip a` полезен ещё и тем, что подтвердил: `vboxnet0` на хосте действительно имеет IP `192.168.56.1/24` — то есть все последующие команды с этим адресом (в конфигах CRI-O на нодах, в `deployment.yaml`) корректны, ничего пересчитывать не нужно.

## Шаг 3. Научить CRI-O доверять этому registry — на node1, node2, node3

Проблема: ваш registry работает по обычному HTTP (без TLS-сертификата), а CRI-O по умолчанию требует HTTPS для всех registry, кроме явно разрешённых как "insecure". Настройка идёт через тот же самый файл, что использует `podman`/`buildah` — CRI-O делит с ними общую библиотеку работы с образами. [github](https://github.com/cri-o/cri-o/issues/2193)

**На каждой из трёх нод:**

```bash
sudo mkdir -p /etc/containers/registries.conf.d
cat <<EOF | sudo tee /etc/containers/registries.conf.d/local-registry.conf
[[registry]]
location = "192.168.56.1:5000"
insecure = true
EOF
sudo systemctl restart crio
```

`registries.conf.d` — это "drop-in" директория: можно добавлять отдельные `.conf`-файлы, не трогая основной `/etc/containers/registries.conf`, что удобнее и безопаснее правки общего файла. [mikroways](https://www.mikroways.net/blog/2021-12-29-crio-with-harbor/)

Проверка на каждой ноде:

```bash
sudo crictl pull 192.168.56.1:5000/scott-pilgrim-quotes:1.0.0
```

Если команда отработала без ошибки TLS — CRI-O теперь умеет тянуть образы из вашего registry.

## Шаг 4. Поправить манифест и задеплоить — с хоста

В `deployment.yaml` замените:

```yaml
image: scott-pilgrim-quotes:1.0.0
```

на:

```yaml
image: 192.168.56.1:5000/scott-pilgrim-quotes:1.0.0
```

Дальше — ровно по инструкции курса:

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service-clusterip.yaml
kubectl get pods -l app=scott-pilgrim -w
```

проверка clusterip:

```bash
kubectl run tmp --rm -it --image=curlimages/curl --restart=Never -- \
scott-clusterip.default.svc.cluster.local/quote
```

## Если хочется точно повторить вариант B из курса (через podman)

Технически это тоже возможно — CRI-O делит общее хранилище образов именно с `podman` (не с Docker!), потому что оба используют библиотеку `containers/storage`. Пришлось бы поставить `podman` на каждую ноду: [github](https://github.com/cri-o/cri-o/issues/5696)

```bash
sudo apt-get install -y podman
```

А дальше — ровно команды из инструкции курса (`docker save` на хосте → `scp` на ноду → `podman load` на ноде). Но вариант с registry (шаги 1–4 выше) удобнее для вас на будущее — при каждом обновлении образа не нужно гонять `.tar`-файлы по трём нодам вручную, достаточно `docker push` один раз, а `crictl pull`/kubelet сам подтянет новую версию на нужную ноду.

Какой вариант берёте — через registry или через podman? Помогу довести до рабочего пода.

```




```

## вариант 1: запустить с хоста

Раз вы уже настроили `kubectl` на хосте с копией `admin.conf` (мы это делали пару шагов назад) — просто выполните ту же команду **на хосте**, а не на node1:

```bash
kubectl port-forward svc/scott-clusterip 8080:80
```

`kubectl port-forward` не требует прямой сетевой связности с подом — команда открывает туннель через сам API-сервер (`192.168.56.101:6443`), а он уже сам проксирует трафик к поду внутри кластера. Раз хост умеет достучаться до API-сервера (что вы уже проверяли — `kubectl get nodes` работает с хоста), значит и port-forward сработает точно так же с хоста.

Проверка (тоже с хоста):

```bash
curl localhost:8080/quote
```

## Вариант 2: если хотите оставить port-forward именно на node1

Тогда нужно явно указать, что слушать нужно не только `127.0.0.1`, а все интерфейсы ноды:

```bash
kubectl port-forward svc/scott-clusterip --address 0.0.0.0 8080:80
```

И обращаться с хоста уже по host-only IP node1, а не `localhost`:

```bash
curl http://192.168.56.101:8080/quote
```

Но это менее удобно и без явной причины не нужно — вариант 1 проще.

## Вариант 3 (ближе к тому, что показано в курсе для "внешнего" доступа) — NodePort

Раз у вас есть реальные VM с реальными IP (а не minikube), логичнее по инструкции курса попробовать именно `service-nodeport.yaml` — тогда порт будет открыт **постоянно** на каждой ноде, без необходимости держать `kubectl port-forward` в отдельном терминале:

```bash
kubectl apply -f k8s/service-nodeport.yaml
kubectl get svc
```

Дальше с хоста (или откуда угодно в host-only сети):

```bash
curl http://192.168.56.101:30080/quote
curl http://192.168.56.103:30080/quote
```

Обратите внимание — NodePort открывает порт **на всех** нодах кластера одновременно (даже на той, где физически нет пода) — это одна из ключевых особенностей, которую стоит явно увидеть: postgres-запрос на node3 (где реально сидит под) и на node1 (где пода нет) должен вернуть одинаковый ответ, потому что kube-proxy сам перенаправит трафик куда нужно.

После долгого простоя виртуалок у них сбивается время.

```bash
# на каждой ноде
# sudo apt-get install -y chrony
sudo systemctl enable --now chrony
sudo chronyc makestep
date
timedatectl status

# с хоста
kubectl get pods -n kube-system | grep calico
# если не Running, то
kubectl delete pods -n kube-system -l k8s-app=calico-node
```

```






```

внешнее имя вместо ip

```bash
kubectl apply -f k8s/service-externalname.yaml
```

проверка :

```bash
kubectl run tmp --rm -it --image=curlimages/curl --restart=Never -- \
scott-externalname.default.svc.cluster.local
```

## Что реально означает ExternalName

Напомню механику: `ExternalName` — это **не проксирование трафика**, а просто DNS-трюк. Когда под спрашивает `scott-externalname.default.svc.cluster.local`, CoreDNS отвечает CNAME-записью `ya.ru` — и всё, дальше под сам напрямую резолвит и подключается к `ya.ru`, без какого-либо участия Kubernetes в самом соединении. Значит, `curl: (52) Empty reply from server` — это уже проблема на уровне "под ↔ реальный интернет", а не "под ↔ Service".

## Наиболее вероятная причина — IPv6

`ya.ru` отдаёт и IPv4, и IPv6-адреса. Современный `curl` по умолчанию предпочитает IPv6, если он доступен. А у вас в конфиге Calico явно стоит:

```
FELIX_IPV6SUPPORT: false
```

Вы это видели ещё в самом первом `kubectl describe pod calico-node`. Это значит, что IPv6-трафику из пода банально некуда маршрутизироваться наружу — TCP-соединение по IPv6 может "как бы" установиться на уровне локального стека, но реально не долетать до `ya.ru`, обрываясь без данных — отсюда и "пустой ответ".

## Диагностика

**С хоста:**

```bash
kubectl run tmp --rm -it --image=curlimages/curl --restart=Never -- curl -v -4 http://scott-externalname.default.svc.cluster.local
```

Разбор флагов:

- `-v` — подробный вывод, покажет, по какому именно IP и протоколу curl пытался подключиться;
- `-4` — принудительно заставляет использовать IPv4, исключая IPv6 из уравнения.

Если с `-4` всё сразу заработает (увидите `HTTP/1.1 302 Found` — `ya.ru` действительно всегда редиректит с `http://` на `https://`, это нормальное поведение сайта) — гипотеза про IPv6 подтверждена, и это особенность вашей текущей сетевой настройки Calico, а не баг Kubernetes.

## Дополнительно — добавить `-L`, чтобы куда следовать за редиректом

```bash
kubectl run tmp --rm -it --image=curlimages/curl --restart=Never -- curl -v -4 -L http://scott-externalname.default.svc.cluster.local
```

`-L` заставит curl автоматически перейти по `302 → https://ya.ru` и показать реальную HTML-страницу — так вы увидите полноценный успешный ответ, а не только заголовок редиректа.

## Итог для понимания ExternalName в целом

Если тест с `-4` пройдёт успешно — это доказывает, что сам паттерн `ExternalName` работает правильно (полезно, например, чтобы дать подам внутри кластера удобное короткое DNS-имя для внешней БД или стороннего API — типа `internal-db` вместо длинного `mydb.us-east-1.rds.amazonaws.com`). Проблема, которую вы поймали, — это побочный эффект того, что ваш кластер сознательно настроен только на IPv4 (тот самый выбор, который вы делали, изучая режимы Calico), и никак не связана с корректностью самого `Service` типа `ExternalName`.
