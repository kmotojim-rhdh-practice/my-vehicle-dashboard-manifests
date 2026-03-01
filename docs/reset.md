# リセット手順

本アプリケーションはデモ用のため、何度でも実行できるようリセット手順を提供します。

## クイックリセット（アプリケーションのみ）

```bash
# ArgoCDアプリケーションを再同期
argocd app sync my-vehicle-dashboard-dev --force

# または、Deploymentを再起動
oc rollout restart deployment/my-vehicle-dashboard-frontend -n my-vehicle-dev
oc rollout restart deployment/my-vehicle-dashboard-backend -n my-vehicle-dev
```

## PipelineRunの削除

```bash
# 全てのPipelineRunを削除
oc delete pipelinerun --all -n my-vehicle-dev
```

## E2Eテスト結果の削除

```bash
oc delete job -l app.kubernetes.io/name=e2e-test -n my-vehicle-dev
```

## フルリセット（Namespace削除）

```bash
# 1. ArgoCDアプリケーションを削除
argocd app delete my-vehicle-dashboard-dev --cascade=false

# 2. Namespaceを削除
oc delete namespace my-vehicle-dev

# 3. 待機
oc wait --for=delete namespace/my-vehicle-dev --timeout=120s

# 4. パイプラインを再デプロイ（Namespaceも作成されます）
oc apply -k pipelines/

# 5. 外部レジストリ認証を再設定
oc create secret docker-registry quay-credentials \
  --docker-server=quay.io \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n my-vehicle-dev
oc secrets link pipeline quay-credentials --for=pull,mount -n my-vehicle-dev

# 6. ArgoCDアプリケーションを再登録
oc apply -f argocd/application-dev.yaml
```

## 本番環境のリセット

```bash
argocd app delete my-vehicle-dashboard-prod --cascade=false
oc delete namespace my-vehicle-prod
oc wait --for=delete namespace/my-vehicle-prod --timeout=120s
oc apply -f argocd/application-prod.yaml
```

## Finalizerが原因で削除できない場合

```bash
oc patch application my-vehicle-dashboard-dev -n openshift-gitops \
  -p '{"metadata":{"finalizers":null}}' --type=merge
```
