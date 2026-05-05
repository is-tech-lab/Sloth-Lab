# ユニット・オブ・ワーク定義 — Sloth-Lab

**管理構造**: モノレポ（1リポジトリ内に全ユニットを配置）

---

## ユニット一覧

| # | ユニット名 | 技術スタック | 配置場所 |
|---|-----------|------------|---------|
| 1 | `backend-api` | Node.js / TypeScript / AWS Lambda | `/backend/` |
| 2 | `ios-app` | Swift / WidgetKit / App Intents | `/ios/` |
| 3 | `infrastructure` | AWS CDK (TypeScript) | `/infrastructure/` |

---

## ユニット 1: `backend-api`

### 責務
- `POST /darui` のリクエスト処理（位置情報保存 + 近接カウント返却）
- `GET /darui/count` のリクエスト処理（近接カウントのみ返却）
- DynamoDB への読み書き操作
- バウンディングボックス方式による近接クエリロジック

### 含まれるコンポーネント
- `DaruiHandler`（Lambda 関数 1）
- `CountHandler`（Lambda 関数 2）
- `DaruiService`（ビジネスロジック）
- `LocationRepository`（DynamoDB アクセス）

### コード構造
```
backend/
  src/
    handlers/
      daruiHandler.ts       ← POST /darui エントリーポイント
      countHandler.ts       ← GET /darui/count エントリーポイント
    services/
      daruiService.ts       ← ビジネスロジック（recordAndCount / countNearby）
    repositories/
      locationRepository.ts ← DynamoDB 読み書き
    models/
      locationEvent.ts      ← DynamoDB アイテム型定義
      apiTypes.ts           ← リクエスト/レスポンス型定義
  package.json
  tsconfig.json
```

### テスト対象
- `DaruiService.recordAndCount()` のユニットテスト
- `LocationRepository.countNearby()` のバウンディングボックス計算テスト
- Lambda ハンドラーの統合テスト（ローカル実行）

---

## ユニット 2: `ios-app`

### 責務
- ホーム画面ウィジェットの UI レンダリング（近接カウント表示・キャラクター増殖）
- 「だるい」タップの AppIntent 処理
- バックエンド API との HTTP 通信
- GPS 位置情報の取得
- 匿名デバイス ID の管理
- 位置情報権限の初回リクエスト

### 含まれるコンポーネント
- `SlothLabApp`（メインアプリ Target）
- `SlothLabWidget`（Widget Extension Target）
- `DarUIIntent`（AppIntent）
- `SlothLabAPIClient`（ネットワーク）
- `LocationManager`（GPS）
- `DeviceIDManager`（デバイスID）

### コード構造
```
ios/
  SlothLab.xcodeproj
  SlothLab/                         ← Main App Target
    App/
      SlothLabApp.swift             ← @main エントリーポイント
    Views/
      ContentView.swift             ← メインアプリ画面（最小限）
    Shared/                         ← Main App と Widget Extension 両方に追加
      Network/
        SlothLabAPIClient.swift
        APIModels.swift             ← DaruiResponse / CountResponse
      Location/
        LocationManager.swift
      Identity/
        DeviceIDManager.swift
  SlothLabWidget/                   ← Widget Extension Target
    SlothLabWidget.swift            ← Widget / TimelineProvider / Entry View
    AppIntent.swift                 ← DarUIIntent
    SlothLabWidgetBundle.swift      ← @main WidgetBundle
  Assets.xcassets
  Info.plist
```

**注**: `Shared/` 配下のファイルは Main App Target と Widget Extension Target の両方に追加（AppGroup 不使用）

### テスト対象
- `DeviceIDManager` の ID 生成・永続化テスト
- `SlothLabAPIClient` のモックAPIを使ったユニットテスト
- iOS Simulator でのウィジェット表示確認
- AppIntent のタップ → API 送信 → reloadTimelines フロー確認

---

## ユニット 3: `infrastructure`

### 責務
- DynamoDB テーブル（`darui-events`）のプロビジョニング
- Lambda 関数 x2（DaruiHandler / CountHandler）のデプロイ
- API Gateway の設定（POST /darui, GET /darui/count）
- IAM ロールの設定（Lambda → DynamoDB 読み書き権限）

### 含まれるコンポーネント
- `SlothLabStack`（AWS CDK Stack）

### コード構造
```
infrastructure/
  bin/
    infrastructure.ts       ← CDK App エントリーポイント
  lib/
    sloth-lab-stack.ts      ← SlothLabStack 定義
  package.json
  tsconfig.json
  cdk.json
```

### テスト対象
- `cdk synth` でのテンプレート生成確認
- `cdk deploy` でのデプロイ確認
- API Gateway エンドポイントの疎通確認（curl）

---

## モノレポ全体構造

```
Sloth-Lab/                          ← リポジトリルート（現在の作業ディレクトリ）
  ios/                              ← ユニット 2: iOS アプリ
  backend/                          ← ユニット 1: バックエンド API
  infrastructure/                   ← ユニット 3: AWS CDK インフラ
  aidlc-docs/                       ← AI-DLC ドキュメント（アプリコードは含まない）
  README.md
```

---

## 開発順序（推奨）

| フェーズ | ユニット | 理由 |
|---------|---------|------|
| 1st | `infrastructure` | DynamoDB テーブルと API エンドポイントを先に確立 |
| 2nd | `backend-api` | インフラが揃ったら Lambda をデプロイ |
| 3rd | `ios-app` | API エンドポイント URL が確定したら iOS 開発 |

**並列開発**: iOS と Backend は API コントラクト（エンドポイント・リクエスト/レスポンス型）が合意できれば並列開発可能
