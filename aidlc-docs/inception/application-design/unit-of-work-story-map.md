# ユニット・ストーリーマッピング — Sloth-Lab

## ストーリー × ユニット マッピング

| ストーリー | `ios-app` | `backend-api` | `infrastructure` |
|-----------|-----------|---------------|-----------------|
| US-01 位置情報許可 | ✅ 主担当 | — | — |
| US-02 ウィジェットからの「だるい」送信 | ✅ 主担当（AppIntent / UI）| ✅ 支援（POST /darui）| ✅ 支援（API GW）|
| US-03 近接同志数の取得・表示 | ✅ 支援（表示）| ✅ 主担当（クエリロジック）| ✅ 支援（DynamoDB）|
| US-04 キャラクター増殖表示 | ✅ 主担当（UI ロジック）| — | — |
| US-05 ホーム画面での常時表示 | ✅ 主担当（WidgetKit）| — | — |
| US-06 ウィジェットの自動更新 | ✅ 主担当（TimelineProvider）| ✅ 支援（GET /darui/count）| — |

**凡例**: ✅ 主担当 = そのユニットが主に実装 / ✅ 支援 = そのユニットが一部を担当

---

## ユニット別ストーリー詳細

### `ios-app` が担当するストーリー

| ストーリー | 実装コンポーネント | 受け入れ基準との対応 |
|-----------|-----------------|---------------------|
| US-01 | `SlothLabApp` → `LocationManager` | 初回起動時の Apple 標準ダイアログ表示 |
| US-02 | `DarUIIntent` → `SlothLabAPIClient` | ウィジェットからアプリを開かずにタップ送信 |
| US-03 | `SlothLabWidget` → `SlothLabAPIClient` | 近接人数の取得・表示（「3人が一緒にだるい」）|
| US-04 | `SlothLabWidget`（View ロジック）| count に応じたキャラクター数の UI 切り替え |
| US-05 | `SlothLabWidget`（WidgetKit）| ホーム画面への追加・常時表示 |
| US-06 | `SlothLabWidget.Provider`（TimelineProvider）| reloadTimelines + 自動更新タイムライン |

### `backend-api` が担当するストーリー

| ストーリー | 実装コンポーネント | 受け入れ基準との対応 |
|-----------|-----------------|---------------------|
| US-02 | `DaruiHandler` → `DaruiService` | 位置情報 + タイムスタンプの保存 |
| US-03 | `DaruiService` → `LocationRepository` | 半径3km・直近30分以内のユーザー数返却 |
| US-06 | `CountHandler` → `DaruiService` | 定期更新用カウント取得エンドポイント |

### `infrastructure` が担当するストーリー

| ストーリー | 実装コンポーネント | 受け入れ基準との対応 |
|-----------|-----------------|---------------------|
| US-02 | `SlothLabStack`（API Gateway）| バックエンド API のエンドポイント提供 |
| US-03 | `SlothLabStack`（DynamoDB）| TTL 付き位置情報データの保存基盤 |

---

## ハッカソンデモ「ゴールデンパス」とユニットの対応

```
デモステップ 1: ホーム画面に Sloth-Lab ウィジェットが表示されている
  → ios-app: SlothLabWidget（WidgetKit）

デモステップ 2: ウィジェットの「だるい」をタップ → 別デバイスもタップ
  → ios-app: DarUIIntent（AppIntent）
  → backend-api: DaruiHandler → DaruiService → LocationRepository
  → infrastructure: DynamoDB Table, API Gateway

デモステップ 3: ウィジェットの人数が増える → キャラクターが増殖する
  → ios-app: SlothLabWidget（UI更新, reloadTimelines後）
  → backend-api: CountHandler → DaruiService → LocationRepository

デモステップ 4: 「これが怠惰の連帯です」
  → 全ユニット連携の結果
```

---

## すべてのストーリーがユニットに割り当てられていることの確認

| ストーリー | 割り当て済み |
|-----------|------------|
| US-01 | ✅ ios-app |
| US-02 | ✅ ios-app + backend-api + infrastructure |
| US-03 | ✅ ios-app + backend-api + infrastructure |
| US-04 | ✅ ios-app |
| US-05 | ✅ ios-app |
| US-06 | ✅ ios-app + backend-api |

**結果**: 全6ストーリーが1つ以上のユニットに割り当て済み ✅
