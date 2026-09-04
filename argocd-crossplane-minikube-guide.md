# Руководство: ArgoCD + Crossplane на minikube + тестовый AWS EC2

Ниже — пошаговая инструкция: поднимаем minikube, ставим ArgoCD, ставим Crossplane с AWS-провайдером (актуальная схема через `provider-family-aws`), настраиваем доступ к AWS и через Crossplane CRD (по желанию — через ArgoCD Application, т.е. GitOps) создаём тестовый EC2-инстанс.

---

## 0. Предварительные требования

- Docker или другой драйвер для minikube
- `kubectl` ≥ 1.28
- `helm` ≥ 3.8
- `minikube` последней версии
- AWS-аккаунт с IAM-пользователем/ключами, у которых есть права на EC2 (для теста достаточно `AmazonEC2FullAccess` или узкой кастомной политики)

Проверка версий:

```bash
kubectl version --client
helm version
minikube version
```

---

## 1. Запуск кластера minikube

Crossplane и ArgoCD достаточно прожорливы, поэтому кластеру нужно чуть больше ресурсов, чем дефолт:

```bash
minikube start --cpus=4 --memory=8192 --driver=docker
kubectl get nodes
```

---

## 2. Установка ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Ждём, пока поды поднимутся
kubectl -n argocd wait --for=condition=available --timeout=300s deployment --all
```

### Доступ к UI и CLI

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Открыть `https://localhost:8080` (логин `admin`).

Получить начальный пароль:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Логин через CLI (опционально, если установлен `argocd` CLI):

```bash
argocd login localhost:8080 --username admin --password <пароль> --insecure
```

---

## 3. Установка Crossplane

Crossplane можно поставить напрямую через Helm, либо (что органичнее сочетается с ArgoCD) — как ArgoCD `Application`, указывающий на Helm-чарт Crossplane. Ниже — оба варианта, выбирайте один.

### Вариант A — напрямую через Helm

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

kubectl create namespace crossplane-system

helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --set args='{--enable-usages}'

kubectl -n crossplane-system get pods -w
```

### Вариант B — через ArgoCD (GitOps-стиль)

```yaml
# crossplane-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.crossplane.io/stable
    chart: crossplane
    targetRevision: "*"
    helm:
      parameters:
        - name: args[0]
          value: "--enable-usages"
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f crossplane-app.yaml
```

Проверка, что CRD и контроллер поднялись:

```bash
kubectl get crds | grep crossplane.io
kubectl -n crossplane-system get pods
```

---

## 4. Установка AWS-провайдера Crossplane

Текущая (актуальная на 2026 год) схема — семейство провайдеров `provider-family-aws` от Upbound: ставится «корневой» провайдер, а под конкретные сервисы (EC2, S3, IAM и т.д.) подключаются отдельные лёгкие суб-провайдеры. Для теста с EC2 нужны `provider-family-aws` + `provider-aws-ec2`.

```yaml
# provider-aws-family.yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-family-aws
spec:
  package: xpkg.upbound.io/upbound/provider-family-aws:v1.0.0
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-ec2
spec:
  package: xpkg.upbound.io/upbound/provider-aws-ec2:v1.0.0
```

> Версии пакетов проверьте на https://marketplace.upbound.io/providers перед установкой — там всегда актуальные теги.

```bash
kubectl apply -f provider-aws-family.yaml

# Ждём готовности провайдеров
kubectl wait --for=condition=healthy provider.pkg.crossplane.io/provider-family-aws --timeout=300s
kubectl wait --for=condition=healthy provider.pkg.crossplane.io/provider-aws-ec2 --timeout=300s

kubectl get providers
```

Если ставите через ArgoCD — тот же YAML кладётся в Git-репозиторий, и создаётся `Application`, указывающий на эту папку/манифест.

---

## 5. Настройка credentials для AWS-провайдера

Самый простой вариант для теста в minikube — статические ключи через `Secret` (для продакшена лучше IRSA/OIDC, но в minikube это неприменимо).

```bash
cat <<EOF > aws-creds.conf
[default]
aws_access_key_id = <ВАШ_ACCESS_KEY>
aws_secret_access_key = <ВАШ_SECRET_KEY>
EOF

kubectl create secret generic aws-secret \
  -n crossplane-system \
  --from-file=creds=./aws-creds.conf

rm aws-creds.conf
```

Создаём `ProviderConfig`, который свяжет секрет с провайдером:

```yaml
# providerconfig-aws.yaml
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-secret
      key: creds
```

```bash
kubectl apply -f providerconfig-aws.yaml
kubectl get providerconfig
```

---

## 6. Создание тестового EC2-инстанса через Crossplane

Пример минимального managed resource — Amazon Linux 2 `t2.micro` в регионе `us-east-1` (замените AMI/регион под себя):

```yaml
# test-ec2-instance.yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Instance
metadata:
  name: crossplane-test-instance
spec:
  forProvider:
    region: us-east-1
    ami: ami-0c101f26f147fa7fd   # Amazon Linux 2, us-east-1 - проверьте актуальный ID
    instanceType: t2.micro
    tags:
      Name: crossplane-test-instance
      ManagedBy: crossplane
  providerConfigRef:
    name: default
```

Применяем:

```bash
kubectl apply -f test-ec2-instance.yaml
```

Проверка статуса:

```bash
kubectl get instance.ec2.aws.upbound.io
kubectl describe instance.ec2.aws.upbound.io crossplane-test-instance
```

Когда в колонке `SYNCED` и `READY` появится `True` — инстанс создан в AWS. Проверить можно и напрямую:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=crossplane-test-instance" \
  --query "Reservations[].Instances[].{ID:InstanceId,State:State.Name}"
```

### Вариант через ArgoCD (полный GitOps-цикл)

Если хотите, чтобы EC2-манифест применялся не вручную, а через ArgoCD:

1. Положите `test-ec2-instance.yaml` в Git-репозиторий (например, папку `crossplane/resources/`).
2. Создайте `Application`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-ec2-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<ваш-репозиторий>.git
    targetRevision: main
    path: crossplane/resources
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
kubectl apply -f argocd-ec2-app.yaml
```

ArgoCD синхронизирует ресурс, а Crossplane — реально создаст EC2 в AWS. Изменения EC2-манифеста в Git = изменения в облаке.

---

## 7. Удаление тестового инстанса

```bash
kubectl delete -f test-ec2-instance.yaml
```

Crossplane удалит EC2-инстанс из AWS через `deletionPolicy` (по умолчанию `Delete`). Проверьте:

```bash
aws ec2 describe-instances --filters "Name=tag:Name,Values=crossplane-test-instance"
```

---

## 8. Частые проблемы

| Симптом | Причина / решение |
|---|---|
| `Provider` висит в `Installed=True`, `Healthy=Unknown` | Подождать 1–2 минуты, проверить `kubectl -n crossplane-system logs deploy/crossplane` |
| `SYNCED=False` на Instance | Проверить `kubectl describe` — обычно неверные AWS-креды или права IAM |
| minikube не тянет ресурсы | Увеличьте `--memory`/`--cpus` при `minikube start`, или используйте `minikube start --driver=docker --memory=no-limit` (Docker Desktop с достаточным лимитом) |
| ArgoCD `OutOfSync` на Crossplane-ресурсах | Обычно нормально для managed resources, у которых часть полей заполняется контроллером после создания — настройте `ignoreDifferences` в Application при необходимости |

---

## Итоговая последовательность команд (кратко)

```bash
minikube start --cpus=4 --memory=8192
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
kubectl create namespace crossplane-system
helm install crossplane crossplane-stable/crossplane -n crossplane-system

kubectl apply -f provider-aws-family.yaml
kubectl create secret generic aws-secret -n crossplane-system --from-file=creds=./aws-creds.conf
kubectl apply -f providerconfig-aws.yaml
kubectl apply -f test-ec2-instance.yaml
```

Все YAML-файлы из руководства можно объединить в один Git-репозиторий и полностью отдать под управление ArgoCD (`app-of-apps` паттерн), если нужен единый источник правды.
