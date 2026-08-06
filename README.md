# goit-argo — GitOps-репозиторій

Джерело правди для Kubernetes-ресурсів у EKS-кластері. Кластер синхронізується
з цим репозиторієм автоматично — руками `kubectl apply` не робимо.

## Структура

```
goit-argo
├── namespace
│   ├── application
│   │   ├── demo-nginx.yaml   # Deployment + Service демозастосунку
│   │   └── ns.yaml           # namespace "application"
│   └── infra-tools
│       └── ns.yaml           # namespace "infra-tools" (тут живе Argo CD)
└── README.md
```

## Як це працює

У namespace `infra-tools` розгорнуто `ApplicationSet` **namespaces**
(створюється Terraform-ом, див. `terraform/argocd/main.tf` в основному репозиторії).
Він використовує **git-генератор типу `directories`** з патерном `namespace/*`
і для кожної знайденої директорії створює окремий Argo CD `Application`:

| Директорія в Git       | Argo CD Application | Цільовий namespace |
| ---------------------- | ------------------- | ------------------ |
| `namespace/application` | `application`       | `application`      |
| `namespace/infra-tools` | `infra-tools`       | `infra-tools`      |

Тобто **ім'я директорії = ім'я Application = ім'я namespace**.

Політика синхронізації: `automated` + `prune` + `selfHeal`,
`CreateNamespace=true`, `ServerSideApply=true`.

## Як додати новий namespace

```bash
mkdir -p namespace/monitoring
cat > namespace/monitoring/ns.yaml <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
EOF

git add namespace/monitoring
git commit -m "feat: add monitoring namespace"
git push
```

Через ~60 секунд (`timeout.reconciliation: 60s`) ApplicationSet сам створить
Application `monitoring`, а Argo CD — namespace у кластері. Terraform чіпати не треба.

## Як додати застосунок у наявний namespace

Покладіть маніфест у відповідну директорію (наприклад
`namespace/application/my-app.yaml`) і зробіть `git push`.
`directory.recurse: true` означає, що будуть підхоплені всі `*.yaml` у папці.

## Важливо

- Не видаляйте `namespace/infra-tools/ns.yaml` — там анотація `Prune=false`,
  але сама директорія все ще потрібна ApplicationSet.
- Гілка за замовчуванням — `main` (`targetRevision`). Комітіть саме в неї,
  інакше Argo CD не побачить змін.
