# サービス定義 — Sloth Feed

## サービスレイヤーの役割

```
API Route（薄いコントローラ）
     ↓ 呼び出し
Service（ビジネスロジック・オーケストレーション）
     ↓ 呼び出し
Repository（DynamoDB アクセス）または外部API（Claude API）
```

---

## PostService — 投稿作成オーケストレーター

PostService は Sloth Feed のコアフローを担う中心サービス。
1つの `createPost` 呼び出しの中で、フィルタリング・コメント生成・永続化を順番に実行する。

```
createPost(authorId, content)
  │
  ├─[1] AIFilteringService.filterPost(content)
  │        ├─ allowed: false  →  { success: false, reason }  を返す（終了）
  │        └─ allowed: true   →  続行
  │
  ├─[2] AICommentService.generateComment(content)
  │        └─ aiComment: string
  │
  └─[3] PostRepository.create({ content, authorId, aiComment })
           └─  { success: true, post }  を返す
```

**依存サービス**: AIFilteringService, AICommentService, PostRepository

---

## AuthService — 認証オーケストレーター

ユーザー登録・ログインの流れを管理し、JWT を発行する。

```
register(email, password)
  ├─[1] UserRepository.findByEmail(email)  → 重複チェック
  ├─[2] bcrypt.hash(password)
  ├─[3] UserRepository.create({ email, passwordHash })
  └─[4] JWT 発行 → { userId, token }

login(email, password)
  ├─[1] UserRepository.findByEmail(email)  → ユーザー取得
  ├─[2] bcrypt.compare(password, user.passwordHash)
  └─[3] JWT 発行 → { userId, token }
```

**依存**: UserRepository, bcrypt, jsonwebtoken

---

## FeedService — データ取得サービス

タイムラインと自分の投稿一覧を取得する読み取り専用サービス。
ビジネスロジックは持たず、Repository への委譲を行う。

```
getTimeline(limit, lastKey?)
  └─ PostRepository.findAll(limit, lastKey)
       └─ PaginatedResult<Post>

getUserPosts(authorId, limit, lastKey?)
  └─ PostRepository.findByAuthorId(authorId, limit, lastKey)
       └─ PaginatedResult<Post>
```

**依存**: PostRepository

---

## AIFilteringService — Claude API フィルタリング

Claude API に投稿内容を送り、Sloth Feed のルールに合致するかを判定する。

- **通過条件**: 仕事・旅行・スポーツ大会・自己啓発成果に該当しない投稿
- **除外条件**: 仕事の成果・昇進・旅行・観光・スポーツ大会結果・自己啓発成果
- 除外時は Claude が日本語で除外理由を生成する（LLM の推論による）

**依存**: Anthropic SDK (@anthropic-ai/sdk), 環境変数 `ANTHROPIC_API_KEY`

---

## AICommentService — Claude API コメント生成

フィルタリング通過後の投稿内容をもとに、称賛コメントを生成する。

- 偉人の名言 / 研究論文 / 心理学的知見のいずれかを引用
- 例: 「ハーバードの研究によると、小さな親切の積み重ねが精神的幸福に最も寄与するとされています。あなたは今日それを実践しました。」
- 出力は必ず 1〜3 文の短いコメント

**依存**: Anthropic SDK (@anthropic-ai/sdk), 環境変数 `ANTHROPIC_API_KEY`

---

## サービス間の責務分離サマリー

| サービス | 外部依存 | ビジネスロジック | オーケストレーション |
|---------|---------|----------------|-----------------|
| AuthService | UserRepository, bcrypt, JWT | 認証ルール | ○ |
| PostService | AIFilteringService, AICommentService, PostRepository | なし（委譲のみ） | ○（コアフロー） |
| FeedService | PostRepository | なし | △（委譲のみ） |
| AIFilteringService | Claude API | フィルタリング判定 | なし |
| AICommentService | Claude API | コメント生成 | なし |

---

## 環境変数一覧

| 変数名 | 用途 |
|--------|------|
| `ANTHROPIC_API_KEY` | Claude API 認証 |
| `DYNAMODB_USERS_TABLE` | Users テーブル名 |
| `DYNAMODB_POSTS_TABLE` | Posts テーブル名 |
| `AWS_REGION` | DynamoDB リージョン |
| `JWT_SECRET` | JWT 署名シークレット |
| `JWT_EXPIRES_IN` | JWT 有効期限（例: `7d`） |
