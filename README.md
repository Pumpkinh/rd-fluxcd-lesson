# rd-fluxcd-lesson

GitOps-репозиторій для домашнього завдання з Flux CD. Мета роботи - описати інфраструктуру застосунку `course-app` у Git та дати Flux автоматично синхронізувати бажаний стан з Kubernetes-кластером.

У проєкті використовується не Dragonfly/Redis, а PostgreSQL через CloudNativePG operator, тому що сам застосунок `course-app` працює з PostgreSQL.

## Структура репозиторію

```text
base/course-app              # базові маніфести course-app: Deployment, Service, Ingress
overlays/development         # dev-середовище
overlays/production          # prod-середовище
infrastructure/storage       # namespace, PV та підготовка hostPath-директорій
clusters/k8scontrol          # точка входу для Flux sync
```

## Що було зроблено

1. Створено GitHub-репозиторій `rd-fluxcd-lesson`.
2. У кластері встановлено Flux через Flux Operator.
3. Створено `FluxInstance`, який синхронізує кластер з цим репозиторієм:

```text
clusters/k8scontrol
```

4. У `base/course-app` винесено базові Kubernetes-ресурси застосунку:

- `Deployment`
- `Service`
- `Ingress`
- `kustomization.yaml`

5. Створено два overlay-середовища:

- `development`
- `production`

6. Для бази даних використано CNPG custom resource:

```yaml
kind: Cluster
apiVersion: postgresql.cnpg.io/v1
```

7. Для GitOps-синхронізації додано Flux `Kustomization` ресурси:

- `infra-storage`
- `app-dev`
- `app-prod`

## Development overlay

Файли розташовані у `overlays/development`.

Особливості:

- namespace: `development`
- `course-app` replicas: `1`
- PostgreSQL CNPG instances: `1`
- ingress host: `dev.course-app.local`
- один PVC для PostgreSQL

Перевірка:

```bash
kubectl get deploy,pods,svc,ingress,cluster,pvc -n development
```

## Production overlay

Файли розташовані у `overlays/production`.

Особливості:

- namespace: `production`
- `course-app` replicas: `3`
- PostgreSQL CNPG instances: `2`
- ingress host: `prod.course-app.local`
- два PVC для PostgreSQL
- додано `HorizontalPodAutoscaler`
- задано resource requests/limits для застосунку

Перевірка:

```bash
kubectl get deploy,pods,svc,ingress,cluster,hpa,pvc -n production
```

## Flux sync

Flux читає цей репозиторій і застосовує ресурси з `clusters/k8scontrol`.

Перевірка через Flux CLI:

```bash
flux get sources git
flux get kustomizations
```

Альтернативна перевірка через `kubectl`:

```bash
kubectl get gitrepositories,kustomizations -n flux-system
```

Очікуваний результат:

- `GitRepository/flux-system` у статусі `Ready=True`
- `Kustomization/flux-system` у статусі `Ready=True`
- `Kustomization/infra-storage` у статусі `Ready=True`
- `Kustomization/app-dev` у статусі `Ready=True`
- `Kustomization/app-prod` у статусі `Ready=True`

## Перевірка застосунку

Для доступу через Traefik використовується HTTP Host header.

```bash
curl -H 'Host: dev.course-app.local' http://<TRAEFIK_IP>/ready
curl -H 'Host: prod.course-app.local' http://<TRAEFIK_IP>/ready
```

У локальному кластері IP Traefik можна подивитися так:

```bash
kubectl get svc -n traefik traefik
```

## Drift check

Щоб перевірити GitOps-поведінку, можна вручну видалити сервіс у production:

```bash
kubectl delete svc course-app -n production
```

Після наступної синхронізації Flux має автоматично відновити ресурс:

```bash
kubectl get svc course-app -n production
```

## Що показати на скріншотах

Flux-ресурси у статусі Ready:

```bash
kubectl get gitrepositories,kustomizations -n flux-system
```

Різниця між development та production:

```bash
kubectl get deploy,pods,svc,ingress,cluster,hpa,pvc -n development
kubectl get deploy,pods,svc,ingress,cluster,hpa,pvc -n production
```

У production має бути видно додаткові ресурси:

- 3 pod-и застосунку
- 2 pod-и PostgreSQL
- `HorizontalPodAutoscaler`
- 2 PVC для PostgreSQL

## Примітки

У цьому кластері використовується `local-storage` та статичні PV, бо динамічний storage provisioner не налаштований. Для перенесення в інший кластер може знадобитися змінити storage-рішення або додати відповідний StorageClass.

Якщо HPA показує `cpu: <unknown>/70%`, це означає, що в кластері не встановлено Metrics API/metrics-server. Сам HPA-ресурс при цьому створений коректно.
