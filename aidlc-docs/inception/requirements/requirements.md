# Requirements — Refactor the World (RTW)

## 意図分析サマリー

| 項目 | 内容 |
|------|------|
| **リクエストタイプ** | 新規プロジェクト（Greenfield） |
| **スコープ** | 複数コンポーネント — AI生成パイプライン・SNSフィード・ユーザー認証・バックエンドAPI |
| **複雑性** | 中程度（外部AI API統合・画像処理・リアルタイムフィードが主な難所） |
| **深度** | 標準（Standard） |

---

## ビルドスコープ（MVP定義）

**MVP = カメラ撮影 → AI変換 → RTWフィードへの投稿・閲覧・いいね**

以下はMVPに含めない（後続フェーズ）：
- ポイント・報酬システム（設計レベルで考慮するが実装しない）
- 企業ダッシュボード
- スポンサー課題
- PoC支援マッチング
- 採用報酬

---

## 機能要件

### FR-01: ユーザー認証
- メールアドレス＋パスワードによる新規登録・ログイン・ログアウト
- セッション管理（JWTトークン）

### FR-02: アプリ内カメラ撮影
- アプリ内でカメラを起動し写真を撮影する
- カメラロールからの画像選択も可能
- 撮影した写真（before画像）をAI変換パイプラインへ渡す

### FR-03: AI変換（Capture & Refactor）
- GPT-4Vで撮影画像を解析し「現状の課題・改善方向」を把握する
- DALL-E 3で「理想の姿」のafter画像を生成する
- 変換はアプリ内で完結（外部ブラウザ等に遷移しない）
- 生成時間の目標：10秒以内
- 変換に失敗した場合はエラーメッセージを表示し再試行を促す

### FR-04: 投稿（Social Commit）
- before/after画像のペアをRTWフィードに投稿する
- 投稿時にカテゴリタグを付与できる（例：食事・街・インテリア・その他）
- 投稿はRTWフィード上で公開される

### FR-05: フィード閲覧
- 他ユーザーの投稿（before/after画像ペア）をスクロールして閲覧する
- 新着順で表示する（MVPでは推薦アルゴリズムなし）

### FR-06: いいね（評価）
- 投稿にいいねを付ける・取り消す
- いいね数を投稿に表示する

### FR-07: マイページ
- 自分の投稿一覧を確認する
- 自分がいいねした投稿一覧を確認する

---

## 非機能要件

### NFR-01: プラットフォーム
- **iOS のみ**（iPhone）
- 最小対応バージョン：iOS 16以上

### NFR-02: AI API
- **画像解析**：GPT-4V（OpenAI Vision API）
- **画像生成**：DALL-E 3（OpenAI Image API）
- APIキーはサーバーサイドで管理（クライアントに露出しない）

### NFR-03: バックエンド
- **言語・ランタイム**：Node.js / TypeScript
- **フレームワーク**：Express または Fastify
- RESTful API設計

### NFR-04: データベース
- **PostgreSQL**
- 主要テーブル：users / posts / likes
- ポイントシステムは後続フェーズのため、usersテーブルにpoints列を予約しておく（NULLable）

### NFR-05: クラウドインフラ（AWS）
- **画像ストレージ**：S3（before/after画像の保存）
- **バックエンド実行**：Lambda（サーバーレス）または EC2/ECS
- **DB**：RDS（PostgreSQL）
- **CDN**：CloudFront（画像配信の高速化）

### NFR-06: モバイルフロントエンド
- **React Native**（iOSビルドに対応）
- Expoを利用してカメラ・画像選択を実装

### NFR-07: パフォーマンス
- フィード初回表示：3秒以内
- AI変換レスポンス：10秒以内（OpenAI APIの応答速度に依存）
- 画像アップロード：5秒以内（Wi-Fi環境）

### NFR-08: セキュリティ
- MVPはPoC扱いのためセキュリティ拡張機能はスキップ
- 最低限の対応：HTTPS通信・JWTトークン認証・SQLインジェクション対策

### NFR-09: テスト（PBT部分適用）
- プロパティベーステストを**部分適用**
- MVP対象：AI変換パイプラインの入出力バリデーション関数
- ポイント計算ロジックは後続フェーズで実装時に適用

---

## 開発優先順位

**最優先（P0）**：カメラ→AI変換の体験（Capture & Refactor）
- FR-02（カメラ撮影）+ FR-03（AI変換）が動くことが最初のマイルストーン
- 「見て驚く」デモ品質を最初に担保する

**優先（P1）**：投稿・フィード・認証
- FR-01（認証）+ FR-04（投稿）+ FR-05（フィード）+ FR-06（いいね）

**通常（P2）**：マイページ
- FR-07（マイページ）

---

## データモデル概要（MVPスコープ）

```
users
  - id (UUID)
  - email (VARCHAR, UNIQUE)
  - password_hash (VARCHAR)
  - username (VARCHAR)
  - points (INTEGER, NULL) ← 後続フェーズ用予約列
  - created_at

posts
  - id (UUID)
  - user_id (FK → users)
  - before_image_url (VARCHAR)
  - after_image_url (VARCHAR)
  - category_tag (VARCHAR)
  - likes_count (INTEGER, DEFAULT 0)
  - created_at

likes
  - id (UUID)
  - user_id (FK → users)
  - post_id (FK → posts)
  - created_at
  - UNIQUE(user_id, post_id)
```

---

## 技術スタック確定サマリー

| レイヤー | 技術 |
|---------|------|
| モバイル | React Native（Expo） |
| バックエンド | Node.js / TypeScript（Express or Fastify） |
| AI — 画像解析 | GPT-4V（OpenAI） |
| AI — 画像生成 | DALL-E 3（OpenAI） |
| データベース | PostgreSQL（AWS RDS） |
| 画像ストレージ | AWS S3 + CloudFront |
| インフラ | AWS（Lambda or ECS, RDS, S3） |

---

## 拡張機能設定

| 拡張機能 | 設定 | 決定タイミング |
|---------|------|--------------|
| security-baseline | **無効（PoC扱い）** | 要件分析 |
| property-based-testing | **部分適用（AI変換バリデーション関数のみ）** | 要件分析 |
