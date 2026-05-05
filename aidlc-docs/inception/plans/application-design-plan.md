# アプリケーション設計計画 — Sloth-Lab

## 設計チェックリスト

- [x] ステップ 1: コンテキスト分析（requirements.md / stories.md 読み込み済み）
- [x] ステップ 2: 設計計画の作成
- [x] ステップ 3: ユーザー回答の収集（Q1=A, Q2=B, Q3=A）
- [x] ステップ 4: 設計成果物の生成
  - [x] components.md
  - [x] component-methods.md
  - [x] services.md
  - [x] component-dependency.md
  - [x] application-design.md（統合ドキュメント）

---

## 識別したコンポーネント（回答前の初期分析）

要件とユーザーストーリーの分析から、以下のコンポーネントが想定されます：

| コンポーネント | 技術 | 役割 |
|--------------|------|------|
| iOS Main App | Swift | 位置情報権限管理、デバイスID管理 |
| Widget Extension | Swift / WidgetKit | ホーム画面UI、タイムライン管理 |
| AppIntent | Swift / App Intents | 「だるい」タップ処理、API呼び出し、reloadTimelines |
| Backend API | Node.js / TypeScript | 位置情報受信・近接カウント返却 |
| Data Access Layer | TypeScript | DynamoDB の読み書き抽象化 |
| Infrastructure | AWS CDK | DynamoDB / Lambda / API Gateway |

---

## 確認が必要な設計上の意思決定

以下の3点は設計に直接影響するため、回答をお願いします。

---

### 質問 1: バックエンドのデプロイ形態

バックエンド API をどのように実装・デプロイしますか？

A) **AWS Lambda + API Gateway**（サーバーレス）  
　　→ Cold start リスクあり。CDK で簡単にデプロイ可能。ハッカソン向き。

B) **Express/Fastify サーバー**（ECS Fargate / EC2）  
　　→ 常時起動。Cold start なし。インフラコスト・複雑度が高い。

C) **その他** (その他の場合は [Answer]: タグの後に記述してください)

[Answer]: A

---

### 質問 2: Widget のデータフロー

「だるい」をタップした後のデータの流れをどうしますか？

A) **AppIntent → API呼び出し → AppGroup に保存 → Widget が AppGroup から読み取る**  
　　→ AppGroup（共有 UserDefaults）を使用。Widget は API を直接呼ばない。  
　　→ AppIntent がカウントを取得し AppGroup に書き込む → reloadTimelines → Widget は AppGroup を読む。

B) **AppIntent → API呼び出し → reloadTimelines → Widget が直接 API を呼び出す**  
　　→ Widget の TimelineProvider が毎回 API を直接呼び出す。  
　　→ AppGroup 不要。Widget の自動更新でも常に最新データを取得。

C) **その他** (その他の場合は [Answer]: タグの後に記述してください)

[Answer]: B

---

### 質問 3: 近接クエリの実装方法

DynamoDB で「半径3km以内」のユーザーを取得する方法を選択してください。

A) **バウンディングボックス方式**（シンプル、PoC向き）  
　　→ 緯度・経度でおよそ±0.027°（≒3km）のバウンディングボックスを計算してフィルタリング。  
　　→ DynamoDB のスキャン or GSI + Lambda でフィルタリング。  
　　→ 実装が単純。精度は若干甘い（正方形近似）がデモには十分。

B) **Geohash 方式**（より正確、少し複雑）  
　　→ 位置情報を Geohash でエンコードし、近隣セルを検索。  
　　→ DynamoDB の GSI で効率的なクエリが可能。  
　　→ 本番想定の設計。実装がやや複雑。

C) **その他** (その他の場合は [Answer]: タグの後に記述してください)

[Answer]: A

---

## 次のステップ

上記3つの質問に回答後、「完了しました」と送信してください。  
回答を分析し、設計成果物（components.md / component-methods.md / services.md / component-dependency.md）を生成します。
