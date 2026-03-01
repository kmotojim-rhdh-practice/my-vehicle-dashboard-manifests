# my-vehicle-dashboard - Tekton CI Pipelines

my-vehicle-dev 環境でCI/CDを実行するためのTekton Pipelinesです。

## 品質ゲートの強制

このパイプラインは**テストと静的解析を必須**としています：

| ステップ | 失敗時の動作 | 目的 |
|---------|-------------|------|
| Unit Test | パイプライン停止 | コードの正常動作を保証 |
| SonarQube | パイプライン停止 | コード品質・セキュリティを保証 |
| Image Build | 実行されない | 品質基準を満たしたコードのみビルド |
| E2E Test | デプロイ失敗とマーク | ユーザー観点での動作を保証 |

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Git Clone  │───▶│  Unit Test  │───▶│  SonarQube  │───▶│   Buildah   │───▶│ Skopeo Copy │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                         │                  │
                         ▼                  ▼
                   テスト失敗で        品質基準未達で
                   パイプライン停止    パイプライン停止
```

## パイプライン構成

### Backend CI Pipeline
- **単体テスト** (Maven): `mvn test` で全テストを実行
- **静的解析** (SonarQube): コード品質・バグ・脆弱性を検査
- **イメージビルド** (Buildah): Dockerイメージを作成
- **イメージコピー** (Skopeo): 外部レジストリへプッシュ

### Frontend CI Pipeline
- **静的解析** (SonarQube): コード品質を検査
- **イメージビルド** (Buildah): Dockerイメージを作成
- **イメージコピー** (Skopeo): 外部レジストリへプッシュ

### E2E Test (ArgoCD Post-Sync Hook)
- **自動実行**: ArgoCDデプロイ完了後に自動起動
- **Cucumber + Playwright**: Gherkin記法でのGUIテスト
- **失敗時**: デプロイがDegradedとしてマーク

## 前提条件

### 1. OpenShift Pipelinesのインストール
```bash
oc get csv -n openshift-operators | grep pipelines
```

### 2. SonarQubeの準備
```bash
# SonarQubeでプロジェクトを作成し、トークンを発行
# プロジェクトキー: my-vehicle-dashboard-backend, my-vehicle-dashboard-frontend
```

### 3. 外部レジストリ認証の設定
```bash
# Quay.ioなどの認証情報を設定
oc create secret docker-registry quay-credentials \
  --docker-server=quay.io \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n my-vehicle-dev

# pipeline ServiceAccountにリンク
oc secrets link pipeline quay-credentials --for=pull,mount -n my-vehicle-dev
```

## デプロイ

```bash
# パイプラインリソースを適用
oc apply -k pipelines/ -n my-vehicle-dev
```

## パイプラインの実行

### 方法1: PipelineRunを編集して実行

```bash
# backend-ci-pipelinerun.yaml のプレースホルダーを編集
# - <YOUR_APP_GIT_URL>: アプリケーションのGitリポジトリURL
# - <YOUR_SONARQUBE_URL>: SonarQubeサーバーのURL
# - <YOUR_SONAR_TOKEN>: SonarQubeの認証トークン
# - <YOUR_EXTERNAL_REGISTRY>: 外部レジストリ（例: quay.io/myorg）

oc create -f pipelines/backend-ci-pipelinerun.yaml
oc create -f pipelines/frontend-ci-pipelinerun.yaml
```

### 方法2: tkn CLIで実行

```bash
# バックエンドCIパイプライン
tkn pipeline start my-vehicle-dashboard-backend-ci-pipeline \
  -p GIT_URL=https://github.com/your-org/your-app.git \
  -p SONAR_HOST_URL=http://sonarqube.example.com:9000 \
  -p SONAR_PROJECT_KEY=my-vehicle-dashboard-backend \
  -p SONAR_TOKEN=<your-token> \
  -p EXTERNAL_REGISTRY_URL=quay.io/your-org/my-vehicle-dashboard-backend \
  -w name=workspace,claimName=pipeline-workspace-pvc \
  -n my-vehicle-dev

# フロントエンドCIパイプライン
tkn pipeline start my-vehicle-dashboard-frontend-ci-pipeline \
  -p GIT_URL=https://github.com/your-org/your-app.git \
  -p SONAR_HOST_URL=http://sonarqube.example.com:9000 \
  -p SONAR_PROJECT_KEY=my-vehicle-dashboard-frontend \
  -p SONAR_TOKEN=<your-token> \
  -p EXTERNAL_REGISTRY_URL=quay.io/your-org/my-vehicle-dashboard-frontend \
  -w name=workspace,claimName=pipeline-workspace-pvc \
  -n my-vehicle-dev
```

## パイプライン実行の確認

```bash
# PipelineRunの一覧
oc get pipelinerun -n my-vehicle-dev

# 実行中のログ
tkn pipelinerun logs -f <pipelinerun-name> -n my-vehicle-dev

# 状態確認
tkn pipelinerun describe <pipelinerun-name> -n my-vehicle-dev
```

## E2Eテストの確認

```bash
# E2EテストJobの一覧
oc get jobs -n my-vehicle-dev -l app.kubernetes.io/name=e2e-test

# E2EテストPodのログ
oc logs -n my-vehicle-dev -l app.kubernetes.io/name=e2e-test --tail=100

# 最新のE2EテストPodのログ全文
oc logs -n my-vehicle-dev $(oc get pods -n my-vehicle-dev -l app.kubernetes.io/name=e2e-test --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
```

## パラメータ一覧

| パラメータ | 説明 | 必須 |
|-----------|------|:----:|
| GIT_URL | アプリケーションのGitリポジトリURL | ✅ |
| GIT_REVISION | Gitブランチ/タグ (デフォルト: main) | |
| SOURCE_PATH | ソースコードパス (backend/frontend) | |
| SONAR_HOST_URL | SonarQubeサーバーURL | ✅ |
| SONAR_PROJECT_KEY | SonarQubeプロジェクトキー | ✅ |
| SONAR_TOKEN | SonarQube認証トークン | ✅ |
| INTERNAL_REGISTRY_URL | 内部レジストリURL | |
| EXTERNAL_REGISTRY_URL | 外部レジストリURL | ✅ |
| IMAGE_TAG | イメージタグ (デフォルト: latest) | |

## なぜテストと静的解析を強制するのか

車載機・建機・鉄道などの制御システムでは、ソフトウェアの品質が人命に直結します。
このパイプラインは以下を保証します：

1. **単体テスト必須**: ロジックの正常動作を保証
2. **静的解析必須**: バグ・脆弱性・コードスメルを事前検出
3. **E2Eテスト必須**: ユーザー操作シナリオの動作を保証
4. **品質ゲート**: 基準を満たさないコードはビルドされない

これにより、チーム全体に品質文化を定着させ、本番環境への不具合混入を防ぎます。
