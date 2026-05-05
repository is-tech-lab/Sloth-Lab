# サービス定義 — Sloth-Lab

## サービスの役割

**サービスレイヤー**は、複数のコンポーネントをオーケストレーションしてユーザーストーリーのユースケースを実現する。

---

## iOS サービス

### DarUITapService（DarUIIntent.perform() 内で実行）

**目的**: US-02「ウィジェットからの「だるい」送信」を実現するオーケストレーション

**トリガー**: ウィジェットの「だるい」ボタンタップ（AppIntent）

**実行フロー**:
```
1. LocationManager.requestCurrentLocation()
      ↓ CLLocation（緯度・経度）
2. DeviceIDManager.getOrCreateDeviceID()
      ↓ deviceId（UUID文字列）
3. SlothLabAPIClient.postDarui(lat:lng:deviceId:)
      ↓ DaruiResponse { count: Int }
4. WidgetCenter.shared.reloadTimelines(ofKind: "SlothLabWidget")
      ↓ Widget タイムラインが即時更新される
5. return .result()
```

**エラーハンドリング（PoC最小限）**:
- 位置情報取得失敗: 処理を中断（Widget に変化なし）
- API 失敗: 処理を中断（Widget に変化なし）

---

### WidgetTimelineService（SlothLabWidget.Provider.getTimeline() 内で実行）

**目的**: US-03「近接同志数の取得・表示」と US-05「ホーム画面での常時表示」を実現

**トリガー**: WidgetKit によるタイムライン更新（自動更新 or reloadTimelines 呼び出し後）

**実行フロー**:
```
1. DeviceIDManager.getOrCreateDeviceID()
      ↓ deviceId（位置情報取得のための識別子）
2. LocationManager.requestCurrentLocation()
      ↓ CLLocation（現在地）
3. SlothLabAPIClient.getCount(lat:lng:)
      ↓ CountResponse { count: Int }
4. DaruiEntry(date: now, count: count) を構築
5. Timeline([entry], policy: .after(nextRefreshDate)) を返す
```

**更新ポリシー**:
- 次回更新: 現在時刻 + 15分（WidgetKit のシステムポリシーに準拠）
- タップ直後は DarUITapService が reloadTimelines を呼ぶため即時更新される

---

## バックエンド サービス

### DaruiService

**目的**: 「だるい」イベントの記録と近接カウントのオーケストレーション

#### recordAndCount（POST /darui 用）

**トリガー**: `POST /darui { lat, lng, deviceId }`

**実行フロー**:
```
1. 入力バリデーション（lat/lng の数値チェック）
2. TTL 計算（現在時刻 + 30分の Unix timestamp）
3. LocationRepository.saveEvent(deviceId, lat, lng, ttl)
      ↓ DynamoDB に保存（既存レコードを上書き）
4. LocationRepository.countNearby(lat, lng, radiusKm=3, withinMinutes=30)
      ↓ バウンディングボックス方式で近接ユーザー数を取得
5. return count（自分自身は除外）
```

#### countNearby（GET /darui/count 用）

**トリガー**: `GET /darui/count?lat=&lng=`

**実行フロー**:
```
1. クエリパラメータから lat, lng を取得
2. LocationRepository.countNearby(lat, lng, radiusKm=3, withinMinutes=30)
3. return count
```

---

## サービス間の関係

```
【iOS タップフロー】
ユーザータップ
  → DarUITapService
      → LocationManager（現在地取得）
      → DeviceIDManager（デバイスID取得）
      → SlothLabAPIClient.postDarui()  →  [Backend] DaruiService.recordAndCount()
      → reloadTimelines()
          → WidgetTimelineService
              → SlothLabAPIClient.getCount()  →  [Backend] DaruiService.countNearby()

【Widget 自動更新フロー（15〜30分ごと）】
WidgetKit スケジューラ
  → WidgetTimelineService
      → LocationManager（現在地取得）
      → SlothLabAPIClient.getCount()  →  [Backend] DaruiService.countNearby()
```
