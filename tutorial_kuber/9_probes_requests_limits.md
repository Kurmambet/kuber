# Probes practice

Good readiness:

```bash
kubectl apply -f 10-probes/probe-demo-good.yaml
kubectl apply -f 10-probes/probe-service.yaml
kubectl get pods
kubectl get endpoints
```

Broken readiness:

```bash
# путь неверный в этой readiness пробе
kubectl apply -f 10-probes/probe-demo-bad-readiness.yaml
kubectl get pods
kubectl describe pod <pod-name>
kubectl get endpoints
```

Вывод: Running != Ready. Service должен отправлять трафик только в Ready pod'ы.

# Requests / limits practice

- Requests - сколько ресурсов выделяется/резервируется при размещении пода
- limits - сколько контейнеру разрешено потреблять

- CPU - соотношение по cpu 1/6 (req)/(lim) -- троттлит, тормозит
- MEMORY - по памяти: 1/2 -- oom killer тупа убивает процесс

```bash
# манифкест просит слишком много - не запустится
kubectl apply -f 11-resources/huge-request-pod.yaml
kubectl get pods
kubectl describe pod huge-request
```

Вывод: scheduler смотрит на requests. Если ресурсов нет — pod Pending.

Рабочий пример:

```bash
kubectl apply -f 11-resources/small-request-pod.yaml
kubectl get pods
```

# QoS классы

### QoS = кого кубер будет беречь, а кого первым выкинет при нехватке ресурсов

- `Guaranteed` - Защищенный/гарантированный класс.
  - evict в последнюю очередь.
  - указаны все cpu/mem req и lim. `req = lim`
- `Burstable` - `req < lim` выше req, но ниже lim - под бёрстится
  - заданы, но req != lim или заданы не все ресурсы.

- `BestEffort` - `запустится если ресурсы есть`, не гарантированно.
  - когда вообще нет (не указаны) requests и limits.
  - 1'й кандидат на выбывание (eviction)
