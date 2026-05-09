# ユニット・オブ・ワーク定義 — Sloth Feed

> **本ドキュメントの前提（2026-05-09 更新）**
> Issue #5 帰着により、Sloth Feed は IP事業として位置づけ直された。
> 最新版：2026-05-09 / 改訂履歴は [`audit.md`](../../audit.md) と [`aidlc-state.md`](../../aidlc-state.md) を参照。

---

## 構成（変更なし）

| Unit | 名称 | 開発順序 |
|------|------|---------|
| Unit 1 | 認証（Auth）→ **Auth + IPファン識別基盤** | 1番目 |
| Unit 2 | 投稿 + AI（Post + AI）→ **ナマケモノ対話エンジン** | 2番目 |
| Unit 3 | フィード（Feed）→ **共同体タイムライン（サンドイッチUI）**| 3番目 |

---

## Unit 1 — Auth + IPファン識別基盤

**スコープ**: Next.js プロジェクト初期化 + **Cognito User Pool セットアップ** + **Auth.js (NextAuth v5) 設定** + 共通基盤
**意味的位置づけ**：**IP のファン識別装置**。ユーザーは商品ではなくファンとして登録され、個別化されたナマケモノとの関係性が始まる起点。

認証は Auth.js (NextAuth v5) + Cognito User Pool（OAuth/OIDC）。詳細は `component-methods.md` の「Authentication & Identity Flow」セクション参照。

### 含まれる機能

- Next.js 14+ App Router プロジェクトのスキャフォールディング
- **AWS Cognito User Pool セットアップ**（カスタム属性 `custom:name` 含む、自動メール検証 OFF は PoC 設定）
- **Auth.js (NextAuth v5) 設定**（`auth.ts` + `app/api/auth/[...nextauth]/route.ts`）
- **Cognito Provider 経由の登録・ログイン**フロー（OAuth/OIDC）
- **middleware.ts** で Auth.js の `auth` を default export し、保護対象パスを matcher で指定
- **app/providers.tsx** で SessionProvider 適用
- `lib/` 共通モジュールの初期化（後続ユニットで再利用）

→ **DynamoDB Users テーブルは作成しない**（Cognito 一本化、Phase 2 で補助テーブル検討）

### 生成するファイル

**プロジェクト基盤**
```
package.json
tsconfig.json
next.config.ts
.env.local.example
app/layout.tsx               # Next.js App Router root layout
app/globals.css              # 全画面共通スタイル
app/providers.tsx            # 新規：SessionProvider（Auth.js）等を集約
```

**共通モジュール（後続ユニットが再利用）**
```
lib/types/index.ts            # 共有型定義（User, Post, FilterResult, Pathway, NamakemonoResponse, AuthResult 等）
lib/db/client.ts              # DynamoDB クライアントシングルトン
lib/utils/errors.ts           # エラーハンドリングユーティリティ
```

**認証モジュール（Auth.js + Cognito）**
```
auth.ts                                  # 新規：Auth.js (NextAuth v5) 設定（Cognito Provider）
app/api/auth/[...nextauth]/route.ts      # Auth.js のルートハンドラ（auth.ts から GET/POST を再エクスポート）
middleware.ts                            # Auth.js の auth を default export、matcher 設定
components/AuthForm.tsx                  # PoC 実装時に決定（Hosted UI 利用なら不要）
app/auth/...                             # PoC 実装時に決定：Cognito Hosted UI / 自前ページ
```

### DynamoDB テーブル
- **PoC では Users テーブルは作成しない**（Cognito User Pool 一本化）
- Phase 2 で Sloth Feed 固有メタデータが必要になれば `{ userId: Cognito sub, ... }` 形式の補助テーブル追加検討

### 完了基準
- **Cognito User Pool が PoC 設定で作成されている**（カスタム属性 `custom:name`、自動メール検証 OFF）
- **Auth.js (NextAuth v5)** が `auth.ts` + `/api/auth/[...nextauth]` で設定済み
- `signIn("cognito")` でログインフローが起動する
- 登録フロー（PoC 実装時に決定：Cognito Hosted UI / 自前フォーム）でユーザー作成 → ログイン状態になる
- **`useSession()` でクライアント側から `{ id, name }` を取得できる**（`session.user.id` = Cognito sub、`session.user.name` = `custom:name`）
- **API Route で `await auth()`** すると同様に `{ id, name }` が取得できる
- **`middleware.ts`** が matcher 指定の保護ルート（`/api/posts`, `/api/my-posts`, `/post`, `/my-posts`）で Auth.js の認証を強制する
- **HttpOnly Cookie** にセッションが保存される（localStorage には JWT を保存しない）
- **DynamoDB Users テーブルが作成されていない**（PoC では Cognito 一本化、Phase 2 で再導入検討）

---

## Unit 2 — ナマケモノ対話エンジン（動的IPの核）

**スコープ**: 「仕事じゃないけど」投稿フロー全体（フィルタリング → AIナマケモノ対話 → 保存）
**意味的位置づけ**：**動的IPの本体**。AI ナマケモノが、**LLM の学習済み知識**と個別化記憶を駆使して、ユーザーごとに異なるナマケモノとの関係性を生み出す。**Sloth Feed のコア体験**を実装するユニット。Phase 2 で S3 + Agentic Search による引用検証を予定（FR-007 参照）。

**統合経緯（Unit 2 + Unit 3 → Unit 2）**：
当初は Unit 2（投稿 + AIフィルタリング）と Unit 3（AIコメント生成）を**別ユニット**として計画していたが、Units Generation 段階の `unit-of-work-plan.md` 質問1 で**統合決定**（B 回答）：
> PostService は内部で AIFilteringService と AINamakemonoService を**同じリクエスト内で連続して呼び出す**設計のため、Unit を分割して stub/mock を挟む意義が薄い。**部分失敗のハンドリング**（filtering_excluded / ai_generation_failed / persistence_failed）も同一フロー内で扱うのが自然。

→ 結果として 4ユニット → 3ユニット に集約され、**Unit 2 が動的IPの核として 5 ストーリー（US-003/004/005/006/009）を担当する責務集中ユニット**となっている。

### 含まれる機能

- 「**仕事じゃないけど**」投稿テキスト入力フォーム（**怠惰系・善行系両方を受容**、prefix 強制なし）
- **Amazon Bedrock 経由の Claude** による**仕事系投稿フィルタリング**（仕事の成果・旅行・充実投稿を除外）
- フィルタリング除外時の理由表示（例：「**仕事の成果はここでは扱いません。Sloth Feed は『仕事じゃないあなた』の場所です**」）
- フィルタリング通過後の **AI ナマケモノ対話**：
  - **「達観した怠惰の老師」人格**（FR-010）を System Prompt で固定（説教・押し付け・馴れ合い禁止）
  - **5経路（過剰生産抵抗 / 創造の余白 / 多様性保護 / 自己への暴力停止 / 集積による文化変容）のいずれかに紐付け**（FR-003 / FR-011）
  - **LLM の学習済み知識**から引用を生成（PoC は Bedrock Claude の事前学習を信用、Phase 2 で S3 + Agentic Search に拡張）
  - **個別化記憶**（FR-006）：UserHistory が過去投稿を取得して System Prompt に組み込む
  - **依存防止切り上げ提案**（FR-009 / US-009）：UserHistory.getActivityMetrics で連続投稿数・滞在時間を取得し、閾値超過時に老師人格で切り上げ提案
  - **出典明記**：PoC では LLM の自己申告（Phase 2 で事実検証）
- 投稿カードの **経路ラベル【経路X】表示**（FR-011）
- 投稿時に **authorName を User からスナップショット保存**（denormalization）
- 投稿 + AI コメント + 引用元 + 経路 + authorName を DynamoDB に保存
- 部分失敗ハンドリング（CreatePostResult enum：filtering_excluded / ai_generation_failed / persistence_failed）

### 前提条件
- Unit 1 完了（認証・`lib/` 共通モジュール・DynamoDB クライアント利用可能）

### 生成するファイル
```
lib/repositories/post.repository.ts
lib/services/ai-filtering.service.ts
lib/services/ai-namakemono.service.ts          # 動的IPの対話エンジン
lib/memory/user-history.ts                     # 新規：個別化記憶（FR-006）+ 活動メトリクス（FR-009）
lib/services/post.service.ts
app/api/posts/route.ts
app/(main)/post/page.tsx
components/PostForm.tsx
components/FilteringFeedback.tsx
components/NamakemonoBubble.tsx                # AI ナマケモノコメントの吹き出し
components/LoadingSpinner.tsx
```

### DynamoDB テーブル
- **Posts** テーブル（PK: postId、GSI: authorId-createdAt-index）
- スキーマ変更：`stamps` フィールド削除、`aiCitationSource` / `pathway` / `authorName` フィールド追加（Stage 5 で確定）

### 完了基準
- `POST /api/posts` に「**布団から3時間出られなかった**」（怠惰系）を送ると Bedrock Claude がフィルタリング判定し通過する
- 「**彼に洗い物しといた**」（善行系）も**等しく通過**する
- 仕事の成果・旅行投稿が 422 で返される（メッセージ：「**仕事の成果はここでは扱いません。Sloth Feed は『仕事じゃないあなた』の場所です**」）
- 通過した投稿に対し、AI ナマケモノが**「達観した怠惰の老師」人格**（FR-010）で応答する：
  - 5経路のいずれかに紐付ける（FR-011 経路ラベル付き）
  - Larry Wall・ラッセル・老子・ケインズ等の引用を生成（LLM 自己申告）
  - 出典が明記される
- 連続投稿5件以上 / 滞在30分以上で**切り上げ提案**が出る（FR-009 / US-009）
- ユーザーごとに個別化された応答が生成される（過去投稿を参照、FR-006）
- DynamoDB Posts テーブルに `aiCitationSource` / `pathway` / `authorName` が保存される
- **🦥 はヘッダラベルにのみ使用、本文には入れない**（FR-010）

---

## Unit 3 — 共同体タイムライン（サンドイッチUI）

**スコープ**: タイムライン閲覧・自分のダメ投稿一覧閲覧
**意味的位置づけ**：**ファン共同体の場**。みんなのダメが並ぶことで「私だけじゃない」という感覚を生む。**サンドイッチUI（BrandFrame）**でブランド構文「仕事じゃないけど…世の中を変える」を空間化する。

### 含まれる機能

- 全ユーザーの「**仕事じゃないけど**」投稿タイムライン（新しい順、未ログインで閲覧可能、PoC では最近 50件まで）
- 自分の過去「仕事じゃないけど」投稿一覧（ログイン必須）
- 各投稿に **AI ナマケモノコメント + 経路ラベル【経路X】 + 引用元 + authorName** を表示
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
- `GET /api/feed` が未認証でもタイムライン（怠惰系・善行系混在）を返す（PoC 最近 50件）
- `GET /api/my-posts` が JWT 必須で自分の「仕事じゃないけど」投稿一覧を返す
- タイムラインページ（`/`）が**サンドイッチUI**で表示される（BrandFrame 上下フレーム）
- 各投稿に **🦥 ヘッダラベル + 【経路X】 + 引用元** が表示される
- **authorName** が表示される（authorId ではない）
- いいね数・フォロワー数・ランキングが**一切表示されない**
- **ビジョン「『仕事じゃないけど、、、』が世の中を変える」**が UI を通じて伝わる

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
| `lib/memory/` | Unit 2 で初期化 | **個別化記憶（FR-006）+ 活動メトリクス（FR-009）** |
| `lib/services/post.service.ts` | Unit 2 | ダメ投稿オーケストレーション |
| `lib/services/feed.service.ts` | Unit 3 | フィード取得 |
| `components/BrandFrame.tsx` | Unit 3 | **サンドイッチUI 上下フレーム** |
