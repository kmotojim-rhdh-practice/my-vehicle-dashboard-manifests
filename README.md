# my-vehicle-dashboard-manifests

Kubernetesマニフェストリポジトリ

## 品質ゲートの強制

このリポジトリは**テストと静的解析を必須**としたCI/CDを含みます：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  CI Pipeline (Tekton)                                                       │
├─────────────┬─────────────┬─────────────┬─────────────┬─────────────────────┤
│  Git Clone  │  Unit Test  │  SonarQube  │   Buildah   │  Skopeo Copy        │
│             │  (必須)     │  (必須)     │             │                     │
└─────────────┴─────────────┴─────────────┴─────────────┴─────────────────────┘
                    │              │
                    ▼              ▼
              テスト失敗で    品質基準未達で
              パイプライン停止 パイプライン停止

┌─────────────────────────────────────────────────────────────────────────────┐
│  CD (ArgoCD + Post-Sync Hook)                                               │
├─────────────┬─────────────────────────────────────────────────────────────┤
│ ArgoCD Sync │  E2E Test (Cucumber + Playwright) ← 自動実行                 │
└─────────────┴─────────────────────────────────────────────────────────────┘
```

## ディレクトリ構成

```
my-vehicle-dashboard-manifests/
├── argocd/                    # ArgoCD Application定義
│   ├── application-dev.yaml   # 開発環境用Application
│   └── application-prod.yaml  # 本番環境用Application
├── base/                      # 基本のKubernetesマニフェスト
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   ├── frontend-route.yaml
│   └── e2e-test-job.yaml      # E2Eテスト（Post-Sync Hook）
├── overlays/                  # 環境別オーバーレイ
│   ├── dev/                   # 開発環境
│   │   └── kustomization.yaml
│   └── prod/                  # 本番環境
│       └── kustomization.yaml
└── pipelines/                 # Tekton CI Pipelines
    ├── kustomization.yaml
    ├── pvc.yaml
    ├── backend-ci-pipeline.yaml
    ├── frontend-ci-pipeline.yaml
    ├── backend-ci-pipelinerun.yaml
    ├── frontend-ci-pipelinerun.yaml
    └── README.md
```

## 前提条件

- OpenShiftクラスタ
- ArgoCD がインストール済み
- OpenShift Pipelines (Tekton) がインストール済み
- SonarQube がセットアップ済み
- コンテナイメージがレジストリにプッシュ済み

## 使用方法

### 1. CI Pipelineのセットアップ

```bash
# パイプラインリソースを適用
oc apply -k pipelines/ -n my-vehicle-dev

# 外部レジストリの認証情報を設定
oc create secret docker-registry quay-credentials \
  --docker-server=quay.io \
  --docker-username=<username> \
  --docker-password=<password> \
  -n my-vehicle-dev

oc secrets link pipeline quay-credentials --for=pull,mount -n my-vehicle-dev
```

### 2. CI Pipelineの実行

```bash
# バックエンドCIパイプライン（単体テスト + 静的解析 + ビルド）
tkn pipeline start my-vehicle-dashboard-backend-ci-pipeline \
  -p GIT_URL=<your-app-repo-url> \
  -p SONAR_HOST_URL=<your-sonarqube-url> \
  -p SONAR_PROJECT_KEY=my-vehicle-dashboard-backend \
  -p SONAR_TOKEN=<your-sonar-token> \
  -p EXTERNAL_REGISTRY_URL=<your-registry>/my-vehicle-dashboard-backend \
  -w name=workspace,claimName=pipeline-workspace-pvc \
  -n my-vehicle-dev

# フロントエンドCIパイプライン（静的解析 + ビルド）
tkn pipeline start my-vehicle-dashboard-frontend-ci-pipeline \
  -p GIT_URL=<your-app-repo-url> \
  -p SONAR_HOST_URL=<your-sonarqube-url> \
  -p SONAR_PROJECT_KEY=my-vehicle-dashboard-frontend \
  -p SONAR_TOKEN=<your-sonar-token> \
  -p EXTERNAL_REGISTRY_URL=<your-registry>/my-vehicle-dashboard-frontend \
  -w name=workspace,claimName=pipeline-workspace-pvc \
  -n my-vehicle-dev
```

### 3. ArgoCDへのApplicationの登録

開発環境:
```bash
kubectl apply -f argocd/application-dev.yaml
```

本番環境:
```bash
kubectl apply -f argocd/application-prod.yaml
```

### 4. E2Eテストの確認

E2Eテストは ArgoCD の Sync 後に自動実行されます。

```bash
# E2EテストJobの一覧
oc get jobs -n my-vehicle-dev -l app.kubernetes.io/name=e2e-test

# テストログの確認
oc logs -n my-vehicle-dev -l app.kubernetes.io/name=e2e-test --tail=100
```

### 5. 手動でのデプロイ確認（Kustomize）

開発環境のマニフェストをプレビュー:
```bash
kustomize build overlays/dev
```

本番環境のマニフェストをプレビュー:
```bash
kustomize build overlays/prod
```

直接適用する場合:
```bash
kubectl apply -k overlays/dev
```

## 環境別設定

| 設定項目 | 開発環境 (dev) | 本番環境 (prod) |
|---------|---------------|----------------|
| Namespace | my-vehicle-dev | my-vehicle-prod |
| Frontend Replicas | 1 | 2 |
| Backend Replicas | 1 | 2 |
| Image Tag | dev | latest |

## イメージレジストリ

デフォルトでは以下のイメージを使用します:
- Backend: `image-registry.openshift-image-registry.svc:5000/my-vehicle-dashboard/backend`
- Frontend: `image-registry.openshift-image-registry.svc:5000/my-vehicle-dashboard/frontend`

## なぜテストと静的解析を強制するのか

車載機・建機・鉄道などの制御システムでは、ソフトウェアの品質が人命に直結します。

| 品質ゲート | 目的 | 失敗時の動作 |
|-----------|------|-------------|
| 単体テスト | ロジックの正常動作を保証 | イメージビルドされない |
| 静的解析 | バグ・脆弱性を事前検出 | イメージビルドされない |
| E2Eテスト | ユーザー観点での動作保証 | デプロイがDegradedマーク |

これにより、品質基準を満たさないコードが本番環境にデプロイされることを防ぎます。
