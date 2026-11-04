# KubernetesからOpenShiftへのアプリケーション移行・認定タスク一覧

本ドキュメントは、Kubernetes（K8s）環境で稼働している既存アプリケーションをRed Hat OpenShiftへ移植し、Red Hat Universal Base Image（UBI）の採用、**Red Hat Container Certification（コンテナ認定）、およびOpenShift Operator / Helm Certification（OpenShift認定）**を取得・維持するための一連のタスク一覧です。

---

## 1. 全体ロードマップ

| フェーズ | 主要目標 | 主要成果物 |
| :--- | :--- | :--- |
| **Phase 1: ベースイメージ設計・移行** | UBIへの刷新、ライセンス同梱、マルチアーキテクチャ対応 | UBI準拠 Containerfile / Dockerfile |
| **Phase 2: セキュリティ・権限適合** | 非Root実行（任意UID対応）、SCC適合、特権排除 | SCC準拠・非特権対応コンテナイメージ |
| **Phase 3: パッケージング** | Operator (OLM) または Helm Chart 形式でのパッケージ化 | ClusterServiceVersion (CSV) / Helm Chart |
| **Phase 4: 事前テスト・検証** | Partner Connect登録、Preflight CLIによる適合性チェック | Preflight検証レポート・テストログ |
| **Phase 5: 認定申請・運用公開** | Pyxis提出、脆弱性評価、Ecosystem Catalog公開 | 認定完了バッジ、OperatorHubカタログ公開 |

---

## 2. フェーズ別タスク詳細

### Phase 1: ベースイメージのUBI移行とビルド設計

コンテナ認定の必須前提条件として、コンテナイメージがRed Hat Enterprise Linux（RHEL）またはUniversal Base Image（UBI）をベースに構築されている必要があります。

- [ ] **1.1 UBIフレーバーの選定**
  - アプリケーション要件に応じたベースイメージを選択。
    - `ubi9-micro`: パッケージマネージャー非同梱。Go/Rustなどの静的バイナリ向け。
    - `ubi9-minimal`: 軽量パッケージマネージャー（`microdnf`）同梱。一般的なマイクロサービス向け。
    - `ubi9`: 完全な`dnf`スタック同梱。標準的なユーティリティが必要な場合。
    - 公式言語ランタイムイメージ: Python、Node.js、OpenJDK、.NET等。
- [ ] **1.2 パッケージ管理・ビルド手順の刷新**
  - Debian系（`apt`）やAlpine系（`apk`）のインストールコマンドを`dnf`または`microdnf`に置換。
  - パッケージ取得先を公式UBIリポジトリ（`ubi-9-baseos`, `ubi-9-appstream`）に統一。
  - イメージサイズ最小化のためキャッシュクリーンアップ（`microdnf clean all` / `rm -rf /var/cache/yum`）を徹底。
- [ ] **1.3 ライセンスファイルの配置（認定必須要件）**
  - コンテナ内の `/licenses` ディレクトリに、自社ソフトウェア利用許諾契約書および使用しているOSSライセンス群を配置。
- [ ] **1.4 マルチアーキテクチャビルド対応（推奨・要件に応じる）**
  - `podman build --platform=linux/amd64,linux/arm64` によるマニフェストリスト作成とQuay等のレジストリへのプッシュ。

---

### Phase 2: OpenShiftセキュリティ・実行要件への適合

OpenShift標準のSecurity Context Constraints（SCC: `restricted-v2`）配下で完全動作するコンテナ設計を実施します。

- [ ] **2.1 非Rootおよび任意UID（Arbitrary UID）実行対応**
  - Dockerfile内で `USER` にRoot（`0`）を指定せず、`USER 1001` 等の非特権ユーザーを定義。
  - OpenShift実行時にランダムUIDが割り当てられても動作するよう、必要ディレクトリの所有グループをRootグループ（`gid=0`）とし、グループ書き込み権限を付与：
    ```dockerfile
    RUN chown -R 1001:0 /app && \
        chmod -R g=u /app
    USER 1001
    ```
- [ ] **2.2 特権要求およびLinux Capabilitiesの排除**
  - `securityContext.privileged: true` の利用禁止。
  - 不必要なCapabilities（`NET_ADMIN`, `SYS_ADMIN`等）の要求を排除し、デフォルトの制限付き権限で動作することを確認。
- [ ] **2.3 SUID/SGIDバイナリの除去**
  - セキュリティリスクとなる不要なSUID/SGIDビット付きバイナリをビルド時に削除。
- [ ] **2.4 ルーティングとストレージの適合**
  - IngressリソースをOpenShift `Route`（Edge / Passthrough / Re-encrypt）または標準Ingressとして整合性確認。
  - クラウドプロバイダー固有のアノテーションを汎用的な設定に置換。
  - PVC（PersistentVolumeClaim）のアクセスモード（RWO/RWX）とUID/GIDパーミッションの適合確認。

---

### Phase 3: OpenShiftパッケージング（Operator または Helm）

OpenShift認定を受けるための配布形態として、Operator（推奨）またはHelm Chartを作成します。

#### A. Operator方式（Operator Lifecycle Manager / OLM）
- [ ] **3.1 Operatorの作成**
  - Operator SDK（Go, Ansible, Helmベース）を用いてコントローラーを実装。
- [ ] **3.2 ClusterServiceVersion（CSV）の構成**
  - Operatorメタデータ、対応OpenShiftバージョン、アノテーションを記述。
  - `relatedImages` セクションに、Operatorおよび配下のワークロードが利用する全コンテナイメージをダイジェスト（`sha256:`）形式で指定。
  - アイコン、説明文、プロバイダー情報の設定。
- [ ] **3.3 RBAC権限の最小化**
  - ServiceAccountに付与する権限をNamespaceスコープ（Role）に限定し、ClusterRoleBindingの多用を避ける。

#### B. Helm Chart方式
- [ ] **3.4 Helm Chartの調整**
  - Helm v3形式に準拠し、OpenShift固有のリソース（Route等）と互換性のあるテンプレートを作成。

---

### Phase 4: Red Hat Partner Connectでの事前検証とテスト実行

Red Hat公式認定ツール `preflight` CLI を使用して、要件適合性を検証します。

- [ ] **4.1 Partner Connectでのプロジェクト作成**
  - [Red Hat Partner Connect](https://connect.redhat.com/) にログインし、認定プロジェクト（Container Project / Operator Project）を作成してAPIトークンを発行。
- [ ] **4.2 Preflight CLI によるコンテナ認定テスト**
  - コンテナイメージに対してテストを実行：
    ```bash
    preflight check container <イメージURI> \
      --pyxis-api-token=<TOKEN> \
      --certification-project-id=<PROJECT_ID> \
      --docker-config=~/.docker/config.json
    ```
  - UBIベース、レイヤー構造、ライセンスファイル、CVEスキャン結果がすべてPassであることを確認。
- [ ] **4.3 Preflight CLI によるOperator認定テスト**
  - Operator Bundleに対してテストを実行：
    ```bash
    preflight check operator <バンドルイメージURI> \
      --pyxis-api-token=<TOKEN> \
      --certification-project-id=<PROJECT_ID> \
      --docker-config=~/.docker/config.json
    ```
  - OLM仕様適合性、CRDスキーマ妥当性、非特権動作がすべてPassであることを確認。

---

### Phase 5: 認定申請・脆弱性対応・Ecosystem Catalog公開

- [ ] **5.1 認定テスト結果の提出**
  - `--submit` オプションを付けてPreflightを実行し、結果をRed Hat認定ポータル（Pyxis）へ送信。
- [ ] **5.2 脆弱性評価（Health Index）の維持**
  - Red HatスキャナーによるCVE検出結果を確認し、未解決のCritical/Important脆弱性がない状態（Health Index A〜C）を確保。
- [ ] **5.3 Red Hatによる審査と公開**
  - パートナーポータル上で製品プロファイルやサポート情報を入力し、最終承認を申請。
  - 承認後、**Red Hat Ecosystem Catalog** および OpenShiftクラスタ内の **OperatorHub** に正式公開。
- [ ] **5.4 継続的再認定（CI/CDパイプライン統合）**
  - アプリケーション更新時やOpenShift新バージョンリリース時に自動的にPreflightテスト・再認定を行えるようCI/CDパイプラインに組み込み。
