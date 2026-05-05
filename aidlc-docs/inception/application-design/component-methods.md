# コンポーネントメソッド — Sloth-Lab

> **注**: 詳細なビジネスロジックは各ユニットの機能設計 (Functional Design) で定義される。ここではメソッドシグネチャと高レベルの目的のみ記載。

---

## iOS コンポーネント

### SlothLabApp

```swift
// 位置情報権限のリクエスト（初回起動時）
func requestLocationPermission() -> Void

// 匿名デバイスIDの初期化（初回起動時）
func initializeDeviceID() -> Void
```

---

### DarUIIntent

```swift
// WidgetKit AppIntent の必須メソッド。「だるい」タップ時に実行される。
// 処理: 現在地取得 → API送信 → reloadTimelines
func perform() async throws -> some IntentResult
```

---

### SlothLabAPIClient

```swift
// POST /darui：「だるい」イベントを送信し、近接カウントを返す
// Input: 緯度、経度、匿名デバイスID
// Output: 近接「だるい」人数
func postDarui(lat: Double, lng: Double, deviceId: String) async throws -> DaruiResponse

// GET /darui/count：現在の近接カウントのみ取得（Widget の定期更新用）
// Input: 緯度、経度
// Output: 近接「だるい」人数
func getCount(lat: Double, lng: Double) async throws -> CountResponse
```

**データ型:**
```swift
struct DaruiResponse: Codable {
    let count: Int           // 近接「だるい」人数（自分を除く）
}

struct CountResponse: Codable {
    let count: Int
}
```

---

### LocationManager

```swift
// 現在のGPS位置情報を非同期で取得
// Output: CLLocation（緯度・経度・精度）
// Throws: 位置情報取得失敗時（権限なし等）
func requestCurrentLocation() async throws -> CLLocation
```

---

### DeviceIDManager

```swift
// 匿名デバイスIDを取得または新規生成してUserDefaultsに保存
// Output: UUID文字列（例: "550e8400-e29b-41d4-a716-446655440000"）
func getOrCreateDeviceID() -> String
```

---

### SlothLabWidget.Provider（TimelineProvider）

```swift
// スナップショット取得（ウィジェットギャラリー表示用）
func getSnapshot(in context: Context, completion: @escaping (DaruiEntry) -> Void)

// タイムライン取得（実際の表示データ）
// 処理: APIClient.getCount() を呼び出してカウントを取得し、Entryを構築
func getTimeline(in context: Context, completion: @escaping (Timeline<DaruiEntry>) -> Void)

// プレースホルダー（初期表示用）
func placeholder(in context: Context) -> DaruiEntry
```

**データ型:**
```swift
struct DaruiEntry: TimelineEntry {
    let date: Date
    let count: Int           // 近接「だるい」人数
}
```

---

## バックエンド コンポーネント（TypeScript）

### DaruiHandler

```typescript
// POST /darui のエントリーポイント
// Input body: { lat: number, lng: number, deviceId: string }
// Output: { count: number }
export const handler: APIGatewayProxyHandler = async (event, context) => Promise<APIGatewayProxyResult>
```

---

### CountHandler

```typescript
// GET /darui/count のエントリーポイント
// Query params: lat, lng
// Output: { count: number }
export const handler: APIGatewayProxyHandler = async (event, context) => Promise<APIGatewayProxyResult>
```

---

### DaruiService

```typescript
// 位置情報イベントを保存し、近接カウントを返す（POST /darui 用）
async recordAndCount(deviceId: string, lat: number, lng: number): Promise<number>

// 近接カウントのみ返す（GET /darui/count 用）
async countNearby(lat: number, lng: number): Promise<number>
```

---

### LocationRepository

```typescript
// 「だるい」イベントを DynamoDB に保存（TTL付き）
// ttl: Unix timestamp（現在時刻 + 30分）
async saveEvent(deviceId: string, lat: number, lng: number, ttl: number): Promise<void>

// バウンディングボックス方式で近接ユーザー数を取得
// radiusKm: 検索半径（デフォルト3km）
// withinMinutes: 対象時間範囲（デフォルト30分）
async countNearby(lat: number, lng: number, radiusKm: number, withinMinutes: number): Promise<number>
```
