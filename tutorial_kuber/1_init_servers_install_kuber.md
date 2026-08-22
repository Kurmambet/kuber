|                              | Internal Network (intnet)                            | Host-only (vboxnet0)                              |
| ---------------------------- | ---------------------------------------------------- | ------------------------------------------------- |
| VM ↔ VM                      | Работает                                             | Работает                                          |
| VM ↔ хост                    | Невозможно — хост физически не подключён к этой сети | Работает — хост получает свой IP в той же подсети |
| DHCP                         | Нет, IP только вручную                               | Можно включить DHCP-сервер VirtualBox             |
| Видно снаружи (LAN/интернет) | Нет                                                  | Нет                                               |

проброс портов к виртуалкам

```
Имя: ssh-node1
Протокол: TCP
Host IP:   (пусто, или 127.0.0.1 — если хотите доступ только с локальной машины)
Host Port: 2201
Guest IP:  (пусто)
Guest Port: 22
```

```bash
# sshd вообще установлен и запущен?
sudo systemctl status ssh
# если неактивен/не найден:
sudo apt update && sudo apt install -y openssh-server
sudo systemctl enable --now ssh

# фаервол не блокирует 22-й порт?
sudo ufw status
sudo ufw allow 22/tcp

# сам с себя коннектится нормально?
ssh localhost
```

конфигурация сети виртуалок

```bash
sudo nano /etc/netplan/00-installer-config.yaml   # блок enp0s8
sudo netplan apply
ip a show enp0s8   # убедиться, что адрес не изменился и dynamic -> valid_lft forever


node1

network:
  ethernets:
    enp0s3:
      dhcp4: true
      dhcp6: true
      match:
        macaddress: 08:00:27:91:1f:a7
      set-name: enp0s3
    enp0s8:
      dhcp4: false
      match:
        macaddress: 08:00:27:64:09:7b
      set-name: enp0s8
      addresses:
        - 192.168.56.101/24
  version: 2


node2


network:
  ethernets:
    enp0s3:
      dhcp4: true
      dhcp6: true
      match:
        macaddress: 08:00:27:35:0f:06
      set-name: enp0s3
    enp0s8:
      dhcp4: false
      match:
        macaddress: 08:00:27:48:da:f9
      set-name: enp0s8
      addresses:
        - 192.168.56.102/24
  version: 2


node3


network:
  ethernets:
    enp0s3:
      dhcp4: true
      dhcp6: true
      match:
        macaddress: 08:00:27:4d:33:f0
      set-name: enp0s3
    enp0s8:
      dhcp4: false
      match:
        macaddress: 08:00:27:e1:7f:38
      set-name: enp0s8
      addresses:
        - 192.168.56.103/24
  version: 2




sudo netplan try
ip a



node1@node1:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:91:1f:a7 brd ff:ff:ff:ff:ff:ff
    altname enx080027911fa7
    inet 10.0.2.15/24 metric 100 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 86354sec preferred_lft 86354sec
    inet6 fd17:625c:f037:2:a00:27ff:fe91:1fa7/64 scope global dynamic mngtmpaddr noprefixroute
       valid_lft 86355sec preferred_lft 14355sec
    inet6 fe80::a00:27ff:fe91:1fa7/64 scope link proto kernel_ll
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:64:09:7b brd ff:ff:ff:ff:ff:ff
    altname enx08002764097b
    inet 192.168.56.101/24 brd 192.168.56.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe64:97b/64 scope link proto kernel_ll
       valid_lft forever preferred_lft forever




node2@node2:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:35:0f:06 brd ff:ff:ff:ff:ff:ff
    altname enx080027350f06
    inet 10.0.2.15/24 metric 100 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 86362sec preferred_lft 86362sec
    inet6 fd17:625c:f037:2:a00:27ff:fe35:f06/64 scope global dynamic mngtmpaddr noprefixroute
       valid_lft 86363sec preferred_lft 14363sec
    inet6 fe80::a00:27ff:fe35:f06/64 scope link proto kernel_ll
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:48:da:f9 brd ff:ff:ff:ff:ff:ff
    altname enx08002748daf9
    inet 192.168.56.102/24 brd 192.168.56.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe48:daf9/64 scope link proto kernel_ll
       valid_lft forever preferred_lft forever


node3@node3:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:4d:33:f0 brd ff:ff:ff:ff:ff:ff
    altname enx0800274d33f0
    inet 10.0.2.15/24 metric 100 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 86367sec preferred_lft 86367sec
    inet6 fd17:625c:f037:2:a00:27ff:fe4d:33f0/64 scope global dynamic mngtmpaddr noprefixroute
       valid_lft 86367sec preferred_lft 14367sec
    inet6 fe80::a00:27ff:fe4d:33f0/64 scope link proto kernel_ll
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:e1:7f:38 brd ff:ff:ff:ff:ff:ff
    altname enx080027e17f38
    inet 192.168.56.103/24 brd 192.168.56.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fee1:7f38/64 scope link proto kernel_ll
       valid_lft forever preferred_lft forever







node2@node2:~$ ping -c 3 192.168.56.101 && ping -c 3 192.168.56.103
PING 192.168.56.101 (192.168.56.101) 56(84) bytes of data.
64 bytes from 192.168.56.101: icmp_seq=1 ttl=64 time=1.89 ms
64 bytes from 192.168.56.101: icmp_seq=2 ttl=64 time=0.926 ms
64 bytes from 192.168.56.101: icmp_seq=3 ttl=64 time=0.933 ms

--- 192.168.56.101 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2921ms
rtt min/avg/max/mdev = 0.926/1.250/1.893/0.454 ms
PING 192.168.56.103 (192.168.56.103) 56(84) bytes of data.
64 bytes from 192.168.56.103: icmp_seq=1 ttl=64 time=1.97 ms
64 bytes from 192.168.56.103: icmp_seq=2 ttl=64 time=0.352 ms
64 bytes from 192.168.56.103: icmp_seq=3 ttl=64 time=1.78 ms

--- 192.168.56.103 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2074ms
rtt min/avg/max/mdev = 0.352/1.366/1.970/0.721 ms
node2@node2:~$


node3@node3:~$ ping -c 3 192.168.56.101 && ping -c 3 192.168.56.102
PING 192.168.56.101 (192.168.56.101) 56(84) bytes of data.
64 bytes from 192.168.56.101: icmp_seq=1 ttl=64 time=4.71 ms
64 bytes from 192.168.56.101: icmp_seq=2 ttl=64 time=0.264 ms
64 bytes from 192.168.56.101: icmp_seq=3 ttl=64 time=0.946 ms

--- 192.168.56.101 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2604ms
rtt min/avg/max/mdev = 0.264/1.972/4.707/1.953 ms
PING 192.168.56.102 (192.168.56.102) 56(84) bytes of data.
64 bytes from 192.168.56.102: icmp_seq=1 ttl=64 time=1.90 ms
64 bytes from 192.168.56.102: icmp_seq=2 ttl=64 time=1.34 ms
64 bytes from 192.168.56.102: icmp_seq=3 ttl=64 time=1.24 ms

--- 192.168.56.102 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2160ms
rtt min/avg/max/mdev = 1.238/1.493/1.899/0.290 ms
node3@node3:~$



sudo netplan apply


# резолвинг по именам. пригодится и для kubeadm, и позже для Ansible/Kubespray
sudo tee -a /etc/hosts <<'EOF'
192.168.56.101 node1
192.168.56.102 node2
192.168.56.103 node3
EOF
```

```


```

# Установка кубера

```


```

Все команды из этого блока выполняются **одинаково на всех трёх нодах** (node1, node2, node3), под `sudo`.

## Шаг 1. Отключить swap

Kubelet отказывается нормально работать со включённым swap (планировщик ресурсов рассчитывает лимиты памяти, исходя из того, что swap отсутствует) .

```bash
sudo swapoff -a
```

Немедленно выключает весь swap **в текущей сессии** (до перезагрузки). Если сейчас ничего не свопится — команда просто ничего не выведет, это нормально.

```bash
sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab
```

Правит файл `/etc/fstab` (он описывает, какие разделы монтировать при загрузке) — находит строку со словом `swap` и ставит `#` в начале, комментируя её. Это нужно, чтобы swap не включился обратно **после перезагрузки** — `swapoff -a` действует только до ребута.

```bash
sudo systemctl mask swap.target
```

Дополнительная страховка для Ubuntu 24.04+/26.04: там появился systemd-юнит `swap.target`, который может снова активировать swap-раздел даже если он закомментирован в fstab (например, если это zram или swapfile, управляемый systemd). `mask` жёстко блокирует его запуск. [askubuntu](https://askubuntu.com/questions/1065503/how-to-remove-systemd-targets)

Проверка:

```bash
free -h
```

В столбце `Swap` везде должны быть нули.

## Шаг 2. Модули ядра и sysctl-параметры

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

Создаёт файл, который говорит systemd **при каждой загрузке** автоматически подгружать два модуля ядра:

- `overlay` — файловая система, на которой работают слои container-образов (без неё контейнеры не запустятся);
- `br_netfilter` — модуль, который даёт iptables видеть трафик, идущий через Linux-мост (bridge) между подами — без него нет корректной маршрутизации трафика между контейнерами .

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Подгружает эти же модули **прямо сейчас**, не дожидаясь перезагрузки — чтобы не перезагружать VM ради этого шага.

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

Создаёт файл с тремя настройками ядра, которые будут применяться автоматически при загрузке:

- первые две строки — заставляют трафик через bridge реально проверяться правилами iptables/ip6tables (см. объяснение из прошлого ответа);
- `ip_forward = 1` — разрешает ядру пересылать (роутить) пакеты между сетевыми интерфейсами. Без этого нода физически не сможет передать пакет от пода на этой ноде поду на другой ноде.

```bash
sudo sysctl --system
```

Перечитывает **все** файлы sysctl-конфигурации (включая тот, что вы только что создали) и применяет их немедленно, без перезагрузки. Выведет длинный список применённых параметров — это нормально.

## Шаг 3. Установить CRI-O (container runtime)

Kubelet сам по себе не умеет запускать контейнеры — ему нужен отдельный "движок" (container runtime), который общается с kubelet по протоколу CRI. Раньше все ставили Docker, но с 2022 года Kubernetes перестал поддерживать Docker напрямую (убрали dockershim), поэтому ставим CRI-O — он лёгкий и сделан специально под Kubernetes. [kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

```bash
sudo apt-get update
sudo apt-get install -y software-properties-common curl gnupg
```

Обновляет список пакетов и ставит утилиты, нужные для добавления сторонних репозиториев (иначе `apt` не знает, где скачать CRI-O — его нет в стандартных репозиториях Ubuntu).

```bash
export CRIO_VERSION=v1.34
```

Задаём переменную с версией CRI-O — она должна совпадать с минорной версией Kubernetes, которую будем ставить дальше (1.34), чтобы не было проблем совместимости CRI-протокола.

```bash
curl -fsSL https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key |
  sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
```

Скачивает публичный GPG-ключ, которым подписан репозиторий CRI-O, и сохраняет его в формате, понятном `apt` (`.gpg` бинарный вместо текстового `.asc`). Этот ключ нужен, чтобы apt мог проверить, что пакеты не подделаны. [kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

```bash
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/ /" |
  sudo tee /etc/apt/sources.list.d/cri-o.list
```

Добавляет сам репозиторий CRI-O в список источников apt, привязывая его к скачанному ключу.

```bash
sudo apt-get update
sudo apt-get install -y cri-o
```

Обновляет список пакетов (теперь apt уже видит новый репозиторий) и устанавливает сам CRI-O.

```bash
sudo systemctl enable --now crio
```

`enable` — включает автозапуск CRI-O при загрузке системы, `--now` — запускает его сразу же, не дожидаясь перезагрузки.

Проверка:

```bash
sudo systemctl status crio
```

Должно быть `active (running)`.

## Шаг 4. Установить kubeadm, kubelet, kubectl

Три разных инструмента:

- **kubeadm** — утилита для инициализации/присоединения нод к кластеру (её и запустим в следующем шаге);
- **kubelet** — агент, который реально работает на каждой ноде и общается с CRI-O, запуская поды;
- **kubectl** — консольный клиент для управления кластером (нужен только там, откуда вы будете администрировать — но проще поставить везде).

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

Ставит утилиты, нужные для работы с HTTPS-репозиториями и GPG-подписями (аналогично шагу с CRI-O).

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
```

Создаёт стандартную директорию для хранения ключей apt-репозиториев (современный способ, вместо устаревшего `apt-key`), с правами доступа 755.

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key |
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Скачивает официальный GPG-ключ репозитория Kubernetes (`pkgs.k8s.io` — официальный community-owned репозиторий проекта, пришедший на смену старому Google-репозиторию). [devopscube](https://devopscube.com/setup-kubernetes-cluster-kubeadm/)

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' |
  sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Добавляет сам репозиторий с пакетами Kubernetes версии 1.34.

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```

Обновляет список пакетов и ставит все три утилиты сразу.

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

Это важная команда — "замораживает" версии этих трёх пакетов, чтобы обычный `apt upgrade` их случайно не обновил. Обновление Kubernetes — отдельная, ручная и осторожная процедура (несовместимость версий может сломать кластер), поэтому автообновление явно запрещаем. [kubernetes](https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/)

```bash
sudo systemctl enable --now kubelet
```

Включает автозапуск и сразу стартует kubelet. На этом этапе он ещё будет "падать" в цикле рестарта — это нормально, он ждёт, пока `kubeadm init`/`kubeadm join` не сгенерирует ему конфигурацию.

## Проверка перед переходом дальше

```bash
kubeadm version
kubelet --version
kubectl version --client
crio --version
```

Все четыре команды должны отдать версии без ошибок. Когда прогоните это на всех трёх нодах и всё зелёное — переходим к `kubeadm init` на node1.

```





```

# Ининциализация кубера

```





```

Все версии совпадают (`v1.34.11`) на всех трёх нодах — это важно, разнобой версий kubeadm/kubelet между нодами частая причина проблем при инициализации. Переходим к `kubeadm init`.

## Шаг 5. Проверить firewall (только если включён)

```bash
sudo ufw status
```

Если выведет `Status: inactive` — ничего делать не нужно, идите дальше. Если `active` — на node1 нужно открыть порты, которые слушает control-plane:

```bash
sudo ufw allow 6443/tcp
sudo ufw allow 2379:2380/tcp
sudo ufw allow 10250:10259/tcp
```

На node2 — те же правила (он тоже control-plane). На node3 (worker) достаточно:

```bash
sudo ufw allow 10250/tcp
sudo ufw allow 30000:32767/tcp
```

## Шаг 6. Инициализация кластера — только на node1

```bash
sudo kubeadm init \
  --control-plane-endpoint "192.168.56.101:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket unix:///var/run/crio/crio.sock
```

Разбор флагов:

- `--control-plane-endpoint "192.168.56.101:6443"` — адрес, который станет "точкой входа" в API-сервер кластера. Порт `6443` — стандартный порт API-сервера Kubernetes. Указываем именно этот флаг (а не просто дефолт), потому что у нас будет несколько control-plane нод (node1 + node2), и всем клиентам/нодам нужен единый адрес, к которому можно обращаться.
- `--upload-certs` — kubeadm зашифрует и временно (на 2 часа) загрузит сертификаты control-plane в кластер, чтобы node2 при присоединении как второй control-plane мог их автоматически скачать, а не копировать вручную по SCP.
- `--pod-network-cidr=10.244.0.0/16` — резервирует диапазон IP-адресов для подов (виртуализируется CNI-плагином, к вашей физической сети `192.168.56.0/24` отношения не имеет, но должен с ней не пересекаться — не пересекается).
- `--cri-socket unix:///var/run/crio/crio.sock` — явно говорим kubeadm использовать именно CRI-O (а не искать containerd, который на вашей системе не установлен).

Команда выполняется 1–3 минуты. **Важно**: она выведет в консоль:

1. Блок с `kubeadm join ... --control-plane --certificate-key ...` — это команда для **node2**.
2. Блок с `kubeadm join ...` (без `--control-plane`) — это команда для **node3**.

Скопируйте оба блока целиком в текстовый файл на хосте — они одноразовые, токен просроченный (24 часа) их можно перегенерировать, но проще сохранить сразу.

## Шаг 7. Настроить доступ kubectl — на node1

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

`admin.conf` — файл с сертификатами и адресом кластера, который сгенерировал `kubeadm init`. По умолчанию он лежит там, где `kubectl` его не находит и куда у обычного пользователя нет прав на чтение. Эти три команды копируют его в стандартное место (`~/.kube/config`) и меняют владельца с root на вашего пользователя, чтобы `kubectl` можно было запускать без `sudo`.

Проверка (кластер пока будет "не готов" — это нормально, CNI ещё не поставили):

```bash
kubectl get nodes
```

Ожидаемо увидите node1 в статусе `NotReady` — это правильно на этом этапе.

## Шаг 8. Присоединить node2 как второй control-plane

На **node2** выполните команду, которую вывел `kubeadm init` — она выглядит примерно так (у вас будут свои токен и хэш):

```bash
sudo kubeadm join 192.168.56.101:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane \
  --certificate-key <CERT_KEY> \
  --cri-socket unix:///var/run/crio/crio.sock
```

```bash
sudo kubeadm join 192.168.56.101:6443 \
  --token 6tv0r9.tyrtvwnf0z3xc85n \
  --discovery-token-ca-cert-hash sha256:8590f5207a2bcb0aadccbcd0fc9de327c5a3bd1b9670a66ef37802565ff5324a \
  --control-plane \
  --certificate-key e43a14199ee193a73ce743d4cb220349bdb4f64ad1597224bdb609500fc05216 \
  --cri-socket unix:///var/run/crio/crio.sock
```

Флаг `--cri-socket` я дописал вручную — kubeadm его не включает в автогенерируемую команду, но без него на CRI-O системе join иногда падает с ошибкой "multiple CRI sockets found" (у CRI-O и потенциально ещё чего-то может быть несколько сокетов).

## Шаг 9. Присоединить node3 как worker

На **node3** — вторая сохранённая команда:

```bash
sudo kubeadm join 192.168.56.101:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --cri-socket unix:///var/run/crio/crio.sock
```

```bash
sudo kubeadm join 192.168.56.101:6443 \
  --token 6tv0r9.tyrtvwnf0z3xc85n \
  --discovery-token-ca-cert-hash sha256:8590f5207a2bcb0aadccbcd0fc9de327c5a3bd1b9670a66ef37802565ff5324a \
  --cri-socket unix:///var/run/crio/crio.sock
```

Без `--control-plane` — значит, эта нода станет обычным worker (только kubelet, без API-server/etcd).

## Шаг 10. Установить CNI-плагин — только на node1

Без этого шага поды не получат сеть и зависнут в `Pending`.

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

Эта команда скачивает и применяет манифест Calico — набор DaemonSet'ов, которые поднимутся на каждой ноде и настроят сетевые маршруты между подами согласно `--pod-network-cidr`, который вы указали в `kubeadm init`.

## Финальная проверка

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

Ожидаемо: все три ноды в статусе `Ready`, все системные поды (`kube-system`, `calico-system`) в `Running`. Первые 1–2 минуты после join поды Calico могут быть в `Init`/`ContainerCreating` — это нормально, дайте кластеру устаканиться.

Готовы выполнять эти команды по порядку, или сначала хотите, чтобы я прошёлся отдельно по возможным ошибкам `kubeadm init` (что делать, если он зависнет на "waiting for kubelet")?

```




```

# Ошибка чисто для virtualbox

```




```

Нашли настоящую причину — и она довольно коварная, специфичная именно для VirtualBox с двумя сетевыми адаптерами.

## В чём проблема

Посмотрите на вывод `kubeadm init` на node1 — сертификат etcd подписан для IP `[10.0.2.15 127.0.0.1 ::1]`, а **не** для `192.168.56.101`. `kubeadm` сам определяет "основной" IP ноды по интерфейсу с маршрутом по умолчанию — а это у вас `enp0s3` (NAT, `10.0.2.15`), а не `enp0s8` (host-only, `192.168.56.101`), потому что у NAT-интерфейса метрика маршрута ниже (`metric 100`).

А теперь ключевая деталь: **`10.0.2.15` — это NAT-адрес, который VirtualBox выдаёт КАЖДОЙ виртуалке одинаковый**, потому что у каждой VM свой изолированный виртуальный NAT-сегмент, не связанный с другими VM. То есть `10.0.2.15` на node1 и `10.0.2.15` на node2 — это два физически разных, недостижимых друг для друга адреса, просто с одинаковым числом. Когда node2 пытается достучаться до etcd node1 по адресу `10.0.2.15:2379`, он на самом деле стучится в свой собственный NAT-сегмент — оттуда node1 не видно вообще, отсюда `context deadline exceeded` (таймаут). [reddit](https://www.reddit.com/r/kubernetes/comments/16d0uvl/joining_master_node_stuck_at_checketcd_checking/)

Связь между нодами реально работает только через host-only (`192.168.56.x`) — а именно этот адрес kubeadm и не использовал для etcd.

## План исправления

Нужно откатить всё и переинициализировать, явно указав kubeadm использовать host-only IP как основной адрес каждой ноды — и на control-plane (`--apiserver-advertise-address`), и на kubelet (`--node-ip`), иначе та же проблема вылезет позже уже с сетью подов.

### Шаг 1. Откат — на всех трёх нодах

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/etcd $HOME/.kube
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
```

`kubeadm reset` откатывает всё, что сделал init/join (сертификаты, манифесты, конфиги kubelet). Ручная чистка `/etc/kubernetes` и `/var/lib/etcd` подчищает то, что reset иногда оставляет. Очистка iptables убирает старые правила kube-proxy, которые иначе будут мешать новой инициализации.

### Шаг 2. Зафиксировать node-ip для kubelet — на всех трёх нодах

```bash
echo 'KUBELET_EXTRA_ARGS=--node-ip=<HOST_ONLY_IP_ЭТОЙ_НОДЫ>' | sudo tee /etc/default/kubelet
sudo systemctl daemon-reload
```

Пакет kubelet от kubeadm читает этот файл при каждом запуске и добавляет флаг `--node-ip`, который заставляет kubelet регистрировать ноду в кластере именно с этим IP, а не с автоопределённым `10.0.2.15`. Подставьте на node1 — `192.168.56.101`, на node2 — `192.168.56.102`, на node3 — `192.168.56.103`.

### Шаг 3. Переинициализация — на node1

```bash
sudo kubeadm init \
  --control-plane-endpoint "192.168.56.101:6443" \
  --apiserver-advertise-address=192.168.56.101 \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket unix:///var/run/crio/crio.sock
```

```bash
You can now join any number of control-plane nodes running the following command on each as root:

  kubeadm join 192.168.56.101:6443 --token c2di17.yokup4fjksqng48a \
        --discovery-token-ca-cert-hash sha256:188d226629ef41f693336f3ea5bb22e2d0557bc6aebe45e6211f0c38c9d3735d \
        --control-plane --certificate-key 6e48fe147c824b0c8e557c178919eb579e63bc761f99c41983de7af75daba128

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.56.101:6443 --token c2di17.yokup4fjksqng48a \
        --discovery-token-ca-cert-hash sha256:188d226629ef41f693336f3ea5bb22e2d0557bc6aebe45e6211f0c38c9d3735d
node1@node1:~$
```

Добавился `--apiserver-advertise-address=192.168.56.101` — теперь и apiserver, и etcd явно свяжутся именно с host-only адресом, а не с автоопределённым NAT-адресом.

Дальше как раньше:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Сохраните новые `kubeadm join` команды из вывода — токен и certificate-key будут новые.

### Шаг 4. node2 — присоединение с явным advertise-address

```bash
sudo kubeadm join 192.168.56.101:6443 \
  --token <НОВЫЙ_TOKEN> \
  --discovery-token-ca-cert-hash sha256:<НОВЫЙ_HASH> \
  --control-plane \
  --certificate-key <НОВЫЙ_CERT_KEY> \
  --apiserver-advertise-address=192.168.56.102 \
  --cri-socket unix:///var/run/crio/crio.sock
```

```bash
sudo kubeadm join 192.168.56.101:6443 \
  --token c2di17.yokup4fjksqng48a \
  --discovery-token-ca-cert-hash sha256:188d226629ef41f693336f3ea5bb22e2d0557bc6aebe45e6211f0c38c9d3735d \
  --control-plane \
  --certificate-key 6e48fe147c824b0c8e557c178919eb579e63bc761f99c41983de7af75daba128 \
  --apiserver-advertise-address=192.168.56.102 \
  --cri-socket unix:///var/run/crio/crio.sock
```

### Шаг 5. node3 — присоединение как worker

```bash
sudo kubeadm join 192.168.56.101:6443 \
  --token <НОВЫЙ_TOKEN> \
  --discovery-token-ca-cert-hash sha256:<НОВЫЙ_HASH> \
  --cri-socket unix:///var/run/crio/crio.sock
```

```bash
sudo kubeadm join 192.168.56.101:6443 \
  --token c2di17.yokup4fjksqng48a \
  --discovery-token-ca-cert-hash sha256:188d226629ef41f693336f3ea5bb22e2d0557bc6aebe45e6211f0c38c9d3735d \
  --cri-socket unix:///var/run/crio/crio.sock
```

Для worker `--apiserver-advertise-address` не нужен — этот флаг относится только к control-plane компонентам, а node-ip мы уже зафиксировали через `/etc/default/kubelet` в шаге 2.

## Дальше

После успешного join node2 — сразу дайте мне полный вывод (в этот раз он должен пройти этап `[check-etcd]` без ошибки), и я подскажу, когда переходить к CNI (Calico) и финальной проверке `kubectl get nodes`.

```




```

# Все ок

```




```

Отлично, оба join прошли чисто — etcd на node2 подключился без таймаута (фикс с `--apiserver-advertise-address` сработал), node3 тоже успешно присоединился как worker. Да, теперь CNI ставим один раз, только на node1.

## Шаг 10. Проверка перед CNI

Сначала посмотрите текущее состояние — все три ноды уже должны быть видны, но в статусе `NotReady`, потому что сети между подами ещё нет:

```bash
kubectl get nodes -o wide
```

Ожидаемо: три строки (node1, node2, node3), `STATUS = NotReady` у всех. Это нормально на этом этапе.

## Установка Calico

На **node1**:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

Эта команда обращается к API-серверу (через ваш `~/.kube/config`) и создаёт набор ресурсов: DaemonSet `calico-node` (запустится на каждой ноде и настроит маршруты между подами), Deployment `calico-kube-controllers`, CRD-объекты и RBAC-правила. Применять нужно **только один раз и только с одной ноды** — это не установка пакета на каждую машину, а отправка манифеста в кластер, а дальше Kubernetes сам разложит поды Calico по всем трём нодам через API.

## Проверка после установки

Подождите 1–2 минуты (нужно скачать образы Calico на все три ноды через интернет-NAT-адаптер) и проверьте:

```bash
kubectl get pods -n kube-system -o wide
kubectl get nodes
```

Что ожидать:

- Поды `calico-node-xxxxx` — по одному на каждую ноду, в статусе `Running`.
- Под `calico-kube-controllers-xxxxx` — один, в статусе `Running`.
- Все три ноды в `kubectl get nodes` должны переключиться из `NotReady` в `Ready`.

Если через пару минут статус не меняется — пришлите вывод обеих команд, разберём по логам конкретного пода (`kubectl describe pod <имя> -n kube-system` и `kubectl logs <имя> -n kube-system`), потому что при трёх сетевых интерфейсах на VM Calico тоже иногда пытается автоопределить не тот адрес — как это уже было с etcd.
