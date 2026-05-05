# ユニット・オブ・ワーク計画 — Sloth-Lab

## 計画チェックリスト

- [x] ステップ 1: コンテキスト分析（application-design.md / requirements.md / stories.md 読み込み済み）
- [x] ステップ 2: ユニット分解の初期提案
- [x] ステップ 3: ユーザー回答の収集（Q1=A モノレポ）
- [x] ステップ 4: 成果物の生成
  - [x] unit-of-work.md
  - [x] unit-of-work-dependency.md
  - [x] unit-of-work-story-map.md

---

## 初期ユニット分解提案

アプリケーション設計の分析から、以下の3ユニットへの分解を提案します。

| # | ユニット名 | 技術スタック | 主な責務 |
|---|-----------|------------|---------|
| 1 | `backend-api` | Node.js / TypeScript / Lambda | POST /darui, GET /darui/count, DynamoDBアクセス |
| 2 | `ios-app` | Swift / WidgetKit / AppIntent | iOS メインアプリ, Widget Extension, AppIntent |
| 3 | `infrastructure` | AWS CDK (TypeScript) | DynamoDB, Lambda, API Gateway のプロビジョニング |

**分解の根拠**:
- 3つは異なる技術スタックを持ち、それぞれ独立して開発・テスト可能
- `infrastructure` は `backend-api` のデプロイ先を定義するため先行して構築
- `ios-app` はバックエンドの API エンドポイントが確定すれば独立して開発可能

---

## 確認が必要な設計上の意思決定

### 質問 1: コードの管理構造（リポジトリ構成）

3ユニット（ios-app / backend-api / infrastructure）をどのように管理しますか？

A) **モノレポ**（1つのリポジトリ内に全ユニットを配置）
```
Sloth-Lab/                  ← リポジトリルート
  ios/                      ← iOS アプリ（Xcode プロジェクト）
  backend/                  ← Node.js API
  infrastructure/           ← AWS CDK
  aidlc-docs/               ← AI-DLC ドキュメント
```

B) **別リポジトリ**（ユニットごとに独立したリポジトリ）
```
sloth-lab-ios/              ← iOS リポジトリ
sloth-lab-backend/          ← Backend リポジトリ
sloth-lab-infrastructure/   ← Infrastructure リポジトリ
```

C) **その他** (その他の場合は [Answer]: タグの後に記述してください)

[Answer]: A

---

## 次のステップ

上記の質問に回答後、「完了しました」と送信してください。  
回答を分析し、ユニット成果物（unit-of-work.md / unit-of-work-dependency.md / unit-of-work-story-map.md）を生成します。
