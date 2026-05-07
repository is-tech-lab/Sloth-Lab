# アプリケーション設計計画 — Sloth Feed

## 生成チェックリスト

- [x] ステップA: 質問への回答を読み込む
- [x] ステップB: components.md を生成する
- [x] ステップC: component-methods.md を生成する
- [x] ステップD: services.md を生成する
- [x] ステップE: component-dependency.md を生成する
- [x] ステップF: application-design.md（統合版）を生成する

---

## アプリケーション設計に関する質問

以下の質問に回答してください。`[Answer]:` タグの後に回答を記入してください。

---

## 質問 1 — コンポーネント識別

AIフィルタリングとAIコメント生成は、それぞれどのように整理しますか？

A) **分離** — `AIFilteringService`（投稿内容の判定）と `AICommentService`（称賛コメント生成）を別々のサービスとして設計する
B) **統合** — `AIService` として一つにまとめ、フィルタリングとコメント生成を内部メソッドで分担させる
C) Other (その他の場合は [Answer]: タグの後に記述してください)

[Answer]: A

---

## 質問 2 — サービスレイヤー設計

Next.js API Routes のビジネスロジックをどこに置きますか？

A) **サービスクラス分離** — API Route は薄いコントローラとし、ビジネスロジックは `lib/services/` 配下の TypeScript クラス（AuthService / PostService / FeedService 等）に集約する
B) **API Route 直書き** — 各 API Route ファイル内にロジックを直接記述する（PoC のシンプルさを優先）
C) Other (その他の場合は [Answer]: タグの後に記述してください)

[Answer]: A

---

## 質問 3 — DynamoDB アクセスパターン

DynamoDB へのアクセスをどのように設計しますか？

A) **リポジトリパターン** — `UserRepository`・`PostRepository` のようなクラスを用意し、DynamoDB の操作を抽象化する。サービスはリポジトリを経由してデータにアクセスする
B) **サービス直アクセス** — リポジトリレイヤーを設けず、サービスクラスから DynamoDB SDK を直接呼び出す
C) Other (その他の場合は [Answer]: タグの後に記述してください)

[Answer]: A

---

## 質問 4 — DynamoDB テーブル設計

DynamoDB のテーブル構成をどうしますか？

A) **シングルテーブル設計** — 1 つのテーブルにすべてのエンティティ（User・Post）を収め、PK/SK のプレフィックスで識別する（例: `PK: USER#userId`, `PK: POST#postId`）
B) **テーブル分割** — `Users` テーブルと `Posts` テーブルを別々に作成する（シンプルで直感的）
C) Other (その他の場合は [Answer]: タグの後に記述してください)

[Answer]: B

---

## 質問 5 — JWT 認証の実装位置

JWT の検証をどこで行いますか？

A) **ミドルウェア** — `middleware.ts` で一括検証。保護されたルートへのリクエストはすべて自動的に JWT チェックを受ける
B) **各 API Route 内** — 保護が必要な API Route それぞれの先頭で JWT 検証ロジックを呼び出す
C) Other (その他の場合は [Answer]: タグの後に記述してください)

[Answer]: A

---

## 質問 6 — フロントエンド・コンポーネント設計

Next.js の UI コンポーネントをどのように整理しますか？

A) **ページ + 再利用コンポーネント分離** — `app/` 配下のページコンポーネントと、`components/` 配下の再利用 UI コンポーネント（PostCard・FeedList・AICommentBubble 等）に分ける
B) **ページ中心** — 再利用コンポーネントは最小限に抑え、各ページファイル内にロジックと UI を集約する（PoC のシンプルさ優先）
C) Other (その他の場合は [Answer]: タグの後に記述してください)

[Answer]: A
