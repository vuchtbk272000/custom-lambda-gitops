# GitOps — Function manifests

Thư mục này chứa các OpenFunction `Function` YAML được Argo CD đồng bộ vào namespace `serverless-functions`.

## Cấu trúc

```
applications/
├── function-1.yaml
├── function-2.yaml
└── README.md
```

## Secret

```bash
kubectl create namespace serverless-functions --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret docker-registry push-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<user> --docker-password=<token> \
  -n serverless-functions

kubectl create secret generic git-repo-secret \
  --from-literal=username=<user> --from-literal=password=<pat> \
  -n serverless-functions

kubectl annotate secret push-secret git-repo-secret \
  -n serverless-functions build.shipwright.io/referenced.secret=true
```

## Thêm function

1. Thêm file `applications/function-3.yaml` (chỉnh `name`, `image`, `sourceSubPath`).
2. Trên repo source: tạo `function-3/gitops.yaml` với `path: applications/function-3.yaml`.
3. Đẩy cả hai repository.

## Cập nhật sau khi đổi code

GitHub Actions trên repo source cập nhật `spec.build.srcRepo.revision` (yêu cầu secret `GITOPS_TOKEN`). Chi tiết: tài liệu setup, mục CI.

## Argo CD Application

```bash
kubectl apply -f applications/openfunction-functions.yaml
```
