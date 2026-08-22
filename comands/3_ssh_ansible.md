Кластер живой, тестовый под получил IP из пула Calico (`10.244.135.4`) и ответил `200` на `curl localhost`. Это значит **вся** цепочка (kubeadm → CRI-O → CNI → сеть подов) работает от начала до конца.

## Про `calico-node-pb7jr` — не паникуйте

```bash
kubectl get pods -n kube-system -w
```

Подождите ещё 1–2 минуты, скорее всего он сам дойдёт до `Running` — образ `calico/node` (не `calico/cni`, который вы уже вручную протестировали) весит больше и мог просто ещё качаться в момент вашего снимка. Если через 5 минут он всё ещё будет висеть на `Init:2/3` — пришлите `kubectl describe pod calico-node-pb7jr -n kube-system`, разберём отдельно (это, скорее всего, третий init-контейнер `mount-bpffs`, не критичный для базовой сети — но раз есть шанс подождать, лучше не трогать руками).

## Что такое Ansible и зачем он вам

Ansible — это инструмент **автоматизации**: вместо того чтобы вручную заходить по SSH на node1, node2, node3 и печатать одни и те же команды (как вы делали весь этот разговор — `swapoff`, `apt install cri-o`, правки sysctl и т.д.), вы один раз пишете **playbook** (файл-сценарий на YAML), а Ansible сам подключается по SSH ко всем нодам из списка и выполняет там эти команды параллельно. Это особенно ценно, если завтра вы захотите пересобрать кластер с нуля на новых VM — вместо 40 минут ручного копипаста будет один запуск команды `ansible-playbook`.

Важное уточнение по вашему вопросу: **вам нужен один SSH-ключ (одна пара — приватный + публичный), а не три**. Публичная часть этого одного ключа копируется на все три ноды (в `~/.ssh/authorized_keys` каждой), а приватная остаётся только у вас на хосте. Ansible, подключаясь к любой из трёх нод, использует один и тот же приватный ключ — просто заходит на три разных IP-адреса.

## Зачем это сейчас, если кластер уже поднят вручную

Тут стоит явно решить, для чего вам Ansible в этой задаче — два разных сценария:

- **Kubespray** (тот, что вы упоминали изначально) — это готовый набор Ansible-плейбуков, который сам полностью разворачивает Kubernetes-кластер с нуля (то, что вы только что проделали руками через kubeadm). Если запустить его сейчас на уже настроенных нодах — он либо переустановит поверх, либо сконфликтует с существующим kubeadm-кластером. Имеет смысл, только если вы хотите **снести** текущий кластер и пересобрать его через Kubespray для сравнения подходов.
- **Свой playbook** — вы пишете небольшой Ansible-сценарий для повседневных задач на уже работающем кластере: обновление пакетов на всех трёх нодах разом, деплой манифестов, ротация логов и т.д. Это не конфликтует с текущим кластером, а дополняет его.

Дайте знать, какой вариант интереснее — я подстрою план под него. Пока начнём с общей для обоих сценариев базы: раздача ключа.

## Шаг 1. Сгенерировать ключ — на хосте (Linux Mint)

```bash
mkdir -p ~/labs/kubespray-demo && cd ~/labs/kubespray-demo
ssh-keygen -t ed25519 -f ~/.ssh/bm_k8s -C "boostmentor-k8s" -N ""
```

`-t ed25519` — современный, быстрый и компактный тип ключа (лучше устаревшего RSA). `-f ~/.ssh/bm_k8s` — имя файла, чтобы не путать с вашим основным `id_ed25519` (если он у вас уже используется для GitHub и т.п.). `-N ""` — пустая парольная фраза для ключа, чтобы Ansible мог использовать его без интерактивного ввода пароля при каждом запуске (для лабы это ок; для продакшена так делать не стоит).

## Шаг 2. Скопировать публичный ключ на все три ноды

```bash
ssh-copy-id -i ~/.ssh/bm_k8s.pub node1@192.168.56.101
ssh-copy-id -i ~/.ssh/bm_k8s.pub node2@192.168.56.102
ssh-copy-id -i ~/.ssh/bm_k8s.pub node3@192.168.56.103
```

Каждая команда один раз спросит пароль пользователя на соответствующей ноде (тот, что вы задавали при установке Ubuntu), а дальше допишет ваш публичный ключ в `~/.ssh/authorized_keys` на этой ноде.

## Шаг 3. Проверить вход по ключу без пароля

```bash
ssh -i ~/.ssh/bm_k8s node1@192.168.56.101 hostname
ssh -i ~/.ssh/bm_k8s node2@192.168.56.102 hostname
ssh -i ~/.ssh/bm_k8s node3@192.168.56.103 hostname
```

Ожидаемый вывод — три строки: `node1`, `node2`, `node3`, без единого запроса пароля.

Выполните эти три шага и пришлите результат — дальше поставим Ansible на хост и напишем первый инвентори-файл с вашими тремя нодами, чтобы проверить связь командой `ansible all -m ping`.

```





```

```bash
ssh-keygen -t ed25519 -f ~/.ssh/bm_k8s -C "boostmentor-k8s" -N ""

ssh-copy-id -i ~/.ssh/bm_k8s.pub node1@192.168.56.101
ssh-copy-id -i ~/.ssh/bm_k8s.pub node2@192.168.56.102
ssh-copy-id -i ~/.ssh/bm_k8s.pub node3@192.168.56.103

ssh -i ~/.ssh/bm_k8s node1@192.168.56.101 hostname
ssh -i ~/.ssh/bm_k8s node2@192.168.56.102 hostname
ssh -i ~/.ssh/bm_k8s node3@192.168.56.103 hostname
```

чтобы удобнее работалось и через эти алиасы Ansible далее будет работать:

```bash
code ~/.ssh/config


Host node1
  HostName 127.0.0.1
  User node1
  Port 2201
  IdentityFile ~/.ssh/bm_k8s
  IdentitiesOnly yes

Host node2
  HostName 127.0.0.1
  User node2
  Port 2202
  IdentityFile ~/.ssh/bm_k8s
  IdentitiesOnly yes

Host node3
  HostName 127.0.0.1
  User node3
  Port 2203
  IdentityFile ~/.ssh/bm_k8s
  IdentitiesOnly yes
```

Раз прямой путь через host-only сеть уже работает без единого port-forwarding правила `ssh -i ~/.ssh/bm_k8s node1@192.168.56.101 hostname` — конфиг стоит сильно упростить и убрать NAT-путь совсем, чтобы не путаться в двух параллельных способах подключения:

```bash
Host node1
  HostName 192.168.56.101
  User node1
  IdentityFile ~/.ssh/bm_k8s
  IdentitiesOnly yes

Host node2
  HostName 192.168.56.102
  User node2
  IdentityFile ~/.ssh/bm_k8s
  IdentitiesOnly yes

Host node3
  HostName 192.168.56.103
  User node3
  IdentityFile ~/.ssh/bm_k8s
  IdentitiesOnly yes
```

раз соединение идёт через host-only сеть напрямую к самой VM (а не через NAT-проброс на хосте), гостевой sshd слушает именно 22, порт-форвардинг тут ни при чём.

```





```

## на ноде 1

```bash
ssh node1 "sudo cp /etc/kubernetes/admin.conf /home/node1/admin.conf && sudo chown node1:node1 /home/node1/admin.conf"
```

## на локальной машине/хосте, с которого будет управляться кластер

```bash
mkdir -p ~/.kube
scp node1:/home/node1/admin.conf ~/.kube/config
chmod 600 ~/.kube/config
```

Копируем уже "разрешённую" копию к себе на хост и сразу ставим права 600 — этот файл даёт полный доступ администратора к кластеру, его стоит защищать так же, как приватный SSH-ключ.

```bash
ssh node1 "rm /home/node1/admin.conf"
```

Подчищаем временную копию на ноде — незачем оставлять там второй экземпляр админского конфига.

# Проблема 2: на хосте нет kubectl

Ставим только kubectl (не kubeadm/kubelet — они на хосте не нужны, хост не часть кластера, а просто клиент), тем же способом, что и на нодах — через официальный репозиторий pkgs.k8s.io:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```

### Версию v1.34 берём такой же, как на нодах (v1.34.11) — kubectl совместим с ±1 минорной версией сервера, но проще держать в тон, чтобы не думать об этом вообще.

## Проверка

```bash
kubectl get nodes
```

Это должно сработать без флагов, потому что `kubectl` по умолчанию ищет конфиг именно в `~/.kube/config`. Сработает это благодаря тому, что ваш хост Linux Mint уже состоит в той же host-only сети (`192.168.56.0/24` через `vboxnet0`), а admin.conf внутри ссылается на `https://192.168.56.101:6443` — тот самый `--control-plane-endpoint`, который вы указывали при kubeadm init. Проверить, на какой адрес он смотрит, можно так:

```bash
grep server ~/.kube/config
```

После этого вы сможете управлять всем кластером прямо с Linux Mint, ни разу не заходя по SSH на node1
