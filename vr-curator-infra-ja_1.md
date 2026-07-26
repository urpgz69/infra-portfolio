# プロジェクトインフラおよびデプロイアーキテクチャの構成概要 (VR-Curator / AI Exhibition)

## プロジェクト概要
**VR-Curator (AI Exhibition)** は、Web ブラウザ上で 3D 仮想展示館を探索し、作品ごとの AI ドーセント解説とリアルタイムの来場者インタラクションを提供する展示プラットフォームです。

React + Three.js で 3D 展示空間をレンダリングし、Spring Boot が展示データ・管理 API および WebSocket によるリアルタイム機能を担当します。FastAPI ベースの AI サーバーが Gemini と連携し、作品解説・質問応答を生成します。外部 AI が利用できない場合は、ブラウザ内の WebLLM にフォールバックする設計です。

本ドキュメントは、チーム開発において私が担当した**インフラおよびデプロイ領域**についてまとめた資料です。

- **リポジトリ:** [https://github.com/koozic/VR](https://github.com/koozic/VR)

### リポジトリ構成 (モノレポ)

```text
frontend/   React + Three.js クライアント
backend/    Spring Boot API サーバー
ai-server/  FastAPI AI サービス
docs/       プロジェクトドキュメント
shared/     フロントエンド・バックエンドが共用する gallery seed データ
```

フロントエンド・バックエンド・AI サーバーが一つのリポジトリに含まれるモノレポ構成であり、`shared/` の seed データをフロントエンドとバックエンドが共用します。この共用構造は、後の Docker ビルドコンテキスト設計における重要な考慮事項となります (4章)。

## 担当範囲
本プロジェクトにおいて、インフラ・デプロイ領域は私が主導的に担当しました。設計・構築・トラブルシューティングの大半を主導しており、チームと協議して決定した箇所は別途明記します。

- **主導 (自ら調査・設計・構築):**
  - フロントエンド・バックエンド・AI サーバー 3 サービスの Dockerfile 作成およびマルチステージイメージ構成
  - モノレポのビルドコンテキスト設計 (`shared/` 共用リソースの取り込み)
  - `docker-compose.yml` によるサービス統合構成と、環境変数 (`.env`) の分離
  - Apache リバースプロキシの構築 (WebSocket プロキシ / TLS 終端)
  - GHCR イメージのビルド・push および EC2 へのデプロイ手順の構成
  - 要件に沿ったデプロイ範囲の設計 (自動テスト・自動ビルド・自動デプロイを設けず、「デプロイのみ」という要件に合わせ、ローカルビルド → GHCR push → EC2 手動デプロイの流れで構成)
- **担当外の支援 (人員不在への対応):**
  - DB 移行 (ローカル Oracle → ATP) は本来別パートの作業でしたが、担当者が不在で進行が滞っていたため、データ規模が小さいことを踏まえて自ら引き受けました。これに続く**コンテナ接続の構成 (wallet マウント / SSO ライブラリ / 接続文字列) は、インフラ領域として私が主導**しました。
- **チーム協議のうえ反映 (意思決定に参加):**
  - リバースプロキシを Nginx から Apache へ変更 (クラウド経験のあるメンバーの提案を検討・採用)
  - DB 運用方針の決定 (デプロイ上の制約と運用要件をチームと整理したうえで、ATP 移行に確定)

---

## 1. 導入背景および移行経緯
初期はローカル開発環境で各サービスを直接起動する方式でした。

- **フロントエンド:** `npm run dev` (Vite 開発サーバー)
- **バックエンド:** `mvn spring-boot:run` (`local` プロファイル、H2 インメモリ DB)
- **AI サーバー:** `python -m app.main` (uvicorn の直接起動)

この構成はローカル検証には適していましたが、EC2 へのデプロイに移るにあたり、2 つの軸での移行が必要になりました。

### 移行 ① 実行環境: ローカル直接起動 → Docker Compose
異なる 3 つのランタイム (Node / Java 17・Maven / Python) を EC2 ホストに直接インストール・管理すると、再構築やインスタンス入れ替えのたびに環境を合わせ直す必要があり、デプロイ済みコードのバージョン追跡も困難です。これを、各サービスをイメージとして分離した **Docker Compose による統合管理**へ再設計し、ホストには Docker のみを置く形として、再現性とロールバックの安全性を確保しました。

### 移行 ② データベース: ローカル Oracle → Oracle Cloud ATP
開発中はローカルの Oracle を使用していましたが、この DB が**プライベート IP 帯域 (10.x.x.x)** にあるため、EC2 から直接接続できないという制約がありました。デプロイのためには、EC2 からアクセス可能な DB が必要でした。

- **H2 への切り替えを検討 → 見送り:** デプロイは最も容易になりますが、既に Oracle ベースで開発されているため Oracle 固有の SQL を H2 向けに修正する必要があり、運用時に再び Oracle へ戻すとコードを二重管理することになるため見送りました。
- **Oracle Cloud ATP への移行 → 採用:** 同じ Oracle 系のためアプリケーションコードの修正がほとんど不要で、EC2 から接続でき、マネージド DB のため運用負荷も低いことから採用しました。

> **補足: ATP とは** — Oracle Cloud が提供するマネージド型の自律運用データベース (Autonomous Database) です。パッチ適用やチューニングを自動化しており、接続には後述する Wallet (接続用の認証情報一式) を用います。

実際のデータ移行は本来別パートの作業でしたが、担当者不在で滞っていたこと、規模が大きくないことから、時間短縮のため自ら引き受けて進めました。ATP のアカウントをチーム基準で作成したうえで、**約 20 分で完了**しました。**むしろ移行後、「コンテナから ATP へ接続させる」構成 (wallet マウント・SSO ライブラリ・接続文字列) が、インフラ上の課題として今回のデプロイで最も時間を要した関門**であり、これは 3 章で詳述します。

### 改善効果の比較 (ローカル直接起動 → Docker Compose + ATP)

| 項目 | 移行前 | 移行後 |
| --- | --- | --- |
| デプロイ方式 | サービス個別の手動起動 (npm/mvn/python) | Docker Compose による一元管理 |
| バージョン管理 | 稼働中コードのバージョンが不明確 | イメージタグでバージョンを区別 (手動固定) |
| 実行環境 | ホストに 3 種のランタイムを直接インストール | イメージによりランタイムを固定 |
| DB アクセス | ローカル Oracle がプライベート IP → EC2 から接続不可 | ATP → EC2 から接続可能 |
| サービス間通信 | localhost の混同リスク | Compose のサービス名による名前解決 |
| 障害対応 | プロセスの手動確認・再起動 | コンテナ単位の確認・自動再起動 |

---

## 2. インフラおよびネットワーク構成

ホスト (EC2) には外部ポート (HTTP 80 / HTTPS 443) のみを公開し、その他のサービスは Docker の内部ネットワークでのみ通信する構成としました。特に **AI サーバーは外部に公開せず、バックエンドからのみ内部で呼び出す**構成とし、**Apache が TLS 終端 (HTTPS の復号) と WebSocket プロキシを兼ねて**担当します。

```text
[クライアントブラウザ]
      │ (HTTPS 443 / HTTP 80)
      ▼
[ホスト EC2]
  └─ [Apache (リバースプロキシ / TLS 終端 / WebSocket プロキシ)]
        ├─ (/)            ──> [frontend コンテナ (Tomcat による静的配信)]
        ├─ (/api)         ──> [backend コンテナ (Spring Boot)]
        ├─ (/uploads)     ──> [backend コンテナ (アップロードファイル配信)]
        ├─ (/ws/gallery)  ──> [backend コンテナ (WebSocket)]
        │                        │
        │                        ├─ (http://ai-server:8010) ──> [ai-server コンテナ (FastAPI)]
        │                        │                                    │
        │                        │                                    └─ (外部 API) ──> [Gemini]
        │                        │
        │                        └─ (TLS / Wallet) ──> [Oracle Cloud ATP]
        │
        └─ ※ ai-server は外部に公開しない (backend からのみ内部呼び出し)
```

### 構成の要点

- **AI サーバーの非公開 (内部専用):** リクエストの流れが `フロント → バックエンド → AI サーバー → Gemini` と続くため、AI サーバーを外部へ公開する必要がありません。したがって Apache の公開経路は `/`・`/api`・`/uploads`・`/ws/gallery` に限定し、AI サーバーはホストポートを開けず、内部ネットワークから `ai-server:8010` へのみアクセスする構成としました。攻撃対象領域 (attack surface) を減らす狙いです。
- **Apache の役割集約:** 単なる静的プロキシではなく、① TLS 終端 (Let's Encrypt 証明書)、② `/api` の HTTP プロキシ、③ `/ws/gallery` の WebSocket プロキシを一箇所で担います。WebSocket プロキシには専用モジュールと設定が必要であり、5 章で詳述します。
- **WebLLM フォールバック (サーバー負荷なし):** 外部 AI を利用できない場合のフォールバックはブラウザ内の WebLLM で動作するため、サーバー側に推論経路や GPU リソースは不要です。インフラの観点では、AI サーバーは Gemini を呼び出す軽量なプロキシとしてのみ維持されます。
- **サービス名ベースの内部通信:** すべてのコンテナを共通のブリッジネットワークに配置し、`http://backend:8080`・`http://ai-server:8010` のようにサービス名で内部の名前解決ができるようにしました。
- **フロントエンドの配信方式 (Apache + Tomcat 構成の学習):** フロントエンドの静的ビルド成果物 (`dist/`) を、一般的な Nginx ではなく **Tomcat の `ROOT` ウェブアプリとして配信**する方式を採りました。Apache リバースプロキシと Tomcat (WAS) を組み合わせる Java 系の標準構成を実際に扱ってみるための選択で、`ProxyPass` と `ProxyPassReverse` を併せて設定し、正常動作を確認しました。`ProxyPassReverse` を併用したのは、バックエンドが返すリダイレクト応答の `Location` ヘッダーに内部アドレス (`http://frontend:8080/...`) がそのまま露出せず、外部ドメイン基準で書き換えられるようにするためです。

---

## 3. Oracle Cloud ATP へのコンテナ接続構成 (本デプロイ最大の関門)

1 章で述べたとおり、データ移行そのものは短時間で終わりましたが、**移行した ATP を「コンテナ環境から実際に接続させる」構成**が、今回のデプロイで最も時間を要した部分です。接続が成立するまで、3 つの問題を順に解決しました。

### 3-1. ドライバのバージョンが ATP に対応していない

**症状:**
クラウド ATP の導入後、既存の Docker イメージが載っている EC2 で環境変数 (`URL` / `ID` / `PASSWORD`) を注入して接続を試みたところ、アプリケーションログに **現在の JDBC ドライバのバージョンでは接続できない旨のメッセージ**が出力されました。

**原因:**
既存イメージに含まれる Oracle JDBC ドライバのバージョンが、Oracle Cloud ATP (Autonomous Database) への接続をサポートする水準ではありませんでした。原因がログに直接示されたケースであり、原因究明よりも**修正範囲の判断**が要点でした。

**解決 (チーム合意のうえ実施):**
ドライバのバージョンアップは `pom.xml` を変更する作業であり、チームの共有コードに影響します。そのため独断で上げず、**メンバーの了承を得たうえで**、Oracle JDBC ドライバを ATP に対応する最新系 (`ojdbc11 23.5.0.24.07`) へ更新しました。

> bitemate プロジェクトの ORA-12529 が「原因が見えず階層ごとに絞り込んだ」ケースだったのに対し、今回は「**ログが原因を示していたが、その修正が共有コードのためチーム合意を要した**」ケースでした。原因究明よりも、変更の波及範囲を判断しチームと調整することが要点でした。

### 3-2. Wallet をコンテナ内から参照できない (ORA-12263)

**症状:**
```text
ORA-12263: Failed to access tnsnames.ora in the directory configured as TNS admin
```

**原因:**
EC2 ホストには wallet フォルダがありましたが、**コンテナ内部にはマウントされていませんでした。** コンテナはホストのファイルシステムを自動的には参照できないため、`TNS_ADMIN` が指すパスがコンテナ内には存在しませんでした。

**解決:**
`docker-compose.yml` の backend サービスに、wallet ディレクトリを読み取り専用 (`:ro`) でマウントしました。

```yaml
volumes:
  - /home/ubuntu/<wallet>:/app/wallet:ro
```

### 3-3. Wallet の SSO keystore を読むライブラリの不足 (ORA-17957 / SSO not found)

**症状:**
```text
ORA-17957: Unable to initialize the key store
java.security.KeyStoreException: SSO not found
ClassNotFoundException: oracle.security.crypto.core.AuthenticationException
```

**原因:**
JDBC ドライバ (`ojdbc11`) だけでは不足で、wallet の `cwallet.sso` (自動ログイン用の keystore) を読むための **companion ライブラリが不足**していました。当初は `oraclepki` を追加すれば足りると見えましたが、実際には `osdt_core`・`osdt_cert` まで必要でした。

**解決:**
`pom.xml` に wallet 関連のランタイム依存を明記しました。

```xml
<dependency>
    <groupId>com.oracle.database.jdbc</groupId>
    <artifactId>ojdbc11</artifactId>
    <version>23.5.0.24.07</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>com.oracle.database.security</groupId>
    <artifactId>oraclepki</artifactId>
    <version>23.5.0.24.07</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>com.oracle.database.security</groupId>
    <artifactId>osdt_core</artifactId>
    <version>21.21.0.0</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>com.oracle.database.security</groupId>
    <artifactId>osdt_cert</artifactId>
    <version>21.21.0.0</version>
    <scope>runtime</scope>
</dependency>
```

**検証:**
「ビルド成功」だけでは依存が実際に成果物へ含まれたか分からないため、ビルドされた Spring Boot の fat jar の内部まで確認しました。

```text
BOOT-INF/lib/ojdbc11-23.5.0.24.07.jar
BOOT-INF/lib/oraclepki-23.5.0.24.07.jar
BOOT-INF/lib/osdt_core-21.21.0.0.jar
BOOT-INF/lib/osdt_cert-21.21.0.0.jar
```

### この関門で得た 2 つの「境界」の感覚

1. **「ホストにファイルがある」≠「コンテナから参照できる」** (3-2)
   ランタイムに必要な wallet・証明書などのファイルは、ホスト上の存在と、コンテナ内での可視性を**別々に**確認する必要があります。
   ```bash
   docker exec -it <backend-container> ls -al /app/wallet
   ```
2. **「ビルド成功」≠「依存が成果物に含まれている」** (3-3)
   依存の問題は、ビルドの成否ではなく、最終的な fat jar (`BOOT-INF/lib`) の内部まで確認して初めて確実になります。

> なお最終構成では、`ojdbc11`・`oraclepki` は 23.5 系、`osdt_core`・`osdt_cert` は 21.21 系とバージョンラインが分かれています。現状は正常に動作しますが、今後ドライバを引き上げる際にこのバージョン組み合わせが SSO 初期化で再び問題となりうるため、併せて管理する必要があります。

---

## 4. モノレポのビルドコンテキスト問題 (共用リソースの欠落)

**症状:**
```text
FileNotFoundException: class path resource [docent-context.json] cannot be opened because it does not exist
```

**原因:**
バックエンドは、リポジトリ直下の `shared/` (gallery seed・docent context) を Maven リソースとして取り込む構成でした。ところが Docker ビルドを `backend` フォルダをコンテキストとして実行していたため、**コンテキスト外にある `../shared/` がイメージに含まれませんでした。** (概要で触れた `shared/` の共用構造が、ここで問題として現れます。)

Docker のビルドコンテキストは、指定したディレクトリの外にあるファイルを `COPY` できません。サービスフォルダをコンテキストにする限り、上位の `shared/` は決して取り込めません。

**解決:**
ビルドコンテキストを**リポジトリ直下に引き上げ**、Dockerfile の位置だけを `-f` で指定する方式に変更しました。

```dockerfile
# backend/Dockerfile (直下コンテキストを基準にパス指定)
COPY backend/pom.xml ./
COPY backend/src ./src
COPY shared ../shared
```

```bash
# ビルドはリポジトリ直下で実行
docker build --no-cache -f backend/Dockerfile -t ghcr.io/<owner>/ai-exhibition-backend:<tag> .
```

コンテキストを直下へ広げると不要なファイルまでビルドに取り込まれうるため、直下に `.dockerignore` を置き、`node_modules/`・ビルド成果物・`.git/` などを除外して、コンテキストを必要な範囲に限定しました。

**検証:**
最終的な jar に共用リソースが実際に含まれたかを確認しました。

```text
BOOT-INF/classes/gallery-seed.json
BOOT-INF/classes/docent-context.json
```

> **原則:** モノレポで複数サービスが共用するリソースがある場合、ビルドコンテキストは**直下に取り、`.dockerignore` で範囲を絞る**組み合わせが最も安定します。

---

## 5. Apache リバースプロキシの構成 (WebSocket・TLS)

Nginx に代えて Apache でリバースプロキシを構築する過程で、モジュール・プロキシ先・証明書マウントの 3 点を順に整理しました。

### 5-1. 必要なモジュールの読み込みとプロキシ先の指定

**症状:**
```text
Invalid command 'Redirect'
DNS lookup failure for: backend returned by /api
```

**原因:**
- `Redirect` は `mod_alias` モジュールがないと動作しません。
- WebSocket プロキシ (`ws://`) には `mod_proxy_wstunnel` が必要です。このモジュールがないと通常の HTTP プロキシは動くものの **WebSocket だけが失敗**し、「3D 画面は表示されるが、リアルタイム機能 (位置共有・チャット) だけ動かない」症状になります。
- `ProxyPass` の宛先は Docker Compose の**サービス名**を基準にしないと、内部の名前解決ができません。

**解決:**
必要なモジュールを明示的に読み込み、

```apache
LoadModule alias_module modules/mod_alias.so
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
LoadModule proxy_wstunnel_module modules/mod_proxy_wstunnel.so
```

プロキシ先をサービス名基準で指定しました。(WebSocket 経路は `ws://`、その他は `http://` で区別)

```apache
ProxyPass        /api          http://backend:8080/api
ProxyPass        /uploads      http://backend:8080/uploads
ProxyPass        /ws/gallery   ws://backend:8080/ws/gallery
ProxyPass        /             http://frontend:8080/
ProxyPassReverse /             http://frontend:8080/
```

**再発防止 (プロキシ障害時はまず内部 DNS を確認):**
```bash
docker exec -it <apache-container> getent hosts backend
docker exec -it <apache-container> getent hosts frontend
```

### 5-2. Let's Encrypt 証明書のマウント (symlink の落とし穴)

**症状:**
```text
SSLCertificateFile: file '/etc/letsencrypt/live/.../fullchain.pem' does not exist or is empty
```

**原因:**
ホストには証明書があるのに、コンテナからは正しく見えませんでした。Let's Encrypt の `live/` 配下のファイルは実体ではなく、`archive/` を指す **symlink** です。そのため `live/` だけをマウントすると、リンク先である `archive/` の実体ファイルがコンテナ内に無く、アクセスに失敗します。

**解決:**
`/etc/letsencrypt` **全体**を読み取り専用でマウントし、symlink とその参照先 (`archive/`) が併せて見えるようにしました。

```yaml
volumes:
  - /etc/letsencrypt:/etc/letsencrypt:ro
```

> これも 3 章と同じ「ホストにある ≠ コンテナから見える」の延長であり、特に **symlink は参照先までマウント範囲に含める**必要がある点が追加の要点です。

---

## 6. HTTPS 環境での WebSocket Mixed Content

**症状:**
```text
Mixed Content: The page was loaded over HTTPS, but attempted to connect to
the insecure WebSocket endpoint 'ws://...:8080/ws/gallery'
```

**原因:**
フロントエンドの Dockerfile のビルド引数の既定値に、HTTP/WS の絶対アドレスが固定されていました。

```dockerfile
ARG VITE_API_BASE_URL=http://<domain>:8080
ARG VITE_WS_BASE_URL=ws://<domain>:8080/ws/gallery
```

HTTPS で読み込まれたページでは、ブラウザが平文の `ws://` 接続をセキュリティ上ブロックします。しかし Apache が既に `/api`・`/ws/gallery` を HTTPS ドメインでプロキシしているため、フロントのバンドルに内部ポート (`:8080`) や `ws://` アドレスを固定する理由はありませんでした。

**解決:**
ビルド引数の既定値を空にし、フロントが**現在の接続 origin を基準に API・WebSocket のアドレスを自動生成**するようにしました。

```dockerfile
ARG VITE_API_BASE_URL=
ARG VITE_WS_BASE_URL=
```

```text
https://<domain>  ->  wss://<domain>/ws/gallery  (自動導出)
```

**再発防止:**
運用・発表向けのデプロイ前に、フロントのバンドルへ `localhost`・`:8080`・`http://`・`ws://` のような環境依存アドレスが埋め込まれていないか検査します。

```bash
grep -R "ws://\|http://.*8080\|localhost" frontend/dist
```

> **原則:** HTTPS の背後にリバースプロキシを置く構成では、フロントのバンドルに絶対アドレスを固定せず、**origin 基準で相対的に導出**するほうが安全です。同じイメージをどのドメインにデプロイしても動作し、mixed content も原理的に防げます。

---

## 7. Docker Compose によるサービス統合構成

4 サービス (apache / frontend / backend / ai-server) のコンテナ定義、wallet・アップロード用ボリューム、シークレットの分離をまとめた `docker-compose.yml` の構成です。(実際のアカウント名・パスなど機微な情報はプレースホルダに置き換えています。)

```yaml
services:
  apache:
    image: httpd:2.4
    container_name: ai-exhibition-apache
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./apache/httpd.conf:/usr/local/apache2/conf/httpd.conf:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro          # 証明書全体をマウント (5-2 参照)
    depends_on:
      - frontend
      - backend
    restart: unless-stopped

  frontend:
    image: ghcr.io/<owner>/ai-exhibition-frontend:003   # 手動固定のタグ
    container_name: ai-exhibition-frontend
    restart: unless-stopped

  backend:
    image: ghcr.io/<owner>/ai-exhibition-backend:006
    container_name: ai-exhibition-backend
    env_file:
      - .env                                          # シークレットは .env から注入
    environment:
      AI_SERVER_BASE_URL: http://ai-server:8010        # 内部通信はサービス名基準
      UPLOAD_STORAGE_DIR: /app/uploads
    volumes:
      - backend-uploads:/app/uploads                   # アップロードファイルの永続化
      - /home/ubuntu/<wallet>:/app/wallet:ro           # ATP wallet を読み取り専用でマウント (3-2 参照)
    depends_on:
      - ai-server
    restart: unless-stopped

  ai-server:
    image: ghcr.io/<owner>/ai-exhibition-aiserver:002
    container_name: ai-exhibition-ai-server
    env_file:
      - .env                                          # Gemini キーなどを注入
    restart: unless-stopped                            # ホストポート非公開 (内部専用)

volumes:
  backend-uploads:
```

### 構成の要点

- **AI サーバーの内部専用化:** `ai-server` には `ports` を設けず、ホストにポートを公開しません。バックエンドが内部ネットワークから `http://ai-server:8010` でのみ呼び出すため (2 章の図)、外部公開面を最小化しています。
- **アップロードファイルの永続化:** 作品画像などのアップロードは `backend-uploads` の named volume に保存し、コンテナを再生成してもデータが残るようにしました。コンテナのレイヤに置くとデプロイのたびに失われるためです。
- **wallet の読み取り専用マウント:** ATP 接続用の wallet は改変防止のため `:ro` でマウントしました。
- **起動順序の制御:** `depends_on` により、apache は frontend・backend の後、backend は ai-server の後に起動する順序としました。
- **イメージタグの手動固定:** 自動イメージデプロイ (CI) を設けない要件のため、イメージタグ (`:003`・`:006`・`:002`) を手動で固定しています。bitemate プロジェクトで CI がタグを自動更新 (`${TAG:-既定値}`) していた方式とは異なり、本プロジェクトはデプロイ時点でタグを明示的に管理する構成です。

---

## 8. 環境変数およびシークレットの管理

シークレットが Git に混入しないよう、次のルールを適用しました。

- **`.env` (Git 管理対象外):** DB 接続情報 (URL/ID/PW)、Gemini API キーなどの実値を記述。`.gitignore` によりリポジトリへの混入を防止。
- **`.env.example` (Git 管理対象):** 設定キー名のみを示すテンプレート。チームが必要な環境変数の種類を安全に共有できるようにする。
- **wallet・証明書:** wallet ディレクトリと Let's Encrypt 証明書もイメージに焼き込まず、ランタイムにマウントし、Git 管理から除外。

```gitignore
# .gitignore 抜粋
.env
.env.*
!.env.example
```

シークレットをイメージや compose ファイルに直接書かず、`.env` 参照とランタイムマウントに分離した結果、イメージ自体には機微な情報が含まれず、GHCR に載せても安全です。

---

## 9. デプロイの流れ (手動デプロイ)

要件に沿って、自動テスト・自動ビルド・自動デプロイは設けず、ローカルビルド → GHCR push → EC2 手動デプロイの流れで構成しました。ビルドはローカル (または開発マシン) で行い、EC2 は pull と実行のみを担うため、EC2 のリソースは実行だけに使われます。

```bash
# 1) ローカル: イメージのビルド (backend は直下コンテキスト、4 章参照)
docker build -f backend/Dockerfile -t ghcr.io/<owner>/ai-exhibition-backend:<tag> .

# 2) ローカル: GHCR への push
docker push ghcr.io/<owner>/ai-exhibition-backend:<tag>

# 3) EC2: compose のイメージタグを更新後、pull と再起動
docker compose pull
docker compose up -d
docker compose ps
```

- **ビルドと実行の分離:** EC2 で直接ビルドしないため、異なるランタイム (Node/Java/Python) をホストに入れる必要がなく、実行に必要なメモリだけ確保すればよい構成です。
- **デプロイ前の検証:** CI がない代わりに、デプロイ前に各サービスがローカルで `docker build` によって正常にビルドできることを確認することを、安全策としました。

---

## 10. 運用・診断用コマンド

実際のデプロイ・点検で使用したコマンドです。デプロイの問題は一度にまとめて見ず、**DB / AI サーバー / Apache / ブラウザをそれぞれ独立に**検証して原因を絞りました。

**コンテナ状態とサービス名の確認:**
```bash
docker compose ps
docker compose config --services
```

**DB 接続の確認 (HikariCP の起動ログ):**
```bash
docker compose logs backend | grep -i "HikariPool"
# 正常: HikariPool-1 - Start completed.
```

**AI サーバー連携の確認 (バックエンド→AI サーバーの内部経路):**
```bash
# 内部ネットワークから ai-server の health を呼び出し
docker run --rm --network "<net>" curlimages/curl http://ai-server:8010/health
# 応答の gemini.configured が true か確認
```

AI 機能の不具合は、次の順に階層を分けて確認しました: ① ai-server コンテナの起動有無 → ② backend から `ai-server:8010/health` への到達可否 → ③ health 応答の `gemini.configured` → ④ AI サーバーへの直接呼び出しの成功 → ⑤ backend 経由の呼び出しの成功。

**Apache の内部 DNS・証明書の確認:**
```bash
docker exec -it <apache-container> getent hosts backend
docker exec -it <apache-container> getent hosts frontend
docker exec -it <apache-container> ls -al /etc/letsencrypt/live/<domain>/
```

**ディスク使用量の点検 (イメージ蓄積への備え):**
```bash
docker system df
df -h
```

---

## 11. 学んだこと

- **ホストとコンテナの境界を常に意識する。**
  「ホストにファイルがある」と「コンテナから参照できる」は別問題である。wallet (3-2)・証明書 (5-2) で繰り返し確認し、特に symlink は参照先までマウント範囲に含める必要があることを学んだ。
- **「ビルド成功」を信頼の終点にしない。**
  依存が成果物に実際に含まれたか (3-3)、共用リソースがイメージに入ったか (4) は、ビルドの成否ではなく最終成果物の内部まで確認して初めて確実になる。
- **変更の波及範囲を判断し、チームと調整する。**
  ドライバのバージョンアップ (3-1) のように共有コードに影響する変更は、技術的に可能かとは別に、チーム合意を経る必要がある。原因究明と同じくらい「どこまで触れてよいか」の判断が重要だった。
- **デプロイの問題は階層を分けて独立に検証する。**
  DB・AI サーバー・Apache・ブラウザのコンソールをまとめて見ると原因を見失う。それぞれを health/DNS/ログ単位で分離して確認すると、素早く絞り込める。
- **要件の範囲を正確に実装する。**
  自動化 (CI/CD) が常に正解とは限らない。「デプロイのみ」が求められた規模では手動デプロイが適切であり、必要以上に作り込まない判断も設計の一部だった。
- **問題の根を理解すれば、制約下の代替案が見えてくる。**
  ATP 接続問題の根が「wallet (mTLS) の SSO をドライバが読むこと」だと理解すれば、ドライバのバージョンアップが不可能な状況で、TLS 接続方式へ切り替えて wallet 自体を無くす回避策も設計できる。
