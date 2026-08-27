ConfigMap - не секретная конфигурация, которые не зашиты внутрь докер образа

Secret по умолчанию не зашифрован, хранится в base64.
Vault в проде или external secrets

```bash
kubectl create configmap app-conmfig --from-literal=APP_MODE=production --from-literal=APP_COLOR=blue
# опечатка, ну и х с ним
kubectl get configmap
kubectl get configmap app-conmfig -o yaml
```

```bash
kubectl create secret generic app-secret --from-literal=DB_PASSWORD=supersecret --from-literal=API_TOKEN=token123

kubectl get secrets
kubectl get secret app-secret -o yaml
# тут уже закодированно в base64
kubectl get secret app-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d; echo
# без проблем декодировалось. это не шифрование
```

развернем бд с секретами. снимем taint с node1. (см. 6 создание подов.md)
установка докера на node1:

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
noble stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update


sudo usermod -aG docker $USER
newgrp docker

docker rm -f $(docker ps -aq)
docker rmi hello-world:latest


cd /home/node1/ultimate-devops-guide/app/db
docker compose up --build -d

docker compose exec db psql -U scott -d scott -c 'select count(*) from quotes'
docker compose exec db psql -U scott -d scott -c 'select * from quotes'
```

применить секреты:

```bash
kubectl apply -f ultimate-devops-guide/app/k8s/secret.yaml
# изменяем манифест как в 11 my app in kuber md
# image: 192.168.56.1:5000/scott-pilgrim-quotes:1.0.0
        #   env:
        #     - name: DB_HOST
        #       value: "192.168.56.101"

# проверяем, работает ли registry
docker ps | grep registry
docker start registry
curl http://192.168.56.1:5000/v2/scott-pilgrim-quotes/tags/list
# и применяем
kubectl apply -f ultimate-devops-guide/app/k8s/deployment-db.yaml

kubectl get pods -l app=scott-pilgrim -w
```

где хранятся секреты:
в etcd

```bash
ps -efww | grep '[e]tcd'
ps -efww | grep '[e]tcd-keyfile'
```
