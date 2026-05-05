# アプリケーション設計 — Sloth-Lab

**設計方針（確定した回答より）**:
- バックエンド: AWS Lambda + API Gateway（サーバーレス）
- Widget データフロー: TimelineProvider が直接 API を呼び出す（AppGroup 不要）
- 近接クエリ: バウンディングボックス方式（PoC向き、シンプル）

---

## 1. コンポーネント一覧

### iOS（Swift）

| コンポーネント | 責務 | 詳細 |
|--------------|------|------|
| **SlothLabApp** | エントリーポイント、位置情報権限管理 | [components.md](components.md) |
| **SlothLabWidget** | WidgetKit UI・タイムライン管理 | [components.md](components.md) |
| **DarUIIntent** | 「だるい」タップ → API送信 → reloadTimelines | [components.md](components.md) |
| **SlothLabAPIClient** | バックエンド HTTP 通信の抽象化 | [components.md](components.md) |
| **LocationManager** | CoreLocation による GPS 取得 | [components.md](components.md) |
| **DeviceIDManager** | 匿名デバイス ID の生成・永続化 | [components.md](components.md) |

### バックエンド（Node.js / TypeScript）

| コンポーネント | 責務 | 詳細 |
|--------------|------|------|
| **DaruiHandler** | POST /darui Lambda 関数 | [components.md](components.md) |
| **CountHandler** | GET /darui/count Lambda 関数 | [components.md](components.md) |
| **DaruiService** | ビジネスロジックのオーケストレーション | [components.md](components.md) |
| **LocationRepository** | DynamoDB 読み書き抽象化 | [components.md](components.md) |

### インフラ（AWS CDK）

| コンポーネント | 責務 | 詳細 |
|--------------|------|------|
| **SlothLabStack** | DynamoDB / Lambda / API Gateway のプロビジョニング | [components.md](components.md) |

---

## 2. 主要メソッドシグネチャ

→ 詳細は [component-methods.md](component-methods.md) を参照

**iOS 主要メソッド:**
```swift
// タップ処理（AppIntent）
DarUIIntent.perform() async throws -> IntentResult

// API クライアント
SlothLabAPIClient.postDarui(lat: Double, lng: Double, deviceId: String) async throws -> DaruiResponse
SlothLabAPIClient.getCount(lat: Double, lng: Double) async throws -> CountResponse

// Widget タイムライン
SlothLabWidget.Provider.getTimeline(in:completion:)
```

**バックエンド主要メソッド:**
```typescript
DaruiService.recordAndCount(deviceId, lat, lng): Promise<number>
DaruiService.countNearby(lat, lng): Promise<number>
LocationRepository.saveEvent(deviceId, lat, lng, ttl): Promise<void>
LocationRepository.countNearby(lat, lng, radiusKm, withinMinutes): Promise<number>
```

---

## 3. サービスオーケストレーション

→ 詳細は [services.md](services.md) を参照

| サービス | トリガー | 目的 |
|---------|---------|------|
| **DarUITapService** | ウィジェットタップ | 現在地取得 → API送信 → Widget更新 |
| **WidgetTimelineService** | WidgetKit スケジューラ | 定期的な近接カウント取得・表示更新 |
| **DaruiService（Backend）** | POST /darui, GET /darui/count | イベント保存・近接カウント返却 |

---

## 4. コンポーネント依存関係

→ 詳細・図は [component-dependency.md](component-dependency.md) を参照

```
iOS
  DarUIIntent ─→ APIClient ─→ POST /darui ─→ DaruiHandler ─→ DaruiService ─→ Repo ─→ DynamoDB
  Widget      ─→ APIClient ─→ GET /darui/count ─→ CountHandler ─→ DaruiService ─→ Repo ─→ DynamoDB
  App         ─→ LocationManager, DeviceIDManager

Infrastructure
  CDK Stack ─→ DynamoDB Table / Lambda x2 / API Gateway
```

---

## 5. API エンドポイント定義

### POST /darui

| 項目 | 内容 |
|------|------|
| **目的** | 「だるい」イベント送信・近接カウント取得 |
| **Request Body** | `{ "lat": number, "lng": number, "deviceId": string }` |
| **Response** | `{ "count": number }` |
| **処理** | イベント保存（TTL=30分） + バウンディングボックスカウント |

### GET /darui/count

| 項目 | 内容 |
|------|------|
| **目的** | Widget 自動更新用の近接カウント取得 |
| **Query Params** | `lat=number&lng=number` |
| **Response** | `{ "count": number }` |
| **処理** | バウンディングボックスカウントのみ |

---

## 6. DynamoDB テーブル設計（概要）

| 属性 | 型 | 役割 |
|-----|-----|------|
| `deviceId` | String (PK) | ユーザー識別子（重複防止） |
| `timestamp` | String (SK) | タップ時刻（ISO 8601）|
| `lat` | Number | 緯度 |
| `lng` | Number | 経度 |
| `ttl` | Number | TTL（Unix timestamp、30分後）|

**バウンディングボックスクエリ**: Lambda 内で `lat ± 0.027°` / `lng ± 0.027°`（≒3km）を計算し、Scan + FilterExpression で近接ユーザーを取得。

---

## 7. ユーザーストーリーとコンポーネントのマッピング

| ストーリー | 関連コンポーネント |
|-----------|-----------------|
| US-01 位置情報許可 | SlothLabApp → LocationManager |
| US-02 ウィジェットからタップ | DarUIIntent → SlothLabAPIClient → DaruiHandler |
| US-03 近接同志数取得 | DaruiService → LocationRepository → DynamoDB |
| US-04 キャラクター増殖 | SlothLabWidget（UI ロジック、count に応じた表示）|
| US-05 ホーム画面常時表示 | SlothLabWidget（WidgetKit）|
| US-06 自動更新 | SlothLabWidget.Provider → SlothLabAPIClient → CountHandler |
