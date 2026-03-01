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
├─────────────┬───────────────────────────────────────────────────────────────┤
│ ArgoCD Sync │  E2E Test (Cucumber + Playwright) ← 自動実行                  │
└─────────────┴───────────────────────────────────────────────────────────────┘
```

## なぜテストと静的解析を強制するのか

車載機・建機・鉄道などの制御システムでは、ソフトウェアの品質が人命に直結します。

| 品質ゲート | 目的 | 失敗時の動作 |
|-----------|------|-------------|
| 単体テスト | ロジックの正常動作を保証 | イメージビルドされない |
| 静的解析 (SonarQube) | バグ・脆弱性を事前検出 | イメージビルドされない |
| E2Eテスト | ユーザー観点での動作保証 | デプロイがDegradedマーク |

## ディレクトリ構成

```
my-vehicle-dashboard-manifests/
├── argocd/                    # ArgoCD Application定義
│   ├── application-dev.yaml   # 開発環境用Application
│   └── application-prod.yaml  # 本番環境用Application
├── base/                      # 基本のKubernetesマニフェスト
│   └── e2e-test-job.yaml      # E2Eテスト（Post-Sync Hook）
├── overlays/                  # 環境別オーバーレイ
│   ├── dev/
│   └── prod/
└── pipelines/                 # Tekton CI Pipelines
```

## 環境別設定

| 設定項目 | 開発環境 (dev) | 本番環境 (prod) |
|---------|---------------|----------------|
| Namespace | my-vehicle-dev | my-vehicle-prod |
| Frontend Replicas | 1 | 2 |
| Backend Replicas | 1 | 2 |
| Image Tag | dev | latest |
