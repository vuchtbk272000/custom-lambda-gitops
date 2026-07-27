# Tenant GitOps layout (Tesco-style Helm)

Same pattern as `application-template` + `api` in tesco-k8s:

| Tesco | OpenFunction |
|---|---|
| `applications/application-template` | `tenants/function-template` |
| `applications/api` (Chart.yaml + values.yaml) | `tenants/tenant-a`, `tenants/tenant-b` |

```
tenants/
├── function-template/     # shared Helm chart (excluded from ApplicationSet)
├── tenant-a/              # thin chart → depends on function-template
│   ├── Chart.yaml
│   └── values.yaml
└── tenant-b/
    ├── Chart.yaml
    └── values.yaml
```

## Add a tenant

```bash
mkdir tenants/tenant-c
# Chart.yaml with dependency file://../function-template
# values.yaml under key function-template: { namespace, functions: [...] }
cd tenants/tenant-c && helm dependency update
git add tenants/tenant-c && git push
```

## Secrets

Still namespace-scoped. Bootstrap from `default`:

```bash
./scripts/bootstrap-tenant-secrets.sh tenant-a-lambda tenant-b-lambda
```

Or enable `function-template.secrets.enabled` with Sealed/External Secrets — never commit plaintext tokens.
