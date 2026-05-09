# コンポーネント定義 — Sloth Feed

> 最新版：2026-05-09 / 改訂履歴は [`audit.md`](../../audit.md) と [`aidlc-state.md`](../../aidlc-state.md) を参照。

## アーキテクチャ概要

```
app/                        # Next.js App Router ページ
components/                 # 再利用 UI コンポーネント
lib/
  services/                 # ビジネスロジック（薄いコントローラの背後）
  repositories/             # DynamoDB アクセス抽象化（PostRepository のみ。UserRepository は Cognito 一本化により PoC 外）
  memory/                   # 個別化記憶（UserHistory）
  types/                    # 共有型定義
auth.ts                     # 新規：Auth.js (NextAuth v5) 設定（Cognito Provider）
middleware.ts               # Auth.js auth に委譲（Cookie ベースのセッション検証）
```

**Phase 2 で追加予定**: `lib/agents/`（S3 + Agentic Search ツール、FR-007 参照）

---

## バックエンド・コンポーネント

### 1. PostService
| 項目 | 内容 |
|------|------|
| **パス** | `lib/services/post.service.ts` |
| **責務** | 投稿作成フローの全体オーケストレーション（フィルタリング → ナマケモノ対話 → 保存） |
| **依存** | PostRepository, AIFilteringService, AINamakemonoService |

### 2. FeedService
| 項目 | 内容 |
|------|------|
| **パス** | `lib/services/feed.service.ts` |
| **責務** | タイムライン取得・自分の投稿一覧取得 |
| **依存** | PostRepository |

### 3. AIFilteringService
| 項目 | 内容 |
|------|------|
| **パス** | `lib/services/ai-filtering.service.ts` |
| **責務** | **Amazon Bedrock 経由で Claude モデル**を呼び出し、投稿が「仕事外」（**怠惰系・善行系問わず**）かを判定する。仕事の成果・キラキラ充実投稿のみ除外、除外時は理由も生成 |
| **依存** | AWS SDK for JavaScript v3 (`@aws-sdk/client-bedrock-runtime`) |

### 4. AINamakemonoService（**動的IPの核**）
| 項目 | 内容 |
|------|------|
| **パス** | `lib/services/ai-namakemono.service.ts` |
| **責務** | **動的IP の本体**。Amazon Bedrock 経由で Claude モデルを呼び出し、AI ナマケモノ（**「達観した怠惰の老師」人格** / FR-010）として個別化された肯定コメントを生成する。投稿内容を **5経路（過剰生産抵抗 / 創造の余白 / 多様性保護 / 自己への暴力停止 / 集積による文化変容）のいずれかに紐付け**、LLM の学習済み知識から偉人・科学・歴史の引用を生成する（PoC：FR-007）。**個別化記憶（FR-006）**・**依存防止切り上げ提案（FR-009）**・**経路ラベル付き出力（FR-011）**を担う |
| **依存** | AWS SDK for JavaScript v3 (`@aws-sdk/client-bedrock-runtime`)、`UserHistory`（個別化記憶） |
| **Phase 2 構想** | S3 + Agentic Search による引用検証（PoC には含まれない） |

### 5. PostRepository
| 項目 | 内容 |
|------|------|
| **パス** | `lib/repositories/post.repository.ts` |
| **責務** | Posts テーブルの CRUD + GSI によるユーザー別検索 |
| **依存** | AWS SDK v3 |

### 6. auth.ts（**新規・Auth.js 設定**）
| 項目 | 内容 |
|------|------|
| **パス** | プロジェクトルート `auth.ts`（または `lib/auth.ts`）|
| **責務** | Auth.js (NextAuth v5) の設定。Cognito Provider・Session callback・matcher で middleware の保護対象を設定 |
| **エクスポート** | `handlers`（`/api/auth/[...nextauth]/route.ts` で再エクスポート）/ `signIn` / `signOut` / `auth`（API Route や middleware で使用）|
| **依存** | `next-auth`、`next-auth/providers/cognito` |

---

## フロントエンド・コンポーネント

### ページコンポーネント（`app/`）

| ページ | パス | 説明 |
|--------|------|------|
| タイムライン | `app/(main)/page.tsx` | 全ユーザーの投稿フィード |
| 投稿フォーム | `app/(main)/post/page.tsx` | テキスト入力 → フィルタリング → 結果表示 |
| 自分の投稿 | `app/(main)/my-posts/page.tsx` | 自分の過去投稿一覧 |
| ログイン | `app/auth/login/page.tsx` | ログインフォーム |
| 新規登録 | `app/auth/register/page.tsx` | 新規登録フォーム |

### 再利用 UI コンポーネント（`components/`）

| コンポーネント | パス | 責務 |
|---------------|------|------|
| PostCard | `components/PostCard.tsx` | 投稿本文 + AI ナマケモノコメント + 経路ラベルを**サンドイッチUI**で表示 |
| PostForm | `components/PostForm.tsx` | テキスト入力欄・送信ボタン・バリデーション |
| FeedList | `components/FeedList.tsx` | PostCard の一覧レンダリング（PoC では最近 50件のみ表示。**Phase 2 でページネーション対応**を予定）|
| NamakemonoBubble | `components/NamakemonoBubble.tsx` | AI ナマケモノコメントの吹き出し表示。**【経路X】ラベル**（FR-011）・**🦥 ヘッダ**・**引用元（aiCitationSource）**を含む |
| **BrandFrame** | `components/BrandFrame.tsx` | **サンドイッチUI 上下フレーム**。ブランド構文「仕事じゃないけど…これが世の中を変える」を投稿カード上下で保証（FR-008）|
| FilteringFeedback | `components/FilteringFeedback.tsx` | フィルタリング除外時の理由メッセージ表示 |
| AuthForm | `components/AuthForm.tsx` | **PoC 実装時に決定**：Cognito Hosted UI 利用なら不要、自前フォーム採用なら Auth.js の `signIn` / Cognito SignUp API を呼び出す共通フォーム |
| SessionProvider | `app/providers.tsx` 内 | クライアントコンポーネント全体に Auth.js の SessionProvider を適用（`useSession()` を使えるようにする）|
| LoadingSpinner | `components/LoadingSpinner.tsx` | Bedrock Claude 呼び出し中のローディング表示 |

---

## API Route コンポーネント（`app/api/`）

| エンドポイント | パス | 責務 |
|--------------|------|------|
| /api/auth/[...nextauth] | `app/api/auth/[...nextauth]/route.ts` | **Auth.js のルートハンドラ**（`auth.ts` から GET/POST を再エクスポート）。Cognito OAuth フローのコールバック・signIn・signOut すべてをここで処理 |
| POST /api/posts | `app/api/posts/route.ts` | 投稿作成（フィルタリング〜コメント生成まで） |
| GET /api/feed | `app/api/feed/route.ts` | タイムライン取得 |
| GET /api/my-posts | `app/api/my-posts/route.ts` | 自分の投稿一覧取得 |

---

## ミドルウェア

| コンポーネント | パス | 責務 |
|--------------|------|------|
| Auth.js Middleware | `middleware.ts` | **Auth.js の `auth` を default export**。Cookie ベースのセッション検証を Auth.js に委譲。matcher で保護対象パスを指定 |

**保護対象ルート**: `/api/posts`, `/api/my-posts`, `/post`, `/my-posts`
**非保護ルート**: `/api/auth/*`, `/api/feed`（閲覧はログイン不要）

→ 詳細実装は `component-methods.md` の「Authentication & Identity Flow」セクション参照
