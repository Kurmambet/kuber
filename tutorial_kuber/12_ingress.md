Traefic/nginx

````bash
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
```scott-pilgrim

```bash
kubectl apply -f lab/05-ingress/scott-ingress.yaml

# Проверка
kubectl get ingress
# и на полученный ip и хост отправляем
curl -H "Host: scott.local" http://192.168.56.103
curl -H "Host: scott.local" http://192.168.56.103/quote
# проверка, что роутинг работает правильно:
curl -H "Host: web.local" http://192.168.56.103
````

# Labels/Selectors

сервисы и репликасеты не знают поды по имени, они выбирают по лейблам

```bash
kubectl get pods --show-labels
kubectl get pods -l app=scott-pilgrim
```
