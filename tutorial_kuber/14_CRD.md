custom resource definition

под оставляет заявку PVC (PersistentVolumeClaim),
StorasgeClass по этой заявке динамически создает PV (PersistentVolume) - реальный диск и монтирует его к поду.

нужен provisioner (создает папки)

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml


kubectl apply -f ultimate-devops-guide/lab/13-storage/pvc-demo.yaml

kubectl get pvc,pv
```

## проверим, что файл переживет удаление пода

```bash
kubectl exec demo-volume -- sh -c 'echo "Hi there" > /usr/share/nginx/html/test.txt'
kubectl delete pod demo-volume
kubectl apply -f ultimate-devops-guide/lab/13-storage/pvc-demo.yaml
kubectl exec demo-volume -- cat /usr/share/nginx/html/test.txt
```
