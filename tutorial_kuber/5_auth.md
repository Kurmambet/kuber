- аутентификация - `кто ты` TLS/mTLS серты или токены

- авторизация - `что тебе можно` - RBAC (Role + RoleBinding + ServiceAccount)

## чтобы выдать доступ к кластеру посторонним с урезанием прав

```bash
kubectl create namespace dev

# новый сервисный аккаунт read-only
kubectl create serviceaccount dev-reader -n dev

kubectl create role pod-reader -n dev --verb=get,list --resource=pods

# теперь связывам эту роль с сервисным аккаунтом
kubectl create rolebinding dev-reader-bind -n dev --role=pod-reader --serviceaccount=dev:dev-reader

# можем ли получить поды в неймспейсе dev от лица сервисного аккаунта dev-reader
kubectl auth can-i get pods --as=system:serviceaccount:dev:dev-reader -n dev

kubectl auth can-i delete pods --as=system:serviceaccount:dev:dev-reader -n dev
```

# или в виде манифестов

```yaml
---
# 1. Namespace — изолированное "пространство имён" для команды dev
apiVersion: v1
kind: Namespace
metadata:
  name: dev

---
# 2. ServiceAccount — "служебный пользователь" для программного доступа
#    (аналог обычного User, но для ботов/CI/приложений, а не живых людей)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dev-reader
  namespace: dev

---
# 3. Role — набор разрешений, ограниченный конкретным namespace (dev)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
  - apiGroups: [""] # "" означает core API group (там, где лежат Pod, Service, Node и т.д.)
    resources: ["pods"]
    verbs: ["get", "list"] # только читать — ни удалить, ни создать, ни изменить

---
# 4. RoleBinding — связка "кому" (ServiceAccount) даём "что" (Role)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-reader-bind
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: dev-reader
    namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

## Как это применить

Вместо четырёх отдельных `kubectl create` команд — один файл и одна команда:

```bash
kubectl apply -f dev-reader-rbac.yaml
```

`---` между блоками — это разделитель нескольких YAML-документов в одном файле, Kubernetes создаст все четыре сущности по порядку.

## Разбор структуры (аналогия с прошлыми объяснениями)

| Сущность         | Что это                                                       | Аналогия                                                    |
| ---------------- | ------------------------------------------------------------- | ----------------------------------------------------------- |
| `Namespace`      | Изолированное пространство имён                               | Как отдельная папка/окружение (dev/staging/prod)            |
| `ServiceAccount` | "Пользователь" для машин/ботов                                | Как сервисный API-токен, а не логин живого человека         |
| `Role`           | Список разрешений (verbs + resources), ограниченный namespace | Что "инструмент" **умеет** — только `get`/`list` над `pods` |
| `RoleBinding`    | Связка "кому" ↔ "какая Role"                                  | Собственно выдача прав конкретному ServiceAccount           |

Важная деталь про `Role` vs `ClusterRole`: `Role` работает **только внутри** указанного `namespace` (тут — `dev`). Если бы нужен был read-only доступ ко **всем** namespace сразу, использовали бы `ClusterRole` + `ClusterRoleBinding` вместо `Role`/`RoleBinding` — структура манифеста та же, только без поля `namespace` в метаданных и без namespace-привязки в биндинге.

## Проверка (те же команды, что у вас уже были)

```bash
kubectl auth can-i get pods --as=system:serviceaccount:dev:dev-reader -n dev
# yes

kubectl auth can-i delete pods --as=system:serviceaccount:dev:dev-reader -n dev
# no
```

`--as=system:serviceaccount:<namespace>:<имя>` — это специальный синтаксис "притвориться" конкретным ServiceAccount при проверке прав, не выпуская реального токена — удобно для отладки RBAC до того, как реально выдавать доступ кому-то.

Если этот `dev-reader` предназначен для реального человека/CI-системы вне кластера — следующий логичный шаг: сгенерировать для него токен (`kubectl create token dev-reader -n dev`) и положить в отдельный kubeconfig с ограниченным доступом, а не выдавать полноценный `admin.conf` целиком.

```




```

Хороший практический вопрос — сборка полноценного изолированного доступа состоит из трёх частей: получить токен, собрать под него отдельный kubeconfig-файл, и решить, какой тип токена нужен (короткоживущий для разработчика или долгоживущий для CI). Все команды — **на хосте**, где у вас уже настроен `kubectl` с полным доступом.

## Шаг 1. Получить токен для ServiceAccount

**Для разработчика (временный доступ, чтобы, например, зайти в контекст поработать час):**

```bash
kubectl create token dev-reader -n dev --duration=8h
```

`--duration` — токен автоматически перестанет работать через указанное время (по умолчанию 1 час). Это современный безопасный способ — токен нигде не хранится постоянно, генерируется по требованию через TokenRequest API.

**Для CI-системы (нужен токен, который не истекает через 8 часов посреди ночного пайплайна):**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dev-reader-token
  namespace: dev
  annotations:
    kubernetes.io/service-account.name: dev-reader
type: kubernetes.io/service-account-token
```

Начиная с Kubernetes 1.24 такие "вечные" токены больше не создаются автоматически при создании ServiceAccount (раньше создавались) — их нужно явно запросить через отдельный Secret такого типа. Применяем:

```bash
kubectl apply -f dev-reader-token-secret.yaml
kubectl get secret dev-reader-token -n dev -o jsonpath='{.data.token}' | base64 -d
```

Это выведет сам токен строкой — именно его CI-система будет использовать постоянно (пока вы явно не удалите Secret).

## Шаг 2. Собрать отдельный kubeconfig-файл

Разработчику или CI нельзя отдавать ваш `admin.conf` — нужен файл с теми же адресом/CA-сертификатом кластера, но с токеном ограниченного ServiceAccount вместо вашего админского.

```bash
CLUSTER_NAME=$(kubectl config view --minify -o jsonpath='{.clusters[0].name}')
SERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
CA_DATA=$(kubectl config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')
TOKEN=$(kubectl create token dev-reader -n dev --duration=8760h)   # для примера — почти год
```

```bash
cat <<EOF > dev-reader.kubeconfig
apiVersion: v1
kind: Config
clusters:
- name: ${CLUSTER_NAME}
  cluster:
    server: ${SERVER}
    certificate-authority-data: ${CA_DATA}
contexts:
- name: dev-reader-context
  context:
    cluster: ${CLUSTER_NAME}
    namespace: dev
    user: dev-reader
current-context: dev-reader-context
users:
- name: dev-reader
  user:
    token: ${TOKEN}
EOF
```

Получившийся файл `dev-reader.kubeconfig` — это самодостаточный конфиг: и адрес кластера, и сертификат для проверки TLS, и токен конкретного ограниченного пользователя. `namespace: dev` в контексте означает, что все команды по умолчанию будут работать именно в этом namespace, без надобности каждый раз писать `-n dev`.

## Шаг 3. Отдать файл и проверить

Разработчику/CI передаёте **только** этот файл (не ваш `~/.kube/config`!). У них команды выглядят так:

```bash
kubectl --kubeconfig=dev-reader.kubeconfig get pods
```

Сработает (это разрешено ролью `pod-reader`), а вот:

```bash
kubectl --kubeconfig=dev-reader.kubeconfig delete pod <любой>
```

Вернёт `Forbidden` — RBAC не даёт `delete`, ровно как вы и проверяли через `kubectl auth can-i`.

Для CI переменная окружения `KUBECONFIG=/path/to/dev-reader.kubeconfig` перед вызовом `kubectl`/`helm` — стандартный способ прокинуть этот файл в пайплайн (GitHub Actions/GitLab CI хранят такой файл как защищённый secret, base64-закодированным).

## Важные практики безопасности

- **Никогда** не коммитьте `dev-reader.kubeconfig` или сырой токен в Git — только через секреты CI-системы (GitHub Secrets, GitLab CI/CD Variables).
- Для разработчиков предпочитайте короткоживущие токены (`kubectl create token`) — их не нужно вручную отзывать, они просто истекают.
- Для CI, если Secret с токеном скомпрометирован — просто удалите его (`kubectl delete secret dev-reader-token -n dev`) и создайте заново, старый токен сразу станет недействителен.
- Если ролей/ServiceAccount'ов станет много — рассмотрите `kubectl create rolebinding` не напрямую на ServiceAccount, а через **Group**, чтобы управлять доступом пачками, а не по одному человеку.

Хотите, покажу, как то же самое сделать не для ServiceAccount, а для **реального пользователя с именем** (через клиентский X.509-сертификат) — это актуально, если у вас появится второй живой человек с собственным `kubectl` на своей машине, а не CI-бот?
