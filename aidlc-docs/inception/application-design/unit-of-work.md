# ユニット・オブ・ワーク定義 — Sloth Feed

> **本ドキュメントの前提（2026-05-09 更新）**
> Issue #5 帰着により、Sloth Feed は IP事業として位置づけ直された。
> **3ユニットの構造変更はない**が、各ユニットの**責務・意味を「動的IP × AI技術」文脈で再記述**している。

---

## 構成（変更なし）

| Unit | 名称（旧 → 新）| 開発順序 |
|------|------|---------|
| Unit 1 | 認証（Auth）→ **Auth + IPファン識別基盤** | 1番目 |
| Unit 2 | 投稿 + AI（Post + AI）→ **ナマケモノ対話エンジン** | 2番目 |
| Unit 3 | フィード（Feed）→ **共同体タイムライン（サンドイッチUI）**| 3番目 |

---

## Unit 1 — Auth + IPファン識別基盤

**スコープ**: Next.js プロジェクト初期化 + メール・パスワード認証 + 共通基盤
**意味的位置づけ**：**IP のファン識別装置**。ユーザーは商品ではなくファンとして登録され、個別化されたナマケモノとの関係性が始まる起点。

### 含まれる機能

- Next.js 14+ App Router プロジェクトのスキャフォールディング
- メールアドレス＋パスワードによるユーザー登録
- ログイン・JWT 発行・JWT 検証ミドルウェア
- DynamoDB Users テーブルの作成
- `lib/` 共通モジュールの初期化（後続ユニットで再利用）

### 生成するファイル

**プロジェクト基盤**
```
package.json
tsconfig.json
next.config.ts
.env.local.example
```

**共通モジュール（後続ユニットが再利用）**
```
lib/types/index.ts            # 共有型定義（User, Post, FilterResult, Citation 等）
lib/db/client.ts              # DynamoDB クライアントシングルトン
lib/utils/errors.ts           # エラーハンドリングユーティリティ
```

**Auth モジュール**
```
lib/repositories/user.repository.ts
lib/services/auth.service.ts
app/api/auth/register/route.ts
app/api/auth/login/route.ts
app/auth/login/page.tsx
app/auth/register/page.tsx
components/AuthForm.tsx
middleware.ts
```

### DynamoDB テーブル
- **Users** テーブル（PK: userId、GSI: email-index）

### 完了基準
- `POST /api/auth/register` でユーザー（IPファン）を作成できる
- `POST /api/auth/login` で JWT を取得できる
- `middleware.ts` が保護ルートで JWT を検証し、`x-user-id` ヘッダを付与する

---

## Unit 2 — ナマケモノ対話エンジン（動的IPの核）

**スコープ**: ダメ投稿フロー全体（フィルタリング → AIナマケモノ対話 → 保存）
**意味的位置づけ**：**動的IPの本体**。AI ナマケモノが、RAG引用ライブラリと個別化記憶を駆使して、ユーザーごとに異なるナマケモノとの関係性を生み出す。**Sloth Feed のコア体験**を実装するユニット。

### 含まれる機能

- ダメ投稿テキスト入力フォーム（「仕事じゃないけど」prefix 強制なし）
- Claude API による**仕事系投稿フィルタリング**（仕事の成果・旅行・充実投稿を除外）
- フィルタリング除外時の理由表示（「ここはダメを誇る場所です」）
- フィルタリング通過後の **AI ナマケモノ対話**：
  - **RAG 引用ライブラリ**（Larry Wall・ラッセル・老子・ニュートン・フレミング等）から引用検索
  - **ユーザーの過去投稿履歴**（個別化記憶）を AI コンテキストに含める
  - **ナマケモノキャラクター人格**を System Prompt で固定
  - **出典明記**（ハルシネーション対策）
- ダメ投稿 + AI コメント + 引用元の DynamoDB 保存

### 前提条件
- Unit 1 完了（認証・`lib/` 共通モジュール・DynamoDB クライアント利用可能）

### 生成するファイル
```
lib/repositories/post.repository.ts
lib/services/ai-filtering.service.ts
lib/services/ai-namakemono.service.ts          # 旧 ai-comment.service.ts を改名・拡張
lib/rag/citations.json                         # 新規：偉人引用データベース
lib/rag/retriever.ts                           # 新規：引用検索ロジック
lib/memory/user-history.ts                     # 新規：個別化記憶
lib/services/post.service.ts
app/api/posts/route.ts
app/(main)/post/page.tsx
components/PostForm.tsx
components/FilteringFeedback.tsx
components/NamakemonoBubble.tsx                # 旧 AICommentBubble を改名
components/LoadingSpinner.tsx
```

### DynamoDB テーブル
- **Posts** テーブル（PK: postId、GSI: authorId-createdAt-index）
- スキーマ変更：`stamps` フィールド削除、`aiCitationSource` フィールド追加

### 完了基準
- `POST /api/posts` に「**布団から3時間出られなかった**」を送ると Claude がフィルタリング判定する
- 仕事の成果・旅行投稿が 422（「ダメを誇る場所です」のメッセージ付き）で返される
- 「布団から3時間出られなかった」のような**真のダメ投稿**が通過し、AI ナマケモノが**Larry Wall や老子の引用付き**で肯定コメントを生成する
- AI コメントには**出典が明記**される
- ユーザーごとに個別化された応答が生成される

---

## Unit 3 — 共同体タイムライン（サンドイッチUI）

**スコープ**: タイムライン閲覧・自分のダメ投稿一覧閲覧
**意味的位置づけ**：**ファン共同体の場**。みんなのダメが並ぶことで「私だけじゃない」という感覚を生む。**サンドイッチUI（BrandFrame）**でブランド構文「仕事じゃないけど…世の中を変える」を空間化する。

### 含まれる機能

- 全ユーザーのダメ投稿タイムライン（新しい順、未ログインで閲覧可能）
- 自分の過去ダメ投稿一覧（ログイン必須）
- 各投稿に **AI ナマケモノコメント + 引用元** を表示
- **サンドイッチUI構造**：投稿カード上下に「仕事じゃないけど」「これが世の中を変える」のブランドフレームが常に表示される

### 前提条件
- Unit 1 完了（認証・`lib/` 共通モジュール）
- Unit 2 完了（PostRepository・Posts テーブル利用可能）

### 生成するファイル
```
lib/services/feed.service.ts
app/api/feed/route.ts
app/api/my-posts/route.ts
app/(main)/page.tsx                # タイムライン（未ログインでも閲覧可、サンドイッチUI）
app/(main)/my-posts/page.tsx       # 自分のダメ投稿一覧
components/PostCard.tsx            # サンドイッチUIで投稿+AIコメント表示
components/FeedList.tsx
components/BrandFrame.tsx          # 新規：サンドイッチUIの上下フレーム
```

### 完了基準
- `GET /api/feed` が未認証でもタイムラインを返す
- `GET /api/my-posts` が JWT 必須で自分のダメ投稿一覧を返す
- タイムラインページ（`/`）が**サンドイッチUI**で表示される
- いいね数・フォロワー数・ランキングが**一切表示されない**

---

## 実行順序

```
Unit 1（Auth + IPファン識別）
  └─→ Unit 2（ナマケモノ対話エンジン）   ← Unit 1 の lib/ 共通モジュールを利用
          └─→ Unit 3（共同体タイムライン）  ← Unit 1・2 の Repository・テーブルを利用
```

---

## コード整理戦略

すべてのユニットは **1 つの Next.js モノリポジトリ** 内に配置する。

| フォルダ | 所有ユニット | 説明 |
|---------|------------|------|
| `lib/types/` | Unit 1 で初期化、全ユニット利用 | 共有型定義 |
| `lib/db/` | Unit 1 で初期化、全ユニット利用 | DynamoDB クライアント |
| `lib/utils/` | Unit 1 で初期化、全ユニット利用 | 共通ユーティリティ |
| `lib/repositories/user.repository.ts` | Unit 1 | Users テーブルアクセス |
| `lib/services/auth.service.ts` | Unit 1 | 認証ロジック |
| `lib/repositories/post.repository.ts` | Unit 2 | Posts テーブルアクセス |
| `lib/services/ai-filtering.service.ts` | Unit 2 | 仕事系投稿の弾き |
| `lib/services/ai-namakemono.service.ts` | Unit 2 | **動的IP対話エンジン** |
| `lib/rag/` | Unit 2 で初期化 | **RAG 引用ライブラリ** |
| `lib/memory/` | Unit 2 で初期化 | **個別化記憶** |
| `lib/services/post.service.ts` | Unit 2 | ダメ投稿オーケストレーション |
| `lib/services/feed.service.ts` | Unit 3 | フィード取得 |
| `components/BrandFrame.tsx` | Unit 3 | **サンドイッチUI 上下フレーム** |

---

## 旧版からの主な変更点

**構造変更**：なし。3ユニットの境界・依存関係は維持。

**意味的変更**：

| Unit | 旧の責務 | 新の責務（動的IP × AI 観点）|
|---|---|---|
| Unit 1 | 認証 | **IPファン識別基盤** |
| Unit 2 | 投稿 + AI | **ナマケモノ対話エンジン**（動的IPの核：AIキャラ人格 + RAG + 個別化記憶）|
| Unit 3 | フィード | **共同体タイムライン**（サンドイッチUI でブランド構文を空間化）|

**新規追加要素**（既存ユニット内）：

- Unit 2: `lib/rag/`（引用ライブラリ）、`lib/memory/`（個別化記憶）
- Unit 3: `components/BrandFrame.tsx`（サンドイッチUI）
- Posts スキーマ：`aiCitationSource` 追加、`stamps` 削除
