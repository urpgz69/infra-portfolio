# プロジェクトインフラおよびデプロイアーキテクチャの構成概要 (bitemate / Team-CAL)

## プロジェクト概要
**bitemate (Team-CAL)** は、店舗運営を支援する Web・モバイル統合プラットフォームです。
スタッフ管理、シフト作成、勤怠管理、休暇・代替勤務の申請、給与計算、書類管理といった店舗運営業務に加え、**OpenCV (YOLO) による来店客数のカウント・混雑度の算出**機能を備えています。店舗カメラ映像から人物を検出して客数・混雑度を算出し、バックエンドに連携する OpenCV/FastAPI サービスを持つため、機械学習ライブラリを含む複数サービスのコンテナ統合が必要となりました。

本ドキュメントは、チーム開発において私が担当した**インフラおよびデプロイ領域**についてまとめた資料です。

- **リポジトリ:** [https://github.com/kkmin20200331-max/Team-CAL-](https://github.com/kkmin20200331-max/Team-CAL-)

## 担当範囲
複数人のチーム開発であり、インフラ領域における私の担当を、関与の度合いに応じて正確に区分すると以下の通りです。

- **主導 (自ら調査・設計・設定):**
  - Oracle Cloud ATP (Autonomous Transaction Processing) の接続構成およびトラブルシューティング (ORA-12529 / HikariCP)
  - `docker-compose.yml` による複数サービスのコンテナ統合構成と、環境変数 (`.env`) の分離設計
- **ペア作業 (メンバーと共同で構築):**
  - GitHub Actions を用いた各サービスの CI/CD パイプライン
  - 各サービスの Dockerfile 作成・イメージ軽量化
- **調整のみ (主担当ではない):**
  - 既存の Nginx 設定を、Docker コンテナ構成に合わせて調整 (プロキシの転送先をコンテナが公開するポートに向ける等)。Nginx 自体の設計や SSL 構成の主担当ではありません。

---

## 1. 導入背景と移行経緯
初期段階では、仮想マシン (Azure VM) 上で各プロセスを直接起動する手法をとっていました。

- **バックエンド:** `./gradlew bootRun` によるフォアグラウンド実行
- **フロントエンド:** `npm run build` 実行後、成果物を `/var/www` に手動コピー
- **OpenCV/FastAPI:** `uvicorn` のバックグラウンド実行

この構成は初期動作検証には適していましたが、運用管理上において以下の課題が発生しました。

- サーバー再起動時のプロセス自動復旧が困難
- 実際にデプロイされているコードのバージョンが追跡できない
- 各サービスで配備手順が異なり、オペレーションミスを誘発しやすい
- OpenCV や PyTorch などの機械学習系ライブラリの依存関係が重く、ホストVM環境の整合性を保ちにくい
- 各サービス間のネットワーク接続において、`127.0.0.1` や `host.docker.internal` の接続先混乱が発生する

これらの課題を解決するため、**Docker Compose を用いたコンテナ統合管理**へと移行し、再現性と安全性の高いインフラへと再設計しました。

---

## 2. インフラおよびネットワーク構成

ホストマシン上には外部ポート (HTTP 80 / HTTPS 443) のみを公開し、コンテナ間の内部通信およびデータベースへの接続経路を制御する構成としています。

```text
[クライアントブラウザ]
      │ (HTTPS 443 / HTTP 80)
      ▼
[ホスト VM (Azure VM)]
  └─ [Nginx (リバースプロキシ / SSL終端) ※既存構成]
        ├─ (127.0.0.1:3000) ──> [frontend コンテナ]
        └─ (127.0.0.1:8080) ──> [backend コンテナ (Spring Boot)]
                                    │
                                    ├─ (http://opencv:8000) ──> [opencv コンテナ (FastAPI / YOLO)]
                                    │
                                    └─ (TLS / Credential Wallet) ──> [Oracle Cloud ATP]
```

> ※ Nginx はチームの既存構成として先に用意されており、私はコンテナ化に合わせた設定調整のみを担当しました (詳細は 5 章)。上図はサービス間の接続関係を示すものです。

### 改善効果の比較 (VM直接実行 → Docker Compose)

| 項目 | 移行前 (VM直接実行) | 移行後 (Docker Compose) |
| --- | --- | --- |
| デプロイ方式 | サービス個別の手動実行 | Docker Composeによる一元管理 |
| バージョン管理 | 稼働中のコードバージョンが不明確 | `sha-xxxxxxx` タグによる明確な追跡 |
| フロントエンド配備 | `/var/www` への手動ビルド・コピー | コンテナイメージの入れ替え |
| OpenCV実行環境 | VMのローカルPython環境に依存 | Dockerイメージによる環境の固定化 |
| サービス間通信 | localhostの混同による接続失敗 | Composeのサービス名による名前解決 |
| 障害対応 | プロセスの手動確認・再起動 | コンテナ単位の確認・自動再起動設定 |

---

## 3. Oracle ATP 接続障害 (ORA-12529) の解決プロセス

本プロジェクトで私が最も主導的に取り組んだのが、Oracle Cloud の自律型データベース (ATP) への接続障害対応です。接続時に **ORA-12529 (TNS: Connect fail)** が発生しましたが、これは複数の独立した問題が重なって起きていたため、**まず手元の構成 (`.env`・Docker・VM 上の各 yml ファイル) から確認し、そこからネットワーク → Oracle 認証 → アクセス制御へと外側に向かって階層的に切り分け**て解決しました。

### 階層的アプローチによる切り分け手順（内側の構成 → 外側へ）
1. **内側の構成 (`.env` / `docker-compose.yml` / VM 上の各 yml):** 環境変数の重複、接続記述子 (URL) の記述、`sqlnet.ora` の Wallet ディレクトリパス指定とマウントの整合性、マウント先の読み取り権限を確認
2. **ネットワーク境界:** クラウドのネットワーク・セキュリティ・グループ (NSG) およびインバウンド規則の検証
3. **Oracle 認証 (Wallet):** クレデンシャル・ウォレットファイルの破損有無および最新性の検証
4. **アクセス制御 (ACL):** Oracle Cloud 側の ATP アクセス制御リスト (ACL) の適用状態確認

### 判明した3つの複合原因と対策

ORA-12529 (接続拒否) は、次の3つの独立した原因が重なって発生していました。

1. **アクセス制御 (ACL) の反映漏れ**
   - **原因:** Oracle Cloud コンソール上で ACL に VM のパブリック IP を追加したものの、変更を「保存」する操作が完了しておらず、実際には接続が拒否されていた。
   - **対策:** コンソール上で保存を実行し、IP 制限の適用を確定した。
2. **クレデンシャル・ウォレットの世代不一致**
   - **原因:** データベース側の再作成に伴い認証用の Wallet が更新されていたが、サーバー上には初期検証用の旧 Wallet が残っていた。
   - **対策:** 最新の Wallet ファイルを再取得し配置した。
3. **データソース記述子 (URL) の記述重複**
   - **原因:** Spring Boot の構成ファイル (`application.yml`) 内の記述と、環境変数による接続記述子の指定が混在し、接続文字列の解釈で不整合が起きていた。
   - **対策:** 設定を整理し、接続文字列を環境変数経由の指定に統一した。

> これら3点を解消したことで、ATP への**接続自体は成立する**ようになりました。

---

## 3-2. 接続成功後に発生した起動時エラー (HikariCP "pool sealed")

上記の接続問題とは別に、接続が通るようになった後の**アプリケーション起動時**に、HikariCP の "pool sealed" エラーが発生しました。接続拒否 (ORA-12529) とは切り分けが必要な、別レイヤーの問題です。

- **確実に理解している範囲:** HikariCP は、コネクションプールが一度初期化 (起動) されると、その後は設定を変更できない (sealed) 仕様です。この仕様を知らないまま、`docker-compose.yml` の環境変数 (`SPRING_DATASOURCE_HIKARI_*` : プール名・最大プールサイズ・各種タイムアウト等) で HikariCP のプール設定を後から上書きしようとしたため、競合が生じてエラーになったものと理解しています。
- **対策:** これらの HikariCP 個別設定を環境変数で上書きするのをやめ (該当行をコメントアウトして無効化)、Spring Boot の自動構成に設定を委ねる形へ整理したところ、正常に起動するようになりました。

```yaml
# docker-compose.yml (backend) 抜粋 — 上書きを止めてコメントアウトした HikariCP 設定
environment:
  SPRING_DATASOURCE_URL: ${SPRING_DATASOURCE_URL}
  SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME}
  SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD}
  SPRING_DATASOURCE_HIKARI_DATA_SOURCE_PROPERTIES_ORACLE_NET_TNS_ADMIN: /app/wallet
  # 以下は起動時競合を避けるため無効化し、自動構成に委ねた
  # SPRING_DATASOURCE_HIKARI_POOL_NAME: TeamCalOraclePool
  # SPRING_DATASOURCE_HIKARI_MAXIMUM_POOL_SIZE: 5
  # SPRING_DATASOURCE_HIKARI_CONNECTION_TIMEOUT: 30000
  # ...（他タイムアウト系も同様にコメントアウト）
```

---

## 4. Docker Compose によるサービス統合構成

サービス間のコンテナ定義、Wallet のマウント構造、およびシークレット情報の分離を行った `docker-compose.yml` の設計です。この構成の設計は私が主導しました。（実値・実アカウント名などの機密情報は環境変数参照・プレースホルダに置き換えています。）

```yaml
services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    image: ${DOCKERHUB_NAMESPACE}/bitemateback:${BACKEND_IMAGE_TAG:-latest}
    container_name: shiftops-backend
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_DRIVER_CLASS_NAME: oracle.jdbc.OracleDriver
      SPRING_DATASOURCE_URL: ${SPRING_DATASOURCE_URL}
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD}
      # Walletの参照先パス
      SPRING_DATASOURCE_HIKARI_DATA_SOURCE_PROPERTIES_ORACLE_NET_TNS_ADMIN: /app/wallet
      # この他、LINEログイン・Supabase・OCR等の連携キーも .env から注入（本抜粋では省略）
    volumes:
      # Oracle Walletを格納したホストディレクトリをコンテナ内に読み取り専用(ro)でマウント
      - ./backend/wallet:/app/wallet:ro
    networks:
      - shiftops-network
    restart: unless-stopped

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    image: ${DOCKERHUB_NAMESPACE}/bitematefront:${FRONTEND_IMAGE_TAG:-latest}
    container_name: shiftops-frontend
    ports:
      - "3000:80"     # コンテナ内のNginxポート(80)をホストの3000にバインド
    networks:
      - shiftops-network
    restart: unless-stopped

  opencv:
    image: ${DOCKERHUB_NAMESPACE}/bitemateopencv:${OPENCV_IMAGE_TAG:-latest}
    container_name: shiftops-opencv
    ports:
      - "8000:8000"
    environment:
      YOLO_MODEL: ${YOLO_MODEL:-yolo11s.pt}                 # 客数カウントに使用するYOLOモデル
      SPRING_CONGESTION_URL: ${SPRING_CONGESTION_URL:-http://backend:8080/api/ai/congestion}
      SPRING_API_KEY: ${SPRING_API_KEY}
    networks:
      - shiftops-network
    restart: unless-stopped

networks:
  shiftops-network:
    driver: bridge
```

### 設計の要点
- **環境変数の分離:** パスワードや各種 API キーなどの秘匿情報は compose ファイル内に直接記述せず、`.env` ファイル (Git 管理対象外) から読み込む構造とした。
- **コンテナ間通信の制御:** 共通のカスタムブリッジネットワーク (`shiftops-network`) を構成。これにより、同一ネットワーク内のコンテナからは `http://backend:8080` や `http://opencv:8000` のようにサービス名を用いた内部名前解決が可能となった。OpenCV サービスは算出した客数・混雑度を `http://backend:8080/api/ai/congestion` へ内部連携する。
- **Wallet の保全性:** Oracle ATP 接続に必要な Credential Wallet は、不正な書き換えを防止するため `:ro` (Read-Only) オプションを適用してマウントした。

---

## 5. Nginx 設定のコンテナ対応（調整担当・主担当ではない）

Nginx はチームの既存構成として先に用意されていました。私の担当は、**サービスをコンテナ化するにあたり、既存の Nginx 設定の転送先をコンテナ側に合わせて調整する**ことに限られます。Nginx 自体の設計や SSL/TLS 構成の主担当ではありません。

具体的に行ったのは、`proxy_pass` の転送先を、各コンテナがホストに公開しているポート（フロントエンド `127.0.0.1:3000` / バックエンド `127.0.0.1:8080`）へ向くよう合わせる調整です。ルーティングやヘッダー、SSL 終端（証明書配置・443 設定）は既存構成を踏襲しています。

```nginx
# 調整した箇所（コンテナのポートへ転送先を合わせる）
server {
    listen 443 ssl;
    server_name <YOUR_DOMAIN>;

    ssl_certificate     /etc/nginx/ssl/<project>/fullchain.pem;   # 既存構成を踏襲
    ssl_certificate_key /etc/nginx/ssl/<project>/privkey.pem;

    location /api/ {
        proxy_pass http://127.0.0.1:8080/api/;   # backend コンテナ
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        # 他ヘッダー・設定は既存を踏襲
    }

    location / {
        proxy_pass http://127.0.0.1:3000;        # frontend コンテナ
        proxy_set_header Host $host;
    }
}
```

> ※ 実 IP・実ドメイン・実プロジェクト名は `<YOUR_DOMAIN>` 等のプレースホルダに置換しています。

---

## 6. 環境変数の管理手法

セキュアな運用体制を維持するため、以下の Git 管理ルールを徹底しました。

- **`.env` (リポジトリ登録対象外):**
  DB パスワードや API トークンなどの実データを記述。`.gitignore` によりリポジトリへの混入を防止している。
- **`.env.example` (リポジトリ登録対象):**
  設定キーのみを示したテンプレート。`.gitignore` の除外ルール (`!.env.example`) を適用し、チーム内で必要な環境変数の種類を安全に共有できるようにした。

```gitignore
# .gitignore 抜粋
.env
.env.*
!.env.example
```

---

## 7. CI/CD パイプライン構築 (ペア作業)

モノレポ (Mono-repo) 構成のため、特定のディレクトリが修正された際にのみ該当ワークフローが走るよう GitHub Actions を構築しました。この領域はメンバーとのペア作業です。イメージレジストリには Docker Hub を使用しています。

### ワークフロー構成

| ワークフロー | 監視パス (paths) | 処理内容 | トリガー |
| --- | --- | --- | --- |
| `backend-docker.yml` | `backend/**` | Docker build/push（内部で Java 17 ビルド） | 全ブランチで実行 / push は dev・main のみ |
| `frontend-docker.yml` | `frontend/**` | Docker build/push（内部で Node 20 ビルド、`VITE_*` を build-args 注入） | 全ブランチで実行 / push は dev・main のみ |
| `opencv-docker.yml` | `opencv/**` | Docker build/push（OpenCV/YOLO 環境） | dev・main のみ |
| `native-ci.yml` | `native/**` | 型チェック（`npx tsc --noEmit`） | push・PR（全ブランチ） |

### パイプライン運用の特徴

- **無駄なビルドの抑制:** `paths` フィルタリングを設定し、変更に関係しないサービスの CI 実行を抑制した。
- **トリガーと push の分離:** ワークフローは全ブランチの push で起動するが、レジストリへの push は `push: ${{ github.ref == 'refs/heads/dev' || github.ref == 'refs/heads/main' }}` により **dev / main ブランチのときのみ**行う。feature ブランチではビルド検証のみが走る。
- **コミットハッシュによるタグ管理:** イメージタグに `sha-${GITHUB_SHA::7}` を自動適用（`docker/metadata-action` により `latest` はデフォルトブランチのみ付与）し、デプロイ済みイメージを Git 履歴と一致させて追跡可能にした。
- **自動化と手動適用の分離:** dev / main への反映時、GitHub Actions は Docker Hub へのイメージ push と、デプロイ先 VM の `.env` 内イメージタグの自動書き換え（`appleboy/ssh-action` による `sed` 置換）までを行う。ただし実際のサーバー適用（`docker compose pull && docker compose up -d`）は**オペレーターが任意のタイミングで手動実行**とし、稼働中サービスへの不意な再起動を防いでいる。
- **フロントエンドの環境変数埋め込み:** Vite の環境変数 (`VITE_*`) はビルド時に成果物へ埋め込まれる仕様のため、GitHub Secrets から `build-args` 経由で注入した。加えて、注入漏れを検知する検証ステップ（必須 Secret の存在チェック）をビルド前に設けた。

---

## 8. 運用管理および診断用コマンド

実運用時に使用した、コンテナ状況およびインフラ健全性の点検用コマンド一覧です。

- **コンテナ状態の一覧点検:**
  ```bash
  docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
  ```
- **バックエンドから OpenCV コンテナへの接続テスト:**
  ```bash
  docker exec shiftops-backend wget -qO- http://opencv:8000/health
  ```
- **Docker ストレージ容量の点検:**
  ```bash
  docker system df
  ```
- **VM 全体のディスク容量およびディレクトリ別サイズ点検:**
  ```bash
  df -h
  sudo du -sh /var/lib/docker /var/lib/containerd /home/<user> /var/log
  ```

---

## 9. 学んだこと

- **エラーを層別に切り分けるアプローチの有効性**
  ORA-12529 の対応を通じ、「接続失敗＝ネットワークだけ／コードだけ」と決めつけず、境界領域から段階的にパラメータを固定して検証することの重要性を学んだ。さらに、接続成立後に別レイヤー（HikariCP の起動時設定）で問題が起きたことから、症状の似た障害でも**レイヤーを分けて切り分ける**必要があると実感した。
- **複合的な要因へ粘り強く向き合う**
  ORA-12529 は「コンソール上の保存漏れ」「旧世代の Wallet」「接続 URL の重複」という3つの独立した要因が重なって起きていた。一つ解決しても、全体が動くまで他の原因が残りうる前提でテストを繰り返す大切さを学んだ。
- **協業と自主担当の役割分担**
  全員のリリース効率に直結する CI/CD や Dockerfile はメンバーとペアで整合を取りつつ、Oracle ATP 接続のように複雑な切り分けと個別パラメータの決定が必要な領域は自ら主導して調査・設定を担当した。担当の濃淡（主導／ペア／調整のみ）を自分の中で明確に区別することが、チーム開発では重要だと学んだ。
