# Helm template for OpenFunction tenant functions
#
# Same pattern as Tesco `application-template` + `api`:
# - `function-template` = shared chart (templates)
# - `tenant-*` = thin charts that depend on it and only set values.yaml

## Structure

```
tenants/
├── function-template/          # shared chart (excluded from ApplicationSet)
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── _helpers.tpl
│       ├── function.yaml
│       └── secret.yaml
├── tenant-a/                   # tenant instance
│   ├── Chart.yaml
│   └── values.yaml
└── tenant-b/
    ├── Chart.yaml
    └── values.yaml
```

## Add a new tenant

```bash
mkdir tenants/tenant-c
cat > tenants/tenant-c/Chart.yaml <<'EOF'
apiVersion: v2
name: tenant-c
description: OpenFunction functions for tenant-c
type: application
version: 0.1.0
appVersion: "1.0.0"
dependencies:
  - name: function-template
    version: 0.1.0
    repository: file://../function-template
EOF

# copy values from tenant-a and edit namespace + functions
cp tenants/tenant-a/values.yaml tenants/tenant-c/values.yaml

cd tenants/tenant-c && helm dependency update
```

## Secrets

Secrets are namespace-scoped. Prefer:

```bash
./scripts/bootstrap-tenant-secrets.sh tenant-c-lambda
```

Or set `function-template.secrets.enabled: true` with Sealed Secrets / External Secrets — do not commit plaintext tokens.
