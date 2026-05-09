# サービス定義 — Sloth Feed

> 最新版：2026-05-09 / 改訂履歴は [`audit.md`](../../audit.md) と [`aidlc-state.md`](../../aidlc-state.md) を参照。

## サービスレイヤーの役割

```
API Route（薄いコントローラ）
     ↓ 呼び出し
Service（ビジネスロジック・オーケストレーション）
     ↓ 呼び出し
Repository（DynamoDB アクセス）または AWS SDK（Bedrock 経由 Claude）
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

## AIFilteringService — Bedrock Claude フィルタリング

**Amazon Bedrock 経由で Claude モデル**に投稿内容を送り、Sloth Feed のルールに合致するかを判定する。

- **通過条件**: 仕事・旅行・スポーツ大会・自己啓発成果に該当しない投稿（怠惰系・善行系問わず）
- **除外条件**: 仕事の成果・昇進・旅行・観光・スポーツ大会結果・自己啓発成果
- 除外時は Claude が日本語で除外理由を生成する（LLM の推論による）

**依存**: AWS SDK for JavaScript v3 (`@aws-sdk/client-bedrock-runtime`)、環境変数 `AWS_REGION` / `BEDROCK_MODEL_ID`、IAM 認証情報（ローカルは AccessKey、本番は IAM ロール推奨）

---

## AINamakemonoService — Bedrock Claude × 個別化記憶（動的IPの核）

フィルタリング通過後の投稿内容をもとに、AI ナマケモノの肯定コメントを生成する。**5経路のいずれか**に紐付け、**LLM の学習済み知識**から引用を引き出して「**何が、どう、世の中を変えるのか**」を具体的に言語化する。

- 5経路（過剰生産抵抗 / 創造の余白 / 多様性保護 / 自己への暴力停止 / 集積による文化変容）から1つ選択
- **LLM の学習済み知識**から経路に対応する偉人引用を生成（Larry Wall・ラッセル・老子・ニュートン・フレミング・ケインズ等）
- 想定引用源マッピング（FR-007）はプロンプトのヒントとして提示
- ユーザー履歴を踏まえた個別化記憶を System Prompt に組み込む
- 出典明記（LLM の自己申告ベース）
- 出力は経路ラベル + 本文 + 引用元の3要素

**依存**: AWS SDK for JavaScript v3 (`@aws-sdk/client-bedrock-runtime`)、UserHistory、環境変数 `AWS_REGION` / `BEDROCK_MODEL_ID`、IAM 認証情報

**Phase 2 構想**: S3 + Agentic Search による引用検証（`@aws-sdk/client-s3` + Bedrock Agents または Claude tool use で S3 検索ツールを呼び出す）。FR-007 参照。

---

## サービス間の責務分離サマリー

| サービス | 外部依存 | ビジネスロジック | オーケストレーション |
|---------|---------|----------------|-----------------|
| PostService | AIFilteringService, AINamakemonoService, PostRepository | なし（委譲のみ） | ○（コアフロー） |
| FeedService | PostRepository | なし | △（委譲のみ） |
| AIFilteringService | Amazon Bedrock（Claude）| フィルタリング判定 | なし |
| AINamakemonoService | Amazon Bedrock（Claude）, 個別化記憶 | キャラ人格 + 5経路選択 + LLM 学習済み引用生成 | なし |
| **auth.ts (Auth.js)** | Cognito User Pool（OAuth/OIDC）| 認証は Auth.js + Cognito に委譲 | ○（Auth.js が OAuth フロー全体をオーケストレート）|

---

## 環境変数一覧

| 変数名 | 用途 |
|--------|------|
| `AWS_REGION` | AWS リージョン（Bedrock・DynamoDB 共通）|
| `BEDROCK_MODEL_ID` | 利用する Claude モデル ID（例：`anthropic.claude-3-5-sonnet-20241022-v2:0` または最新版）|
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` または IAM ロール | 開発時はキー、本番ではタスクロール／インスタンスロール推奨 |
| `DYNAMODB_POSTS_TABLE` | Posts テーブル名 |
| `COGNITO_USER_POOL_ID` | Cognito ユーザープール ID |
| `COGNITO_APP_CLIENT_ID` | Cognito App Client ID |
| `COGNITO_APP_CLIENT_SECRET` | Cognito App Client Secret（Auth.js OAuth が要求）|
| `AUTH_SECRET` | Auth.js Cookie 暗号化用ランダム文字列 |
| `NEXTAUTH_URL` | アプリ Base URL（例: `http://localhost:3000`）|
