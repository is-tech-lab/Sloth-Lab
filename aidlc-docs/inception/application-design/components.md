# コンポーネント定義 — Sloth-Lab

## iOS コンポーネント

### 1. SlothLabApp（メインアプリ）

| 項目 | 内容 |
|------|------|
| **パッケージ** | iOS Main App Target |
| **責務** | アプリのエントリーポイント、位置情報権限管理、デバイスID初期化 |
| **インターフェース** | アプリライフサイクル管理、初回起動時の位置情報許可ダイアログ表示 |
| **依存コンポーネント** | LocationManager、DeviceIDManager |

### 2. SlothLabWidget（Widget Extension）

| 項目 | 内容 |
|------|------|
| **パッケージ** | Widget Extension Target |
| **責務** | WidgetKit タイムライン管理、ホーム画面 UI レンダリング、近接カウント表示、キャラクター増殖表示 |
| **インターフェース** | `TimelineProvider` プロトコル実装、`Widget` View 定義 |
| **依存コンポーネント** | SlothLabAPIClient、DeviceIDManager |

**注**: Widget は AppGroup を使用せず、TimelineProvider が直接 API を呼び出す（Q2: B 選択）

### 3. DarUIIntent（AppIntent）

| 項目 | 内容 |
|------|------|
| **パッケージ** | Widget Extension Target（または App Intents framework として共有）|
| **責務** | ウィジェットの「だるい」タップを処理、現在地取得、API 送信、ウィジェットタイムライン即時更新 |
| **インターフェース** | `AppIntent` プロトコル実装、`perform()` メソッド |
| **依存コンポーネント** | SlothLabAPIClient、LocationManager、DeviceIDManager |

### 4. SlothLabAPIClient（ネットワーク）

| 項目 | 内容 |
|------|------|
| **パッケージ** | Widget Extension / App Intents 双方から利用可能な共有モジュール |
| **責務** | バックエンド API との HTTP 通信を抽象化 |
| **インターフェース** | `postDarui(lat:lng:deviceId:)` / `getCount(lat:lng:)` |
| **依存コンポーネント** | URLSession（Foundation）|

### 5. LocationManager（位置情報）

| 項目 | 内容 |
|------|------|
| **パッケージ** | iOS Main App / Widget Extension 共有 |
| **責務** | CoreLocation を使用した現在地取得 |
| **インターフェース** | `requestCurrentLocation()` async |
| **依存コンポーネント** | CoreLocation framework |

### 6. DeviceIDManager（デバイスID）

| 項目 | 内容 |
|------|------|
| **パッケージ** | iOS Main App / Widget Extension 共有 |
| **責務** | 匿名デバイス ID の生成・永続化（UserDefaults）|
| **インターフェース** | `getOrCreateDeviceID()` → String |
| **依存コンポーネント** | Foundation（UUID、UserDefaults）|

---

## バックエンド コンポーネント（Node.js / TypeScript / AWS Lambda）

### 7. DaruiHandler（Lambda 関数）

| 項目 | 内容 |
|------|------|
| **責務** | `POST /darui` のリクエストを処理。位置情報イベントを保存し、近接カウントを返す |
| **インターフェース** | `handler(event, context)` → APIGatewayProxyResponse |
| **依存コンポーネント** | DaruiService |

### 8. CountHandler（Lambda 関数）

| 項目 | 内容 |
|------|------|
| **責務** | `GET /darui/count` のリクエストを処理。近接カウントのみ返す（Widget の定期更新用）|
| **インターフェース** | `handler(event, context)` → APIGatewayProxyResponse |
| **依存コンポーネント** | DaruiService |

### 9. DaruiService（ビジネスロジック）

| 項目 | 内容 |
|------|------|
| **責務** | 位置情報イベントの保存と近接カウントロジックのオーケストレーション |
| **インターフェース** | `recordAndCount(deviceId, lat, lng)` / `countNearby(lat, lng)` |
| **依存コンポーネント** | LocationRepository |

### 10. LocationRepository（データアクセス）

| 項目 | 内容 |
|------|------|
| **責務** | DynamoDB の読み書き操作を抽象化 |
| **インターフェース** | `saveEvent(...)` / `countNearby(lat, lng, radiusKm, withinMinutes)` |
| **依存コンポーネント** | DynamoDB（AWS SDK v3）|

---

## インフラ コンポーネント

### 11. SlothLabStack（AWS CDK）

| 項目 | 内容 |
|------|------|
| **責務** | AWS リソース（DynamoDB Table、Lambda 関数 x2、API Gateway）のプロビジョニング |
| **インターフェース** | CDK Stack クラス、`cdk deploy` コマンド |
| **依存コンポーネント** | AWS CDK（aws-cdk-lib）|
