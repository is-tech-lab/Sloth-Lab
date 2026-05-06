# Context Mapping — Refactor the World (RTW) MVP

---

## 図1: バウンデッドコンテキストマップ

各コンテキスト間の関係とDDDパターンを示す。

```mermaid
flowchart TD
    classDef external fill:#111111,stroke:#333333,color:#666666
    classDef u1 fill:#0D1B2A,stroke:#00BFFF,color:#00BFFF
    classDef u2 fill:#1A0D00,stroke:#FF9F43,color:#FF9F43
    classDef u3 fill:#0E001A,stroke:#A29BFE,color:#A29BFE
    classDef u4 fill:#001A10,stroke:#55EFC4,color:#55EFC4
    classDef pattern fill:#1A1A1A,stroke:#333,color:#555,font-size:11px

    OpenAI["OpenAI API<br/>GPT-4V + DALL-E 3"]:::external
    S3["AWS S3 + CloudFront"]:::external

    IC["🔵 Identity Context<br/>─────────────────────<br/>Unit 1 : 認証<br/>OHS + Published Language<br/>JWT 発行・検証"]:::u1

    TC["🟠 Transform Context<br/>─────────────────────<br/>Unit 2 : Capture & Refactor<br/>AI 変換パイプライン<br/>ACL で外部依存を隔離"]:::u2

    FC["🟣 Feed Context<br/>─────────────────────<br/>Unit 3 : Social Feed<br/>投稿 / フィード / いいね"]:::u3

    PC["🟢 Profile Context<br/>─────────────────────<br/>Unit 4 : My Page<br/>読取専用クエリ"]:::u4

    OpenAI -->|"Upstream ↓ ACL"| TC
    S3 -->|"Upstream ↓ ACL"| TC

    IC -->|"Upstream ↓ Conformist<br/>JWT 検証ミドルウェア"| TC
    IC -->|"Upstream ↓ Conformist<br/>JWT 検証ミドルウェア"| FC
    IC -->|"Upstream ↓ Conformist<br/>JWT 検証ミドルウェア"| PC

    TC -->|"imageUrl : string<br/>疎結合データ渡し<br/>コード依存なし"| FC

    FC -->|"Upstream ↓ Customer/Supplier<br/>IPostRepository.findByUserId<br/>ILikeRepository.findByUserId"| PC
```

| パターン | 意味 |
|---------|------|
| **OHS + PL** | Identity が JWT という公開言語で全コンテキストに認証サービスを提供 |
| **Conformist** | Transform / Feed / Profile はJWTフォーマットをそのまま受け入れる |
| **ACL** | Transform が OpenAI / S3 のAPI変更からドメインを保護する |
| **Customer/Supplier** | Profile (顧客) が Feed (供給者) に追加クエリメソッドをリクエスト |
| **疎結合データ渡し** | Transform → Feed は imageUrl 文字列を渡すだけ。コードimportなし |

---

## 図2: ユーザーストーリー × コンテキストフロー

各USがどのコンテキストに属し、どの順序・依存で繋がるかを示す。

```mermaid
flowchart LR
    classDef u1 fill:#0D1B2A,stroke:#00BFFF,color:#00BFFF
    classDef u2 fill:#1A0D00,stroke:#FF9F43,color:#FF9F43
    classDef u3 fill:#0E001A,stroke:#A29BFE,color:#A29BFE
    classDef u4 fill:#001A10,stroke:#55EFC4,color:#55EFC4

    subgraph AUTH["🔵 Unit 1 : Identity Context　認証"]
        direction TB
        US01["US-01 登録"]:::u1
        US02["US-02 ログイン"]:::u1
        US03["US-03 ログアウト"]:::u1
    end

    subgraph CAP["🟠 Unit 2 : Transform Context　Capture & Refactor"]
        direction TB
        US04["US-04 カメラ撮影"]:::u2
        US05["US-05 ロール選択"]:::u2
        US06["US-06 AI 変換"]:::u2
        US07["US-07 変換確認・再変換"]:::u2
        US04 --> US06
        US05 --> US06
        US06 <--> US07
    end

    subgraph SOC["🟣 Unit 3 : Feed Context　Social Feed"]
        direction TB
        US08["US-08 投稿"]:::u3
        US09["US-09 フィード閲覧"]:::u3
        US10["US-10 いいね"]:::u3
        US11["US-11 いいね取消"]:::u3
        US08 --> US09
        US09 --> US10
        US09 --> US11
        US10 <--> US11
    end

    subgraph PRO["🟢 Unit 4 : Profile Context　My Page"]
        direction TB
        US12["US-12 自分の投稿一覧"]:::u4
        US13["US-13 いいね一覧"]:::u4
    end

    AUTH -->|"認証ゲート\nJWT required"| CAP
    AUTH -->|"認証ゲート\nJWT required"| SOC
    AUTH -->|"認証ゲート\nJWT required"| PRO

    US06 -->|"imageUrl\n文字列渡し"| US08

    US08 -.->|"投稿履歴"| US12
    US10 -.->|"いいね履歴"| US13
```

**凡例**:
- `→` = 実線: 実行時の直接依存（前のUSが完了しないと成立しない）
- `-.->` = 破線: データ参照（同一DBテーブルを読む。コード依存なし）
- `<->` = 双方向: 同一画面・同一フローで往復する操作

---

## 図3: ユニット間インターフェース契約

各ユニットが他ユニットに公開するインターフェースを示す。

```mermaid
flowchart LR
    classDef contract fill:#1A1A1A,stroke:#333,color:#888,font-style:italic
    classDef u1 fill:#0D1B2A,stroke:#00BFFF,color:#00BFFF
    classDef u2 fill:#1A0D00,stroke:#FF9F43,color:#FF9F43
    classDef u3 fill:#0E001A,stroke:#A29BFE,color:#A29BFE
    classDef u4 fill:#001A10,stroke:#55EFC4,color:#55EFC4

    U1["Unit 1\nIdentity"]:::u1
    U2["Unit 2\nTransform"]:::u2
    U3["Unit 3\nFeed"]:::u3
    U4["Unit 4\nProfile"]:::u4

    JWT["Bearer JWT\n RFC 7519"]:::contract
    presignedUrl["GET /upload/presigned-url\nGET /transform"]:::contract
    imgUrl["imageUrl : string\nCloudFront CDN URL"]:::contract
    postApi["POST /posts\nGET /feed\nPOST|DELETE /likes/:id"]:::contract
    repoExt["IPostRepository\n.findByUserId()\nILikeRepository\n.findByUserId()"]:::contract
    userApi["GET /users/me/posts\nGET /users/me/likes"]:::contract

    U1 -- "発行" --> JWT
    JWT -- "全ユニットが検証" --> U2
    JWT -- "全ユニットが検証" --> U3
    JWT -- "全ユニットが検証" --> U4

    U2 -- "公開" --> presignedUrl
    U2 -- "出力" --> imgUrl
    imgUrl -- "入力パラメータ" --> U3

    U3 -- "公開" --> postApi
    U3 -- "CS 契約" --> repoExt
    repoExt -- "拡張利用" --> U4
    U4 -- "公開" --> userApi
```

---

## 禁止依存（境界ルール）

```mermaid
flowchart LR
    classDef bad fill:#2A0000,stroke:#FF4D6A,color:#FF4D6A
    classDef ok fill:#001A0A,stroke:#00FF88,color:#00FF88

    F1["❌ Feed Context がTransform Context を import"]:::bad
    F2["❌ Transform Context がPost / Like 集約を参照"]:::bad
    F3["❌ Profile Context がFeed Usecase を直接呼び出し"]:::bad
    F4["❌ Domain Layer がPrisma/AWS SDK に直接依存"]:::bad

    OK1["✅ imageUrl は string で渡す"]:::ok
    OK2["✅ Repository Interface 経由のみ"]:::ok
    OK3["✅ ACL (Repository) 経由のみ"]:::ok
    OK4["✅ Interface 経由のみ (DI)"]:::ok

    F1 --- OK1
    F2 --- OK2
    F3 --- OK2
    F4 --- OK4
```
