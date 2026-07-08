# Infrastructure & DevOps Portfolio

> クラウドインフラ・コンテナ・デプロイの実践記録
> AWS EC2 での個人プロジェクト運用と、チーム開発でのインフラ・デプロイ担当を通じて学んだことをまとめています。

---

## 概要 (Summary)

開発からインフラまで、実際に手を動かしながら学んできました。インフラは独学で始めましたが、**「動かすこと」だけでなく「なぜそう動くか」を説明できるレベル** を重視し、トラブルを自分で切り分けて解決する経験を重ねています。

- **クラウド**: AWS (EC2, EBS, Security Group, DLM), Azure, Oracle Cloud (ATP)
- **コンテナ / デプロイ**: Docker, Docker Compose, マルチステージビルド, GitHub Actions (自動デプロイ), GHCR / Docker Hub, 手動デプロイ
- **サーバ運用**: Nginx / Apache, Let's Encrypt TLS, Linux (Ubuntu) 運用, 監視 (Netdata), バックアップ自動化
- **保有資格**: AWS Certified Solutions Architect – Associate (SAA)

---

## プロジェクト一覧

### 1. DevDesk — 個人インフラ学習プロジェクト (AWS EC2)

**クラウドインフラを本格的に触るのは初めてでしたが、実運用を想定して個人で設計・構築しました。** 実際のユーザーはいませんが、チュートリアル通りには進まないトラブルを一つずつ切り分けて解決する中で、「動かす」だけでなく「なぜそう動くか」を理解することを重視しました。

**構築したもの**
- Docker Compose によるマルチコンテナ運用 (アプリ + DB + Nginx)
- Nginx リバースプロキシ + Let's Encrypt TLS (webroot方式で **無停止の証明書自動更新**)
- CI/CD: コミットハッシュによるイメージタグ付け、ロールバック構成
- バックアップの二重化: pg_dump (論理) + EBSスナップショット (DLM自動)、バックアップ用に別ボリュームを分離
- Netdata による監視導入

**チュートリアルにはない、実際のトラブル対応**
- **ディスク枯渇 (使用率99%)**: 監視導入中に早期発見 → EBSルートボリュームを **オンライン拡張** (growpart + resize2fs、無停止) で根本解決。「監視を入れた直後に監視が最初の危機を捉えた」経験。
- **TLS証明書の自動更新の不具合**: 設定ファイルの記述ミスやボリュームマウントを一つずつ検証して解決。
- **バックアップの cron 設定ミス**: 作業ディレクトリ指定漏れで空ファイルが生成される問題を発見・修正。

> 「手順通りに動く」段階で終わらせず、なぜ失敗し、どう直したかを理解することを大切にしました。

🔗 リポジトリ: [cicdstudy](https://github.com/urpgz69/cicdstudy) *(公開)*

---

### 2. bitemate (Team-CAL) — マルチサービス構成のインフラ・CI担当

Spring Boot + React + FastAPI + React Native のモノレポ構成のチーム開発。
インフラ・CI・DB接続の領域を担当しました。

> **担当範囲の明記**: CI / Dockerfile / コンテナ構成はチーム (ペア作業を含む) で構築し、その中で **DB接続 (Oracle Cloud ATP) と docker-compose / 環境変数の設定を主に担当** しました。

**主な取り組み**
- バックエンド ↔ Oracle Cloud ATP の接続障害 (ORA-12529) を **階層ごとに切り分けて解決**
  - ネットワーク → ウォレット → 接続URL → 設定ファイルのパス → フォルダ権限 → ATP ACL の順に検証
  - 原因が **複数重なっていた**こと (URL重複 + ウォレット旧バージョン + ACL未保存 + コネクションプール設定の競合) を一つずつ解決し、正常起動を達成
- サービスごとの GitHub Actions CI 構成 (paths フィルタ、shaタグ、キャッシュ)
- 各サービスの Dockerfile (マルチステージビルド)、サービス名によるコンテナ間通信

**学んだこと**
- エラーコードが示す表面的な原因にとらわれず、接続経路を体系的に検証することの重要性
- 原因は一つとは限らない (複数の要因が重なる)
- 他者のインフラ領域と自分の領域を切り分け、権限のある側に引き継ぐ判断

🔗 リポジトリ: [Team-CAL-](https://github.com/kkmin20200331-max/Team-CAL-)
📁 詳細: [team-projects/bitemate.md](./team-projects/) *(準備中)*

---

### 3. vr-curator — マルチサービス構成のデプロイ担当 *(進行中)*

React (Three.js) + Spring Boot (WebSocket) + FastAPI (AI) の3サービス構成のチーム開発。
チームの要件が「デプロイ」だったため、**コンテナ化とデプロイ手順の構築を担当** しています。

> **担当範囲の明記**: アプリ開発はチームで進行中で、私は **3サービスのコンテナ化と、GHCR / AWS EC2 へのデプロイ手順の構築・検証** を担当しています。

**構築したデプロイ手順** (今回は自動化(CI/CD)は要件外のため、手動デプロイ)
- フロント / バック / AI の3サービスそれぞれの Dockerfile を作成し、イメージをビルド
- **GHCR (GitHub Container Registry)** にイメージを push
- AWS EC2 (Ubuntu) 上で Docker / Compose を構成し、GHCR から pull して起動
- **ローカル build → GHCR push → EC2 pull & up** の流れを確立・検証

**取り組み中の課題**
- ローカルOracleが**プライベートIP**のためEC2から接続不可 → **Oracle Cloud (ATP)** への移行で解決予定 (Oracle同士のためコード修正は最小限)

> DevDesk では自動デプロイ (GitHub Actions) を、本プロジェクトでは要件に合わせた手動デプロイを構築。**要件・規模に応じてデプロイ方式を選択** した経験。

---

## 学習ノート

実践で使った技術を、面接で説明できるレベルまで整理したノート。

- [CI/CD & Dockerfile 整理](./notes/cicd-dockerfile.md) — build/push/deploy の違い、モノレポCI、Dockerfileのマルチステージ構造、よく踏む落とし穴

---

## 使用技術 (Tech Stack)

| 領域 | 技術 |
| --- | --- |
| Cloud | AWS (EC2, EBS, DLM, Security Group), Azure, Oracle Cloud (ATP) |
| Container | Docker, Docker Compose, マルチステージビルド |
| Registry | GHCR (GitHub Container Registry), Docker Hub |
| デプロイ | GitHub Actions (自動), 手動デプロイ (build → push → pull & up) |
| Web/Proxy | Nginx / Apache, Let's Encrypt TLS (リバースプロキシ, TLS termination) |
| OS/Runtime | Linux (Ubuntu), Java 17, Node.js, Python |
| 監視/運用 | Netdata, バックアップ自動化 (cron, DLM) |

---

## 補足

このポートフォリオは学習と実務経験の記録です。掲載内容には機密情報 (認証情報、実IPアドレス等) は一切含めていません。
