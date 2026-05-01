<p align="center">
  <img src="https://raw.githubusercontent.com/cert-manager/cert-manager/d53c0b9270f8cd90d908460d69502694e1838f5f/logo/logo-small.png" height="32" width="32" alt="cert-manager project logo" />
</p>

# cert-manager DNS-01 webhook for Beget

Форк [boryashkin/cert-manager-webhook-beget](https://github.com/boryashkin/cert-manager-webhook-beget) с обновлёнными зависимостями и собственной CI-сборкой.

## Что отличается от upstream

- Зависимости: cert-manager `v1.20.2`, k8s.io/* `v0.35.4`, Go `1.26.1`.
- Helm chart репо: `https://metrica-pro.github.io/cert-manager-webhook-beget/`.
- Docker image: `ghcr.io/metrica-pro/cert-manager-webhook-beget`.
- GitHub Actions: CI на каждый push, release сборка + публикация chart на push tag `v*`.

Логика webhook'а — без изменений: тот же DNS-01 solver через Beget DNS API (`api.beget.com`, login + passwd).

## Установка

```bash
helm repo add metrica-pro https://metrica-pro.github.io/cert-manager-webhook-beget/
helm repo update

helm install cert-manager-webhook-beget metrica-pro/cert-manager-beget-webhook \
    --namespace cert-manager \
    --set groupName=acme.your-domain.example
```

## Конфигурация

1. Создать Secret с Beget API кредами в `cert-manager` namespace (или там, где cert-manager controller имеет `--cluster-resource-namespace`):

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: beget-credentials
     namespace: cert-manager
   type: Opaque
   stringData:
     login: "your-login"
     passwd: "your-api-password"
   ```

2. ClusterIssuer:

   ```yaml
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
     name: beget-letsencrypt-prod
   spec:
     acme:
       server: https://acme-v02.api.letsencrypt.org/directory
       email: you@example.com
       privateKeySecretRef:
         name: beget-letsencrypt-prod
       solvers:
         - selector:
             dnsZones:
               - your-domain.example
           dns01:
             webhook:
               groupName: acme.your-domain.example
               solverName: beget
               config:
                 apiLoginSecretRef:
                   name: beget-credentials
                   key: login
                 apiPasswdSecretRef:
                   name: beget-credentials
                   key: passwd
   ```

3. Certificate:

   ```yaml
   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
     name: wildcard-tls
     namespace: <consumer-namespace>
   spec:
     secretName: wildcard-tls
     issuerRef:
       name: beget-letsencrypt-prod
       kind: ClusterIssuer
     dnsNames:
       - "*.your-domain.example"
       - your-domain.example
   ```

## Local dev

```bash
go test ./begetapi/... ./example/...    # unit тесты
go build ./...                           # компиляция
docker build -t webhook:dev .            # локальный образ
helm lint deploy/beget                   # линт chart'а
```

Полный integration test (`go test .`) требует `kubebuilder` tools (etcd + kube-apiserver) — см. `Makefile` target `_test/kubebuilder`.

## Лицензия

Apache 2.0 (как upstream).
