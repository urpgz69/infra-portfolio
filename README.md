# Infrastructure & DevOps Portfolio

> クラウドインフラ・コンテナ・デプロイの実践記録
> AWS EC2 での個人プロジェクト運用と、チーム開発でのインフラ・デプロイ担当を通じて学んだことをまとめています。

---

## 概要 (Summary)

開発からインフラまで、実際に手を動かしながら学んできました。インフラは独学で始めましたが、**「動かすこと」だけでなく「なぜそう動くか」を説明できるレベル** を重視し、トラブルを自分で切り分けて解決する経験を重ねています。

- **クラウド**: AWS (EC2, EBS, Security Group, DLM), Azure, Oracle Cloud (ATP)
- **コンテナ / デプロイ**: Docker, Docker Compose, マルチステージビルド, GitHub Actions (自動デプロイ), GHCR / Docker Hub, 手動デプロイ
- **サーバ運用**: Nginx / Apache (リバースプロキシ, TLS 終端, WebSocket プロキシ), Let's Encrypt TLS, Linux (Ubuntu) 運用, 監視 (Netdata), バックアップ自動化 (cron, DLM)
- **保有資格**: AWS Certified Solutions Architect – Associate (SAA)

---

## プロジェクト一覧

### 1. DevDesk — 個人インフラ学習プロジェクト (AWS EC2)

**クラウドインフラを本格的に触るのは初めてでしたが、実運用を想定して個人で設計・構築しました。** 実際のユーザーはいませんが、チュートリアル通りには進まないトラブルを一つずつ切り分けて解決する中で、「動かす」だけでなく「なぜそう動くか」を理解することを重視しました。

**構築したもの**
- Docker Compose によるマルチコンテナ運用 (アプリ + DB + Nginx)
- Nginx リバースプロキシ + Let's Encrypt TLS (webroot 方式で **無停止の証明書自動更新**)
- CI/CD: コミットハッシュによるイメージタグ付け、ロールバック構成
- バックアップの二重化: pg_dump (論理) + EBS スナップショット (DLM 自動)、バックアップ用に別ボリュームを分離
- Netdata による監視導入

**チュートリアルにはない、実際のトラブル対応**
- **ディスク枯渇 (使用率 99%)**: 監視導入中に早期発見 → EBS ルートボリュームを **オンライン拡張** (growpart + resize2fs、無停止) で根本解決。「監視を入れた直後に監視が最初の危機を捉えた」経験。
- **TLS 証明書の自動更新の不具合**: 設定ファイルの記述ミスやボリュームマウントを一つずつ検証して解決。
- **バックアップの cron 設定ミス**: 作業ディレクトリ指定漏れで空ファイルが生成される問題を発見・修正。

> 「手順通りに動く」段階で終わらせず、なぜ失敗し、どう直したかを理解することを大切にしました。

🔗 リポジトリ: [cicdstudy](https://github.com/urpgz69/cicdstudy) *(公開)*

---

### 2. VR-Curator (AI Exhibition) — 3サービス構成のインフラ・デプロイ主導 ✅ *(完了)*

React + Three.js の 3D 仮想展示館に、Spring Boot (API / WebSocket) と FastAPI + Gemini (AI ドーセント解説) を組み合わせた展示プラットフォーム。**インフラ・デプロイ領域を主導** しました。

> **担当範囲の明記**: 3 サービスの Dockerfile・`docker-compose.yml` 統合・Apache リバースプロキシ・GHCR / EC2 デプロイ手順は **私が主導**。リバースプロキシの Nginx → Apache 変更と DB 運用方針は**チーム協議のうえ決定**。DB 移行 (ローカル Oracle → ATP) は本来別パートでしたが、担当者不在で滞っていたため引き受け、続く**コンテナ接続構成はインフラ領域として主導**しました。

**構築したもの**
- 3 サービス (frontend / backend / ai-server) のコンテナ化と Docker Compose 統合
- **Apache リバースプロキシ**: TLS 終端 (Let's Encrypt) + **WebSocket プロキシ** (`mod_proxy_wstunnel`) を一箇所に集約
- **AI サーバーを外部非公開** (backend からの内部呼び出しのみ) とし、攻撃対象領域を最小化
- Oracle Cloud ATP への移行と、コンテナからの接続構成 (wallet マウント / SSO ライブラリ / 接続文字列)
- 要件に合わせたデプロイ設計: ローカル build → **GHCR** push → EC2 手動デプロイ

**最大の関門 — 「ATP をコンテナから接続させる」**
DB 移行自体は約 20 分で完了しましたが、**移行した ATP をコンテナ環境から接続させる構成**に最も時間を要しました。3 つの問題を順に解決:
1. **JDBC ドライバが ATP 非対応** → `pom.xml` の変更は共有コードに影響するため、独断で上げず**チーム合意のうえ**更新
2. **wallet がコンテナ内に未マウント (ORA-12263)** → 読み取り専用 (`:ro`) でマウント
3. **SSO keystore 用の companion ライブラリ不足 (ORA-17957)** → `oraclepki` / `osdt_core` / `osdt_cert` を追加し、**fat jar の内部まで確認**して同梱を検証

**学んだこと**
- **「ホストにファイルがある」≠「コンテナから参照できる」** — wallet・証明書で繰り返し直面。特に Let's Encrypt の `live/` は symlink のため、参照先 (`archive/`) までマウント範囲に含める必要がある
- **「ビルド成功」≠「依存が成果物に含まれている」** — 最終成果物の内部まで確認して初めて確実になる
- **変更の波及範囲を判断し、チームと調整する** — 共有コードに触れる変更は、技術的な可否とは別に合意が要る
- **要件の範囲を正確に実装する** — 自動化 (CI/CD) が常に正解とは限らず、必要以上に作り込まない判断も設計の一部

🔗 リポジトリ: [VR](https://github.com/koozic/VR)
📁 詳細: [team-projects/vr-curator.md](./team-projects/vr-curator.md)

---

### 3. bitemate (Team-CAL) — マルチサービス構成のインフラ・CI 担当 ✅ *(完了)*

Spring Boot + React + FastAPI (OpenCV / YOLO による来店客数カウント) + React Native のモノレポ構成のチーム開発。インフラ・CI・DB 接続の領域を担当しました。

> **担当範囲の明記**: **主導** = Oracle Cloud ATP の接続構成とトラブルシューティング、`docker-compose.yml` によるコンテナ統合と環境変数の分離設計。**ペア作業** = GitHub Actions の CI/CD、各サービスの Dockerfile。**調整のみ** = 既存 Nginx 設定をコンテナのポートに合わせる調整 (Nginx 自体の設計・SSL 構成の主担当ではありません)。

**主な取り組み**
- バックエンド ↔ Oracle Cloud ATP の接続障害 (**ORA-12529**) を **階層ごとに切り分けて解決**
  - 手元の構成 (`.env` / compose / yml) → ネットワーク (NSG) → ウォレット → ATP の ACL へと、**内側から外側へ**順に検証
  - 原因が **3 つ重なっていた** (ACL の保存漏れ + 旧世代ウォレット + 接続 URL の記述重複) ことを一つずつ解消
  - 接続成立後、**別レイヤー**で HikariCP の "pool sealed" が発生 → プールは初期化後に設定変更できない仕様のため、環境変数での上書きをやめ自動構成に委ねて解決
- サービスごとの GitHub Actions CI 構成 (paths フィルタ、sha タグ、キャッシュ)
- 各サービスの Dockerfile (マルチステージビルド)、サービス名によるコンテナ間通信

**学んだこと**
- エラーコードが示す表面的な原因にとらわれず、接続経路を**層別に**検証することの重要性
- 原因は一つとは限らない (複数の要因が重なる)
- **担当の濃淡 (主導 / ペア / 調整のみ) を明確に区別する** — チーム開発では自分の関与度を正確に把握することが重要

🔗 リポジトリ: [Team-CAL-](https://github.com/kkmin20200331-max/Team-CAL-)
📁 詳細: [team-projects/bitemate.md](./team-projects/bitemate.md)

---

## デプロイ方式の使い分け

3 プロジェクトを通じて、**要件・規模に応じてデプロイ方式を選択**する経験を得ました。

| プロジェクト | デプロイ方式 | 選択理由 |
| --- | --- | --- |
| DevDesk | 自動 (GitHub Actions) | 個人運用のため、CI/CD の全工程を自ら構築・検証 |
| bitemate | CI 自動 + 適用は手動 | イメージ push までは自動化し、稼働中サービスへの不意な再起動を防ぐため適用はオペレーター判断 |
| VR-Curator | 手動 (build → GHCR push → EC2 pull) | 「デプロイのみ」という要件に合わせ、過剰に自動化しない判断 |

---

## 学習ノート

実践で使った技術を、面接で説明できるレベルまで整理したノート。

- [CI/CD & Dockerfile 整理](./notes/cicd-dockerfile.md) — build/push/deploy の違い、モノレポ CI、Dockerfile のマルチステージ構造、よく踏む落とし穴

---

## 使用技術 (Tech Stack)

| 領域 | 技術 |
| --- | --- |
| Cloud | AWS (EC2, EBS, DLM, Security Group), Azure, Oracle Cloud (ATP) |
| Container | Docker, Docker Compose, マルチステージビルド |
| Registry | GHCR (GitHub Container Registry), Docker Hub |
| デプロイ | GitHub Actions (自動), 手動デプロイ (build → push → pull & up) |
| Web/Proxy | Nginx / Apache (リバースプロキシ, TLS 終端, WebSocket プロキシ), Let's Encrypt TLS |
| OS/Runtime | Linux (Ubuntu), Java 17, Node.js, Python |
| 監視/運用 | Netdata, バックアップ自動化 (cron, DLM) |

---

## 補足

このポートフォリオは学習と実務経験の記録です。掲載内容には機密情報 (認証情報、実 IP アドレス等) は一切含めていません。
