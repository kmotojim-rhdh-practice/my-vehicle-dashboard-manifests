# CI/CDパイプライン

## パイプライン構成

### Backend CI Pipeline

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Git Clone  │───▶│  Unit Test  │───▶│  SonarQube  │───▶│   Buildah   │───▶│ Skopeo Copy │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                         │                  │
                         ▼                  ▼
                   テスト失敗で        品質基準未達で
                   パイプライン停止    パイプライン停止
```

| タスク | 説明 |
|-------|------|
| git-clone | GitHubからソースコードをクローン |
| unit-test | `mvn package` で単体テスト実行 + JARビルド |
| sonarqube-scan | SonarQubeで静的解析 + JaCoCoカバレッジレポート |
| buildah | Dockerイメージをビルド |
| skopeo-copy | 外部レジストリへイメージをコピー |

### Frontend CI Pipeline

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Git Clone  │───▶│  SonarQube  │───▶│   Buildah   │───▶│ Skopeo Copy │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

## パイプラインの実行

### tkn CLI での実行

```bash
# バックエンドCIパイプライン
tkn pipeline start my-vehicle-dashboard-backend-ci-pipeline \
  -p GIT_URL=<your-app-repo-url> \
  -p SONAR_HOST_URL=<your-sonarqube-url> \
  -p SONAR_PROJECT_KEY=my-vehicle-dashboard-backend \
  -p SONAR_TOKEN=<your-sonar-token> \
  -p EXTERNAL_REGISTRY_URL=<your-registry>/my-vehicle-dashboard-backend \
  -w name=workspace,claimName=pipeline-workspace-pvc \
  -n my-vehicle-dev

# フロントエンドCIパイプライン
tkn pipeline start my-vehicle-dashboard-frontend-ci-pipeline \
  -p GIT_URL=<your-app-repo-url> \
  -p SONAR_HOST_URL=<your-sonarqube-url> \
  -p SONAR_PROJECT_KEY=my-vehicle-dashboard-frontend \
  -p SONAR_TOKEN=<your-sonar-token> \
  -p EXTERNAL_REGISTRY_URL=<your-registry>/my-vehicle-dashboard-frontend \
  -w name=workspace,claimName=pipeline-workspace-pvc \
  -n my-vehicle-dev
```

## パイプライン実行の確認

```bash
# PipelineRunの一覧
oc get pipelinerun -n my-vehicle-dev

# ログの確認
tkn pipelinerun logs -f <pipelinerun-name> -n my-vehicle-dev
```

## E2Eテスト（ArgoCD Post-Sync Hook）

ArgoCDがSyncするたびに、E2Eテストが自動実行されます。

```bash
# E2EテストJobの一覧
oc get jobs -n my-vehicle-dev -l app.kubernetes.io/name=e2e-test

# テストログの確認
oc logs -n my-vehicle-dev -l app.kubernetes.io/name=e2e-test --tail=100
```
