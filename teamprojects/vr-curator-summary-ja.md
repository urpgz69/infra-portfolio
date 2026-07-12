# VR-Curator (AI Exhibition) — インフラ・デプロイ担当 概要

## プロジェクト概要
Web ブラウザ上で 3D 仮想展示館を探索し、作品ごとの **AI ドーセント解説**とリアルタイムの来場者インタラクションを提供する展示プラットフォーム。React + Three.js で 3D 空間をレンダリングし、Spring Boot が API と WebSocket を、FastAPI + Gemini が解説生成を担当する（外部 AI が使えない場合はブラウザ内 WebLLM にフォールバック）。

- **リポジトリ:** https://github.com/koozic/VR
- **技術スタック:** React + Three.js / Spring Boot / FastAPI (Gemini) / Oracle Cloud ATP / Docker Compose / Apache / GHCR / EC2

## 担当範囲
- **主導（自ら調査・設計・構築）:** 3 サービスの Dockerfile・マルチステージ構成、モノレポのビルドコンテキスト設計、`docker-compose.yml` 統合と環境変数分離、**Apache リバースプロキシ（WebSocket プロキシ / TLS 終端）**、GHCR ビルド → push → EC2 手動デプロイの構成
- **担当外の支援:** 担当者不在で滞っていた DB 移行（ローカル Oracle → ATP）を引き受けて完了。続く**コンテナ接続構成（wallet マウント / SSO / 接続文字列）はインフラ領域として主導**
- **チーム協議のうえ反映:** リバースプロキシを Nginx → Apache へ変更、DB 運用方針（ATP 採用）

## インフラ構成
```
ブラウザ (HTTPS)
  └ Apache（TLS 終端 / WebSocket プロキシ）
       ├ /            → frontend コンテナ
       ├ /api /ws     → backend (Spring Boot)
       │                   ├ ai-server (FastAPI) → Gemini
       │                   └ Oracle Cloud ATP (Wallet)
       └ ai-server は外部非公開（内部呼び出しのみ）= 攻撃対象領域を最小化
```

## 主な成果① ― ATP をコンテナから接続させる（本デプロイ最大の関門）
データ移行自体は約 20 分で完了したが、**移行した ATP をコンテナ環境から接続させる構成**に最も時間を要した。3 つの問題を順に解決：

1. **JDBC ドライバが ATP 非対応** ― `pom.xml` の変更は共有コードに影響するため、独断で上げずチーム合意のうえ `ojdbc11` を対応版へ更新
2. **wallet がコンテナ内に未マウント (ORA-12263)** ― ホストにあってもコンテナには見えないため、`docker-compose.yml` で読み取り専用 (`:ro`) マウント
3. **SSO keystore を読む companion ライブラリ不足 (ORA-17957)** ― `oraclepki`・`osdt_core`・`osdt_cert` を追加し、fat jar の内部まで確認して依存の同梱を検証

この過程で、**「ホストにある ≠ コンテナから見える」**「**ビルド成功 ≠ 依存が成果物に含まれる**」という 2 つの境界感覚を得た。

## 主な成果② ― Apache リバースプロキシ（WebSocket・TLS）
`mod_proxy_wstunnel` で WebSocket をプロキシ（欠けると「3D 画面は表示されるがリアルタイム機能だけ動かない」症状になる）、Let's Encrypt で TLS 終端を担当。証明書は `live/` が `archive/` を指す **symlink** のため、`/etc/letsencrypt` 全体をマウントして解決した（これも「ホストにある ≠ コンテナから見える」の延長）。

## その他の要点
- **モノレポのビルドコンテキスト:** 共用リソース (`shared/`) をイメージに取り込むため、コンテキストを直下に引き上げ `.dockerignore` で範囲を限定
- **要件に沿った範囲設計:** 「デプロイのみ」の要件に合わせ、CI を設けずローカルビルド → GHCR push → EC2 手動デプロイで構成（過剰に自動化しない判断も設計の一部）

## 学んだこと
- **ホストとコンテナの境界を常に意識する**（wallet・証明書で繰り返し確認、symlink は参照先までマウント範囲に含める）
- **「ビルド成功」を信頼の終点にしない**（最終成果物の内部まで確認して初めて確実になる）
- **変更の波及範囲を判断し、チームと調整する**（共有コードに触れる変更は技術的可否とは別に合意が要る）
- **要件の範囲を正確に実装する**（自動化が常に正解とは限らない）
