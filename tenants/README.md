# Tenant GitOps layout

Functions are deployed via Helm charts and synced by Argo CD. Two deployment modes are supported depending on how many namespaces you use.

## Deployment modes

### Single namespace (simple)

All functions live in one namespace (e.g. `default` or `lambda`). Kubernetes Secrets are namespace-scoped, so you only need to create `push-secret` and `git-repo-secret` **once** in that namespace. External Secrets Operator is not required.

```yaml
# function-template/values.yaml (or per-tenant override)
externalSecrets:
  enabled: false
```

Create secrets once:

```bash
kubectl create secret docker-registry push-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<user> --docker-password=<token> \
  -n <namespace>

kubectl create secret generic git-repo-secret \
  --from-literal=username=<user> --from-literal=password=<pat> \
  -n <namespace>

kubectl annotate secret push-secret git-repo-secret \
  -n <namespace> build.shipwright.io/referenced.secret=true
```

### Multi-tenant / multi-namespace

Each tenant gets its own folder, Helm chart, and namespace (e.g. `tenant-a-lambda`, `tenant-b-lambda`). Because secrets do not cross namespace boundaries, every namespace needs its own copy of `push-secret` and `git-repo-secret`.

For many tenants, use [External Secrets Operator](../secrets/README.md) to sync credentials from AWS Secrets Manager (local) or Vault (production) into each namespace automatically:

```yaml
externalSecrets:
  enabled: true
```

| Mode | Namespaces | Secrets |
|------|------------|---------|
| Single namespace | 1 | Create once manually — ESO not needed |
| Multi-tenant | 1 per tenant | ESO replicates to each namespace |

## How it works (multi-tenant)

```
Git repo (tenants/)
        │
        ▼
Argo CD ApplicationSet          one Application per tenants/* folder
        │
        ├── tenant-a  →  namespace tenant-a-lambda
        ├── tenant-b  →  namespace tenant-b-lambda
        └── tenant-c  →  namespace tenant-c-lambda
                │
                ▼
        Helm renders Function CRs (+ ExternalSecrets if enabled)
                │
                ▼
        OpenFunction builds and serves each function
```

| Component | Role |
|-----------|------|
| `function-template/` | Shared Helm chart — templates for `Function` CRs and optional `ExternalSecret` resources. Not deployed directly. |
| `tenant-*/` | Thin chart per tenant — depends on `function-template`, only contains `Chart.yaml` + `values.yaml`. |
| ApplicationSet | Watches `tenants/*`, creates one Argo CD Application per tenant folder. |
| `{tenant}-lambda` | Target namespace for all functions belonging to that tenant. |

## Directory structure

```
tenants/
├── function-template/          # shared chart (excluded from ApplicationSet)
│   ├── Chart.yaml
│   ├── values.yaml             # defaults: builder, runtime, secret names
│   └── templates/
│       ├── function.yaml       # renders OpenFunction Function CRs
│       ├── external-secret.yaml  # only rendered when externalSecrets.enabled
│       └── secret.yaml         # legacy inline secrets (disabled by default)
├── tenant-a/
│   ├── Chart.yaml              # depends on function-template
│   ├── values.yaml             # tenant-a namespace + function list
│   └── charts/                 # packaged dependency (run helm dependency update)
└── tenant-b/
    ├── Chart.yaml
    └── values.yaml
```

Tenant `values.yaml` is nested under the `function-template:` key so overrides map directly onto the subchart:

```yaml
function-template:
  namespace:
    name: tenant-a-lambda
  externalSecrets:
    enabled: true   # false if all functions share one namespace
  functions:
    - name: function-1
      version: "v1.0.0"
      image: "registry/image:v1"
      build:
        srcRepo:
          url: "https://github.com/org/repo.git"
          sourceSubPath: "tenant-a/function-1"
          revision: "<commit-sha>"
```

## Add a tenant

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

cp tenants/tenant-a/values.yaml tenants/tenant-c/values.yaml
# edit namespace.name → tenant-c-lambda and functions list

cd tenants/tenant-c && helm dependency update
git add tenants/tenant-c && git push
```

Argo CD picks up the new folder and creates `tenant-c-functions` targeting namespace `tenant-c-lambda` (`CreateNamespace=true`).

## Deploying function updates

CI pipelines update `values.yaml` — typically `build.srcRepo.revision` or `image` — and push to Git. Argo CD syncs the change, OpenFunction detects the updated `Function` CR and triggers a rebuild.

Each function gets a **stable external URL** that does not change on rebuild:

```
http://{function-name}.{namespace}.ofn.io/
# e.g. http://function-1.tenant-a-lambda.ofn.io/
```

## Secrets (multi-tenant only)

When `externalSecrets.enabled: true`, ESO fetches credentials from an external store and creates `push-secret` and `git-repo-secret` in each tenant namespace. See [gitops/secrets/README.md](../secrets/README.md) for setup.

| Environment | Store | Config |
|-------------|-------|--------|
| Local / staging | AWS Secrets Manager | `externalSecrets.storeRef.name: aws-secretsmanager` |
| Production | HashiCorp Vault | `externalSecrets.storeRef.name: vault` |

After changing `function-template/`, rebuild tenant chart dependencies before pushing:

```bash
cd tenants/tenant-a && helm dependency update
cd tenants/tenant-b && helm dependency update
```

Argo CD renders from the packaged `charts/*.tgz`, not the live `function-template/` folder.

## Local validation

```bash
cd tenants/tenant-a
helm dependency update
helm template .          # dry-run render
```
