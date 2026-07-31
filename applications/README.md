# GitOps — single company (plain Function YAML)

Deploy OpenFunction functions for **one company** onto **that company's cluster**.

- 1 Argo CD `Application` (no ApplicationSet)
- 1 namespace (`lambda`)
- Plain `Function` YAML files — **no Helm template**
- Secrets created once with kubectl

## Layout

```
gitops/applications/
├── function-1.yaml
├── function-2.yaml
└── README.md
```

Argo CD syncs every `*.yaml` in this folder into namespace `lambda`.

## Secrets (create once)

```bash
kubectl create namespace lambda --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret docker-registry push-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<user> --docker-password=<token> \
  -n lambda

kubectl create secret generic git-repo-secret \
  --from-literal=username=<user> --from-literal=password=<pat> \
  -n lambda

kubectl annotate secret push-secret git-repo-secret \
  -n lambda build.shipwright.io/referenced.secret=true
```

## Add a function

Copy `function-1.yaml` → `function-3.yaml`, edit name / image / srcRepo, then:

```bash
git add applications/function-3.yaml && git commit -m "Add function-3" && git push
```

## Update after code change

CI cập nhật `spec.build.srcRepo.revision` (hoặc `spec.image`) trong file YAML tương ứng, rồi push.

## Argo CD Application

```bash
kubectl apply -f helm/argocd/applications/company-functions.yaml
```

## When to use Helm template again

Chỉ khi cần multi-tenant (nhiều namespace, defaults chung, External Secrets).
Tham khảo `../function-template/` và `../tenants/`.
