# セットアップ

## 前提条件

- OpenShiftクラスタ
- ArgoCD がインストール済み
- OpenShift Pipelines Operator がインストール済み
- SonarQube がセットアップ済み

### 確認コマンド

```bash
# OpenShift Pipelines Operatorの確認
oc get csv -n openshift-operators | grep pipelines

# ArgoCDの確認
oc get pods -n openshift-gitops
```

## 1. パイプラインのデプロイ

```bash
# Namespaceとパイプラインを作成
oc apply -k pipelines/
```

## 2. 外部レジストリ認証の設定

```bash
# Quay.ioの認証情報を設定
oc create secret docker-registry quay-credentials \
  --docker-server=quay.io \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n my-vehicle-dev

# pipeline ServiceAccountにリンク
oc secrets link pipeline quay-credentials --for=pull,mount -n my-vehicle-dev
```

## 3. ArgoCDアプリケーションの登録

開発環境:
```bash
oc apply -f argocd/application-dev.yaml
```

本番環境:
```bash
oc apply -f argocd/application-prod.yaml
```

## 4. デプロイの確認

```bash
# ArgoCD Applicationの状態確認
oc get applications -n openshift-gitops

# Podの状態確認
oc get pods -n my-vehicle-dev
```
