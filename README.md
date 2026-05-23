# platform-gitops

Platform Engineering portfolio — GitOps management repository.

## 概要

ArgoCDのソースとなるリポジトリ。プラットフォーム全体の「あるべき姿」を管理する。

## ディレクトリ構成
 
```
platform-gitops/
├── bootstrap/
│   ├── root.yaml          # platform/ 配下を再帰的に監視する App-of-Apps
│   └── apps-root.yaml     # apps/ 配下を監視する App-of-Apps
├── platform/              # プラットフォームコンポーネント（root が自動検出）
│   ├── argocd/
│   ├── cilium/
│   ├── cnpg/
│   ├── crossplane/
│   │   └── config/        # provider / providerconfig / release CR（root 除外）
│   ├── gateway/
│   │   └── config/        # HTTPRoute / ReferenceGrant（root 除外）
│   ├── ingress/
│   │   └── config/        # ClusterIssuer（root 除外）
│   ├── logging/
│   ├── monitoring/
│   ├── policy/
│   │   └── policies/      # Kyverno ポリシー（root 除外）
│   ├── scaling/
│   ├── secrets/
│   │   └── config/        # ClusterSecretStore / ExternalSecret（root 除外）
│   ├── security/
│   ├── tracing/
│   └── vpa/
├── apps/                  # アプリケーション（apps-root が管理）
│   ├── sample-backend.yaml
│   ├── sample-backend/
│   │   ├── values.yaml
│   │   └── manifests/     # ScaledObject など追加リソース
│   ├── sample-frontend.yaml
│   └── sample-frontend/
│       └── values.yaml
└── secrets/
    └── encrypted/         # SOPS × Age で暗号化した Secret
```

---

## Phase 2: GitOps & Secrets

### このPhaseで解決すること
ArgoCDによる「Gitの状態＝クラスタの状態」の実現と、
SecretをGitに直接置かないセキュアな基盤の構築。

### 構成コンポーネント
| コンポーネント | namespace | 役割 |
|---|---|---|
| ArgoCD | argocd | GitリポジトリをソースにK8sリソースを自動同期 |
| External Secrets Operator | external-secrets | 外部ストアからK8s Secretを自動生成 |

### ArgoCDへのアクセス
```bash
# 初期パスワードの取得
argocd admin initial-password -n argocd

# UIへのアクセス（Ingress経由 / Phase 3以降）
# https://argocd.localhost

# CLIログイン
argocd login localhost:8080 --username admin --insecure
```

### App of Appsパターン
`bootstrap/root.yaml` を適用するだけで、`platform/` 以下の
全コンポーネントがArgoCDの管理下に入る。

```bash
kubectl apply -f bootstrap/root.yaml
argocd app list
```

### External Secrets Operatorの動作確認
```bash
# ClusterSecretStoreの確認
kubectl get clustersecretstore

# ExternalSecretの確認
kubectl get externalsecret -A
```

### 設計上の決定事項
- ArgoCDはHelmでインストールし、設定値を `platform/argocd/values.yaml` で管理。
- シークレットストアはローカル環境用に `Fake` プロバイダーを使用。Phase 8でAWS Secrets Managerに切り替え予定。
- PrivateリポジトリへのアクセスはSSH鍵で設定（`argocd repo add --ssh-private-key-path`）。
- APIバージョンは `external-secrets.io/v1`（v1beta1は非推奨）。

---

## Phase 3: Connectivity

### このPhaseで解決すること
Ingress-nginx と cert-manager を導入し、`*.localhost` で即座にHTTPSサービスを公開できる基盤を構築する。
自己署名CAによる証明書の自動発行により、開発者はTLS設定を意識せずにサービスを公開できる。

### 構成コンポーネント
| コンポーネント | namespace | Helmチャート | バージョン |
|---|---|---|---|
| ingress-nginx | ingress-nginx | ingress-nginx/ingress-nginx | 4.15.1 |
| cert-manager | cert-manager | jetstack/cert-manager | v1.20.2 |

### ClusterIssuerの構成
| リソース | 役割 |
|---|---|
| `selfsigned-issuer` | 自己署名でCA証明書を発行するIssuer |
| `local-ca`（Certificate） | selfsigned-issuerが発行したCA証明書。`cert-manager` namespaceの `local-ca-secret` に格納 |
| `local-ca-issuer` | `local-ca-secret` を使って各サービスの証明書を発行するIssuer |

### サービス公開の手順
Ingressマニフェストに以下のアノテーションを付与するだけで証明書が自動発行される。

```yaml
annotations:
  cert-manager.io/cluster-issuer: local-ca-issuer
```

### 設計上の決定事項
- ingress-nginx の Admission Webhook（`ingress-nginx-admission`）は削除済み。k3dローカル環境では証明書検証エラーが発生するため。
- external-secrets の Application には `ServerSideApply=true` を設定。CRDのサイズ超過エラーを回避するため。
- ingress-nginx のリタイアについては Phase 11（EKS）で Gateway API（Envoy Gateway）への移行を検討する。

---

## Phase 4: Observability

### このPhaseで解決すること
メトリクス・ログ・トレースの3本柱を揃え、サービスの内部状態を
「何が起きているか」を後から追跡できる基盤を構築する。

### 構成コンポーネント（LGTMスタック）
| コンポーネント | namespace | Helmチャート | バージョン |
|---|---|---|---|
| kube-prometheus-stack | monitoring | prometheus-community/kube-prometheus-stack | 83.6.0 |
| Loki | logging | grafana-community/loki | 6.55.0 |
| Grafana Alloy | logging | grafana/alloy | 1.7.0 |
| Tempo | tracing | grafana/tempo | 1.24.4 |

### アクセス
```
https://grafana.localhost
```

### 設計上の決定事項
- Grafana / Prometheus / Loki / Tempo を Grafana データソースとして統合し、ログ・メトリクス・トレースをひとつの画面で横断できる。
- `ServiceMonitor` を `common-app` Library Chart に組み込み、`serviceMonitor.enabled: true` だけでアプリの自動収集が完了する。
- Alloy（旧 Grafana Agent）でログ収集。Podラベルをベースにラベルを付与し、Lokiへ転送する。

---

## Phase 5: Platform Abstraction

### このPhaseで解決すること
アプリ開発者が Kubernetes マニフェストを直接書かなくてもデプロイできる「Golden Path」の入口を構築する。
`platform-charts` の Library Chart を `apps/` 配下の Application が参照する形で利用する。

### 構成コンポーネント
| Helmチャート | 役割 |
|---|---|
| `platform-charts/common-app` | Deployment / Service / Ingress / HPA / ServiceMonitor を一元管理 |
| `platform-charts/common-db` | CloudNativePG クラスタ / バックアップスケジュールを一元管理 |

### apps/ の構成
```
apps/
├── sample-backend/
│   ├── application.yaml    # ArgoCD Application リソース
│   └── values.yaml         # common-app + common-db の設定値
└── sample-frontend/
    ├── application.yaml
    └── values.yaml
```

---

## Phase 6: Golden Path（CI/CD）

### このPhaseで解決すること
アプリ Repo の push から ArgoCD による自動デプロイまでを、
開発者がプラットフォームを意識せずに完結できる CI/CD フローを構築する。

### フロー
```
push to main（sample-backend / sample-frontend）
  └─► GitHub Actions: build.yaml
        ├─ イメージビルド & GHCR プッシュ（タグ: git SHA short）
        └─► update-gitops.yaml
              └─► platform-gitops に repository_dispatch を送信
                    └─► .github/workflows/update-image.yaml
                          └─► apps/{service}/values.yaml の image.tag を更新して PR
                                └─► ArgoCD が自動同期
```

### 設計上の決定事項
- image tag の更新は PR 経由とし、デプロイの履歴を Git に残す。
- クロスリポジトリの `repository_dispatch` は `okccl-gitops` GitHub App（`GITOPS_APP_CLIENT_ID` / `GITOPS_APP_PRIVATE_KEY`）で認証する。当初は PAT（`GITOPS_TOKEN`）を使用していたが GitHub App に移行済み。

---

## Phase 7: Guardrail（Kyverno）

### このPhaseで解決すること
ポリシーエンジン Kyverno によるガードレールの導入。
設定ミスやルール違反を「デプロイ前」に検出し、「誰が操作しても安全」な環境を実現する。

### 構成コンポーネント
| コンポーネント | namespace | Helmチャート | バージョン |
|---|---|---|---|
| Kyverno | kyverno | kyverno/kyverno | 3.7.1 |

### 適用ポリシー（`platform/policy/policies/`）
| ポリシー | 種別 | 内容 |
|---|---|---|
| `require-resource-limits.yaml` | Validate | CPU/Memoryの `limits` 未設定のPodをブロック |
| `disallow-latest-tag.yaml` | Validate | `image: xxx:latest` タグのPodをブロック |
| `add-default-labels.yaml` | Mutate | `app.kubernetes.io/name` ラベルを自動付与 |

### 設計上の決定事項
- ポリシー違反は ArgoCD の Sync 失敗として検出される。デプロイパイプラインの一部としてガードレールが機能する。
- `common-app` Library Chart は全ポリシーに準拠した値をデフォルトで持つため、Library Chart 経由でデプロイすれば自動的に通過する。

---

## Phase 8: Data & State

### このPhaseで解決すること
DB の「信頼できる状態」を管理する Library Chart を作り、
データの永続化・バックアップ・リストアをコード化する。

### 構成コンポーネント
| コンポーネント | namespace | Helmチャート | バージョン |
|---|---|---|---|
| CloudNativePG (CNPG) Operator | cnpg-system | cnpg/cloudnative-pg | 0.28.0 |

### 設計上の決定事項
- `common-db` Library Chart で CNPG Cluster を抽象化。`instances: 2` 以上でHA構成になる。
- バックアップ先はS3互換ストレージ（Phase 10で外部MinIOに移行）。
- `backup.enabled: true` にするだけで定期バックアップが有効になる。

---

## Phase 9: Resilience & Chaos

### このPhaseで解決すること
Nodeの障害時にサービスが継続できることを実際に障害を起こして検証する。

### 構成コンポーネント
| コンポーネント | namespace | 役割 |
|---|---|---|
| VPA | vpa | リソース上限の最適値を推定 |
| Goldilocks | goldilocks | VPA推定値をダッシュボードで可視化 |

### Chaos Engineering の検証
```bash
# ノード障害シミュレーション
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# CNPG フェイルオーバーの確認
kubectl get cluster -n <namespace> -w

# 復旧
kubectl uncordon <node-name>
```

### 設計上の決定事項
- `podAntiAffinity` を values に設定し、同一 Pod が同一 Node に集中しないことを保証する。
- Chaos Engineering の結果として DB Primary の切替（Switchover）の RTO を計測・記録した。

---

## Phase 10: DX & Secrets as Code

### このPhaseで解決すること
SecretのコードとしてのGit管理と、1コマンドで再現できるDR基盤の整備。

### SOPS × Age による Secrets as Code
Git 上の Secret は SOPS で暗号化し、Age キーを持つ開発者のみが復号できる。
`make bootstrap` 実行時に自動復号・適用されるため、手動のSecret投入作業がゼロになった。

```bash
# Secret の復号確認
sops --decrypt secrets/encrypted/minio-backup-secret.yaml
```

### Devcontainer によるオンボーディング最小化
`.devcontainer` と `mise` を組み合わせ、`git clone` 直後の環境から
標準化されたツールセットを最短で揃えられるようにする。

### 設計上の決定事項
- Age キーファイル（`~/.config/sops/age/keys.txt`）のみがGit管理外の唯一の前提条件。
- SOPS ルール（`.sops.yaml`）でファイルパスに応じた暗号化キーを自動選択する。
- `make bootstrap` に Secret の自動復号・投入を組み込み、手動作業をゼロにした。

---

## Phase 10.5: DR演習・sync-wave導入・外部MinIO移行

### このPhaseで解決すること
実際にクラスタを全消去し、完全自動復旧を実証する。
また演習を通じて発見した設計上の問題を修正し、より堅牢なプラットフォームに改善する。

### RTO計測結果

| 指標 | 時間 |
|---|---|
| RTO① `make bootstrap` 完了 | **7分37秒** |
| RTO② 全App Synced/Healthy | **15分24秒** |
| 手動作業 | 新規マシンの場合のAgeキーコピーのみ |

### ArgoCD sync-wave の導入

root appが管理する全Applicationにsync-waveを付与し、CRD依存の順序問題を構造的に解消した。

### CRD依存リソースの分離

root appが直接管理するとCRDが存在しない状態でSyncが失敗する問題を解消するため、
CRD依存リソースを専用Applicationに分離した。

| Application | path | 管理するリソース |
|---|---|---|
| `cert-manager-config` | `platform/ingress/config/` | ClusterIssuer, Certificate |
| `external-secrets-config` | `platform/secrets/config/` | ClusterSecretStore, ClusterExternalSecret |

root appのexcludeにこれらのpathを追加し、直接管理を廃止した。

```yaml
# bootstrap/root.yaml
directory:
  recurse: true
  exclude: "{policy/policies/*,ingress/config/*,secrets/config/*}"
```

### 外部MinIOへの移行

クラスタ内MinIOではクラスタを消去するとバックアップも消滅するため、
WSL上のDockerコンテナとして外部MinIOを構築し、CNPGのバックアップ先を変更した。

```yaml
# apps/sample-backend/values.yaml
db:
  backup:
    endpointURL: "http://host.k3d.internal:9000"  # 外部MinIO
```

クラスタ内MinIO（`platform/minio/`）は廃止済み。

### 設計上の決定事項
- sync-wave導入により、`make bootstrap` 実行後は完全自動でSynced/Healthyになる。
- Kyvernoの `resources.limits` 必須ポリシーがCNPG recoveryをブロックする問題を発見。recoveryマニフェストには必ずresourcesを明示する。
- クラスタ内バックアップストレージはDR対象外になるため、本番・検証環境では必ずクラスタ外に置く。
- Phase 13（EKS移行）でバックアップ先をAWS S3に移行し、真のクラウドDRを実現予定。

---
 
## Phase 11: Hardening & Exploration
 
### Phase 11-1: Gateway API（Envoy Gateway）
 
ingress-nginx を廃止し、Envoy Gateway v1.7.2 に移行。
 
### 構成コンポーネント
| コンポーネント | namespace | 管理方法 | バージョン |
|---|---|---|---|
| Envoy Gateway | envoy-gateway-system | Rendered Manifests | v1.7.2 |
 
### 公開サービス一覧
| サービス | ホスト名 |
|---|---|
| sample-backend | sample-backend.localhost |
| sample-frontend | sample-frontend.localhost |
| ArgoCD | argocd.localhost |
| Grafana | grafana.localhost |
| Goldilocks | goldilocks.localhost |
| Argo Rollouts Dashboard | rollouts.localhost |
 
### 設計上の決定事項
- ArgoCDからDockerHub OCIレジストリへの直接参照が認証失敗となるため、`helm template` でレンダリングしたYAMLをGitに格納する **Rendered Manifests Pattern** を採用。
- HTTPRoute + ReferenceGrant によるクロスNamespaceルーティングを実装。
- TLS対応は Phase 11 後半に実施予定。ローカル確認は Edge または curl を使用する（Chrome は HTTP → HTTPS リダイレクトがあるため）。
---
 
### Phase 11-2: Trivy Operator
 
クラスタ全体のワークロードを継続スキャンし、脆弱性レポートを CRD として自動生成する。
 
### 構成コンポーネント
| コンポーネント | namespace | Helmチャート | バージョン |
|---|---|---|---|
| Trivy Operator | trivy-system | aquasecurity/trivy-operator | 0.32.1 |
 
### 動作確認結果（初回スキャン）
| Namespace | Critical | High |
|---|---|---|
| goldilocks | 5 | 24 |
| argocd | 4 | 25 |
| kube-system | 4〜7 | 21〜35 |
| cnpg-system | 2 | 10 |
| sample-app | 0 | 確認済み |
 
### 動作確認
```bash
# 脆弱性レポートの確認
kubectl get vulnerabilityreports --all-namespaces
 
# 設定監査レポートの確認
kubectl get configauditreports --all-namespaces
 
# サマリー（Namespace別 Critical/High 件数）
kubectl get vulnerabilityreports --all-namespaces -o json \
  | jq -r '.items[] | [.metadata.namespace, .metadata.name, .report.summary.criticalCount, .report.summary.highCount] | @tsv' \
  | sort
```
 
### 設計上の決定事項
- `excludeNamespaces: "kube-system,trivy-system"` でスキャン除外対象を設定。
- `ignoreUnfixed: true` で修正版が存在しない脆弱性を除外し、対応可能なものに集中。
- ServiceMonitor を有効化し、kube-prometheus-stack と連携。
---
 
### Phase 11-3: Argo Rollouts
 
Deployment をカナリア戦略の `Rollout` リソースに移行し、段階的なデプロイを実現する。
 
### 構成コンポーネント
| コンポーネント | namespace | Helmチャート | バージョン |
|---|---|---|---|
| Argo Rollouts | argo-rollouts | argo/argo-rollouts | 2.40.9 |
 
### カナリア戦略（sample-backend）
```yaml
rollout:
  enabled: true
  canary:
    steps:
      - setWeight: 20   # 20%のトラフィックをカナリアへ
      - pause: {}       # 手動 promote まで停止
      - setWeight: 100  # 100%切り替え
```
 
### 動作確認
```bash
# Rollout の状態確認
kubectl argo rollouts get rollout sample-backend -n sample-app
 
# カナリアの promote
kubectl argo rollouts promote sample-backend -n sample-app
```
 
### 設計上の決定事項
- `common-app` Library Chart に `Rollout` テンプレートを追加（v0.2.0）。`rollout.enabled` フラグで Deployment と切り替え可能にし、Argo Rollouts がない環境でも同じ Chart が動作するように設計。
- Rollouts Dashboard を `rollouts.localhost` で公開（Envoy Gateway HTTPRoute 経由）。
---
 
### Phase 11-4: KEDA
 
Prometheus メトリクスを元にイベント駆動オートスケールを実現する。
 
### 構成コンポーネント
| コンポーネント | namespace | Helmチャート | バージョン |
|---|---|---|---|
| KEDA | keda | kedacore/keda | 2.19.0 |
 
### ScaledObject（sample-backend）
```yaml
triggers:
  - type: prometheus
    metadata:
      serverAddress: http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090
      query: sum(rate(http_requests_total{job="sample-backend"}[1m]))
      threshold: "10"   # 10 req/sec を超えたらスケールアウト
minReplicaCount: 1
maxReplicaCount: 5
```
 
### 負荷テスト結果
`oha` で 50並列・120秒の負荷（約2,000 req/sec）をかけ、1 → 5 レプリカへのスケールアウトを確認。
負荷終了後、cooldownPeriod（30秒）+ HPA 安定化ウィンドウ（5分）を経て 1 レプリカに戻ることも確認。
 
### 動作確認
```bash
# ScaledObject の状態確認
kubectl get scaledobject -n sample-app
 
# KEDA が自動生成した HPA の確認
kubectl get hpa -n sample-app
```
 
### 設計上の決定事項
- KEDA は HPA を自動生成するため、既存の `hpa.enabled` は無効のまま併用しない。
- Rollout リソースを scaleTargetRef に指定することで、Argo Rollouts と KEDA を連携。
- `common-app` Service に `name: http` を付与（v0.3.0）することで、ServiceMonitor が Prometheus scrape ターゲットを正しく解決できるようになった。

---

### Phase 11-5: DB リトライロジック

sample-backend に asyncpg コネクションプールと tenacity によるリトライ処理を導入し、PostgreSQL フェイルオーバー時の接続断から自動回復できるようにした。

#### アプリ側の変更（sample-backend リポジトリ）

| 変更点 | 内容 |
|---|---|
| コネクションプール | `asyncpg.create_pool()` に移行（min: 2 / max: 10） |
| リトライ処理 | `tenacity` による最大5回・指数バックオフ（1〜16秒） |
| `/health` 改善 | `SELECT 1` による DB ping を追加。readinessProbe と連動 |

#### 検証結果

CNPG primary（db-2）を強制削除してフェイルオーバーを発生させ、db-3 への昇格後にアプリが自動回復することを確認した。

```bash
# フェイルオーバーの発生
kubectl delete pod sample-backend-db-2 -n sample-app

# 回復確認
curl http://sample-backend.localhost/health
curl http://sample-backend.localhost/items
```

#### 設計上の決定事項
- `/health` に DB ping を追加したことで、DB 障害時に readinessProbe が失敗しトラフィックが遮断される。フェイルオーバー完了後は自動で回復する。
- コネクションプールの `max_inactive_connection_lifetime: 30` により、サーバー側で切断されたコネクションを使用前に検知できる。
- リトライ対象は `PostgresConnectionError` / `TooManyConnectionsError` / `OSError` の一時的障害のみ。アプリバグ起因のエラーはリトライしない。

### Phase 11-6: Crossplane（provider-helm）
 
Crossplane v2.2.1 と provider-helm v1.0.4 を導入し、Helm Release を Kubernetes リソースとして GitOps 管理する構成を実証した。
 
#### 構成コンポーネント
 
| コンポーネント | namespace | 管理方法 | バージョン |
|---|---|---|---|
| Crossplane | crossplane-system | Helm Chart | 2.2.1 |
| provider-helm | crossplane-system | Crossplane Provider | v1.0.4 |
 
#### デプロイフロー
 
```
Git（Release CR）→ ArgoCD sync → Crossplane 調整 → Helm release デプロイ
```
 
ArgoCD がプラットフォーム・アプリの宣言を管理し、Crossplane が Helm release のライフサイクルを管理するという責務分離の構造をローカル環境で実証した。Phase 13（EKS）では `provider-aws` を使い、RDS や VPC などのクラウドリソースを同じ構造で管理する。
 
#### 動作確認
 
```bash
# Crossplane が管理する Helm release の状態確認
kubectl get release podinfo
 
# provider-helm の状態確認
kubectl get providers provider-helm
```
 
#### 設計上の決定事項
- `ProviderConfig` と `Release` CR は provider-helm の CRD が揃ってから適用する必要があるため、sync-wave を分割した（wave 1: Provider + RBAC、wave 2: ProviderConfig + Release）。
- `SkipDryRunOnMissingResource=true` を crossplane-config Application に設定し、CRD 未存在時の dry-run 失敗を回避。
- ローカル環境では `InjectedIdentity`（プロバイダ Pod の ServiceAccount をそのまま認証情報として使用）で接続。
---
 
### Phase 11-7: Cilium（CNI置き換え）
 
k3s デフォルトの flannel を Cilium v1.19.3（eBPF CNI）に置き換えた。
 
#### 構成コンポーネント
 
| コンポーネント | namespace | 管理方法 | バージョン |
|---|---|---|---|
| Cilium | kube-system | Helm Chart（bootstrap + ArgoCD） | 1.19.3 |
 
#### bootstrap 順序の変更
 
flannel 無効化後は CNI がない状態でクラスタが起動するため、CoreDNS など全 Pod が Pending になる。Cilium を先にインストールしてからその他の処理を行うよう順序を変更した。
 
```
cluster-create → install-cilium → fix-coredns → bootstrap-argocd → bootstrap-sync
```
 
#### k3d ローカル環境での制約
 
`kubeProxyReplacement=true`（kube-proxy 完全置き換え）は k3d では起動順序の問題で CoreDNS が API サーバーに接続できなくなるため無効化。ローカル環境では Cilium は CNI + NetworkPolicy の担当に留め、kube-proxy は k3s に任せる。
 
#### 動作確認
 
```bash
# Cilium の状態確認
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium
 
# ノードの接続性確認
kubectl get nodes -o wide
```
 
#### 設計上の決定事項
- Cilium は ArgoCD（wave 1）で GitOps 管理するが、bootstrap 時は ArgoCD より先に CNI が必要なため `make install-cilium` で別途インストールする。
- `k8sServiceHost/Port` は `kubeProxyReplacement=false` では不要。指定すると operator が `0.0.0.0:6443` に接続しようとして失敗するため除外する。
- `cluster.yaml` で `--flannel-backend=none` と `--disable-network-policy` を宣言し、Cilium に CNI と NetworkPolicy を委譲。
---
 
### 番外編：image tag 更新を PR 方式に変更
 
`update-gitops.yaml` ワークフローを直接 push から PR + squash merge 方式に変更し、image tag 更新の履歴を GitHub 上の PR として残せるようにした。
 
#### 変更前後の比較
 
| | 変更前 | 変更後 |
|---|---|---|
| 方式 | main に直接 push | PR 作成 → squash merge |
| 履歴 | コミットのみ | PR + コミット |
| ブランチ | なし | `gitops/update-{app}-{tag}`（マージ後自動削除） |
 
---
 
### sync-wave 構成の整理（Phase 11 番外編）
 
ArgoCD + KEDA + ESO の組み合わせで発生した問題を解消するため、sync-wave を整理した。
 
#### 解消した問題
 
**ESO webhook タイミングエラー**：ESO が wave 未設定のため ClusterSecretStore（wave 3）より後に起動し、webhook 検証に失敗していた。ESO を wave 1 に移動して解消。
 
**KEDA と ArgoCD selfHeal の競合**：KEDA がスケールした `replicas` を ArgoCD が Git の値で上書きしていた。`values.yaml` から `replicaCount` を削除して KEDA の `minReplicaCount` に委ね、`ignoreDifferences` + `RespectIgnoreDifferences=true` で sync 時の上書きを防止。
 
**Trivy DB キャッシュロック**：bootstrap 直後に複数のスキャン Job が同時起動し、Trivy DB のキャッシュロックが競合してエラーになっていた。`concurrentScanJobsLimit: 1` と `concurrentNodeScanners: 1` で逐次実行に制限して解消。

---
 
## sync-wave 構成
 
ArgoCD の sync-wave により、CRD → Operator → 設定リソース → アプリの順序でデプロイが制御される。
 
| Wave | 主なコンポーネント | 備考 |
|---|---|---|
| 1 | Cilium、Envoy Gateway CRDs、cert-manager、ESO | CRD・CNI・webhook を先行 |
| 2 | Crossplane、Envoy Gateway、CNPG | Operator 層 |
| 3 | ArgoCD、Monitoring、KEDA、Argo Rollouts、Trivy | プラットフォームサービス層 |
| 4 | cert-manager-config、external-secrets-config、gateway-config | 設定リソース（各 Operator の CRD 依存） |
| 5 | sample-backend、sample-frontend | アプリ層（CNPG Operator Ready 後） |
| 6 | crossplane-config | provider-helm CRD 依存リソース |
