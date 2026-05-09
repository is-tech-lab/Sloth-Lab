# バージョン管理レビュー — Sloth Feed PoC

> 確定した技術選択（Next.js / Auth.js / AWS SDK / React / Node.js / TypeScript）に対し、**2024〜2025 年の実インシデント事例**を踏まえてバージョン管理戦略を評価する。`security-review.md` の **A06 Vulnerable Components** を深掘りする補完成果物。
>
> **対象範囲**：採用パッケージのバージョン選定・ピン留め戦略・CI チェック・サプライチェーン耐性。
> **対象外**：個別 CVE の網羅・依存ツリー全体の SBOM 作成（実装時に行う）。

---

## 1. サマリー

| 領域 | 評価 | 主な根拠 |
|---|---|---|
| Next.js バージョン選定 | 🟡 注意 | **CVE-2025-29927（middleware 認可バイパス）**の影響範囲広く、middleware を認可で使う Sloth Feed は **patched 版必須** |
| Auth.js v5 | 🟡 注意 | 長期 beta から 2024-10 に GA。**マイナー間の破壊的変更履歴あり**、ピン留め推奨 |
| AWS SDK v3 | 🟢 良好 | リリース頻度高いが互換性安定。モジュラー設計で攻撃面が小さい |
| React | 🟢 良好 | 19 GA（2024-12）。Next.js 14 系なら 18.x で安定 |
| Node.js | 🟢 良好 | LTS（20.x / 22.x）を使う前提なら堅牢 |
| TypeScript | 🟢 良好 | マイナー間で型推論が変わるが、ランタイム影響なし |
| **サプライチェーン耐性** | 🟡 **要対策** | **shai-hulud worm / chalk-debug 事件 / xz-utils 事件**を踏まえ、**postinstall 制御 + IAM ロール採用** が重要 |
| ロックファイル運用 | 🟡 要決定 | `package-lock.json` の CI 強制（`npm ci`）/ 開発時 `npm install` 禁止の運用ルール化が必要 |

**総合**：採用スタック自体に致命的な問題はないが、**Next.js のバージョン下限と CI のサプライチェーンチェックを実装時に決める必要がある**。とくに Sloth Feed は middleware で認可しているため、**CVE-2025-29927 への対応が必須**。

---

## 2. 参照する 2024〜2025 年インシデント

### 2.1 CVE-2025-29927 — Next.js middleware 認可バイパス（2025-03）

| 項目 | 内容 |
|---|---|
| 脆弱性 | `x-middleware-subrequest` ヘッダを細工することで middleware を**スキップ**できる |
| 影響 | Next.js 11.1.4 〜 15.2.2 の幅広いバージョン |
| 修正版 | **15.2.3 / 14.2.25 / 13.5.9 / 12.3.5** |
| Sloth Feed への含意 | **middleware で認可している** ため直撃。修正版以上を必須要件にする |

**設計影響**：
- `middleware.ts` 単独で認可を担保する設計は**失敗の単一点**になりうる
- `security-review.md` の Defense in depth 方針通り、**API Route 内でも `await auth()` で再検証**する設計を継続

### 2.2 shai-hulud npm worm（2025-09）

| 項目 | 内容 |
|---|---|
| 攻撃 | 自己複製する npm パッケージのワーム化。`tinycolor` 系列を発端に**postinstall スクリプトで AWS / GCP / GitHub のクレデンシャルを盗み**、別パッケージに伝播 |
| 影響 | 数百パッケージが連鎖的に汚染 |
| Sloth Feed への含意 | **AWS クレデンシャル管理に直撃しうる**。ローカル開発で `AWS_ACCESS_KEY_ID` を `.env.local` に置く運用は worm に読み取られる経路となる |

**設計影響**：
- 本番では **IAM ロール / インスタンスプロファイル / OIDC ロール引き受け** を必須化
- ローカル開発でも `aws sso login` ベースの一時クレデンシャル + `~/.aws/credentials` を使い、`.env.local` に AWS キー直書きしない運用を推奨
- CI で **postinstall 制御**：`npm ci --ignore-scripts` を検討（ただし一部パッケージで動かない可能性あり、要検証）

### 2.3 chalk / debug npm 乗っ取り（2025-09）

| 項目 | 内容 |
|---|---|
| 攻撃 | メンテナのフィッシングで npm publish トークンを奪取し、**`chalk` `debug` `ansi-styles` 等の超人気パッケージに暗号通貨ウォレット盗難コードを注入** |
| 影響 | 週次数億ダウンロード級のパッケージが汚染 |
| Sloth Feed への含意 | **`chalk` `debug` は Next.js / Auth.js / AWS SDK のトランジティブ依存**にほぼ確実に入る |

**設計影響**：
- ロックファイル（`package-lock.json`）を**必ずコミット**し、`npm ci` で同一バージョンのみインストール
- **Dependabot / Renovate** によるバージョン上げは PR レビュー必須化
- `socket.dev` 等のサプライチェーン分析ツール導入を検討（Phase 2）

### 2.4 xz-utils バックドア — CVE-2024-3094（2024-03）

| 項目 | 内容 |
|---|---|
| 攻撃 | 約 2 年に渡る社会工学で xz-utils メンテナ権限を奪取、**バックドアコードを上流に混入** |
| 影響 | sshd まで遡って攻撃される可能性。Linux ディストリで広範に対応 |
| Sloth Feed への含意 | npm エコシステムでも同様の長期社会工学攻撃が起こりうる |

**設計影響**：
- メンテナ数が極端に少ない（1 名）OSS 依存を避ける（避けようがないものは認知のみ）
- CI ベースイメージは **AWS 公式 / Node.js 公式** などの監査されたものを使用

### 2.5 Polyfill.io サプライチェーン（2024-06）

| 項目 | 内容 |
|---|---|
| 攻撃 | `cdn.polyfill.io` のドメインを買収した第三者が、**配信スクリプトに悪性コードを注入** |
| 影響 | 100 万サイト以上に影響 |
| Sloth Feed への含意 | **CDN 経由の外部スクリプトを script タグで読み込まない**設計が安全（Sloth Feed は Next.js でバンドルする方針なので構造的に該当しない）|

---

## 3. パッケージ別評価

### 3.1 Next.js

| 項目 | 推奨 |
|---|---|
| バージョン下限 | **14.2.25 以上 または 15.2.3 以上**（CVE-2025-29927 対応） |
| 推奨採用 | **14.x 系の最新 LTS 相当**（App Router 安定 + React 18 安定）<br>または **15.x 最新**（React 19 採用するなら）|
| ピン留め | `package.json` で `"next": "14.2.25"` のように exact pin を推奨（minor の自動上げを避ける）|
| アップグレード方針 | セキュリティパッチ（patch / minor）は速やかに / メジャーアップは Phase 2 |
| 監視 | https://github.com/vercel/next.js/security/advisories を購読 |

**注意点**：
- Next.js 15 は React 19 を要求。**React 19 の他ライブラリ互換が完全でない可能性**を Phase 2 で検証
- App Router の API は Next.js 13→14→15 で**段階的に変化**。バージョン跨ぎでは動作確認必須

### 3.2 Auth.js (next-auth v5)

| 項目 | 推奨 |
|---|---|
| 採用バージョン | **`next-auth@5.x` GA 版以上**（2024-10 に v5.0.0 GA） |
| ピン留め | `"next-auth": "5.x.x"` exact pin（v5 は GA 後も活発）|
| Cognito Provider | `next-auth/providers/cognito` を利用（公式組み込み）|
| 注意点 | beta 期間中に App Router 対応・Edge ランタイム対応が大きく変わった経緯あり。**v5 GA 以降のドキュメント・サンプルのみ参照**する |

**設計影響**：
- v5 は v4 とインポートパス・設定形式が大幅に異なる。**インターネット上のサンプルコードがどちらか確認する習慣**を実装ルールに含める
- Edge Runtime での `await auth()` 対応は v5 で改善されている

### 3.3 AWS SDK v3

| パッケージ | 用途 | 推奨 |
|---|---|---|
| `@aws-sdk/client-bedrock-runtime` | Bedrock 呼び出し | exact pin・週次〜月次でパッチ確認 |
| `@aws-sdk/client-dynamodb` | DynamoDB 低レベル | 同上 |
| `@aws-sdk/lib-dynamodb` | DocumentClient | 同上 |
| `@aws-sdk/credential-providers` | クレデンシャル解決 | 同上 |

**注意点**：
- AWS SDK v2 は **2025-09 に End of Support**。Sloth Feed は v3 採用なので影響なし（明示確認）
- v3 は **モジュラー設計**で必要な client のみインストールする想定。**`aws-sdk` 単体パッケージは絶対に install しない**（v2 の bundle 物）
- リリース頻度が高い（しばしば日次）。すべて追わず、**月次でまとめて patch を上げる運用**が現実的

### 3.4 React

| 項目 | 推奨 |
|---|---|
| 採用バージョン | Next.js 14.x なら **React 18.3.x**（最新パッチ）<br>Next.js 15.x なら **React 19.x** |
| ピン留め | Next.js が peer 指定で実質ピン留めされるが、`package.json` でも明示 |
| 注意点 | React 19 は Server Components の挙動・useFormStatus 等で変化あり。Auth.js + Next.js 15 + React 19 のスリーピース動作確認は必須 |

### 3.5 Node.js ランタイム

| 項目 | 推奨 |
|---|---|
| バージョン | **20.x LTS** または **22.x LTS**（2026-05 時点で両方アクティブ） |
| 24.x | LTS 化が 2025 年秋予定。LTS 確定までは採用しない |
| 18.x | **2025-04 で End of Life**。採用禁止 |
| ピン留め | `package.json` の `"engines": { "node": ">=20.0.0 <23.0.0" }` |
| Docker / Lambda | `node:22-alpine` または AWS Lambda の `nodejs22.x` ランタイム |

### 3.6 TypeScript

| 項目 | 推奨 |
|---|---|
| 採用バージョン | **5.x 系最新**（2026-05 時点で 5.7+ 想定） |
| ピン留め | minor 単位で固定（型推論差を避ける）|
| 注意点 | strict モード必須。`tsconfig.json` の `strict: true` をベースラインに |

---

## 4. ピン留め戦略

| 戦略 | Sloth Feed 採用 |
|---|---|
| `^x.y.z`（caret）| ❌ minor の自動上げで CVE 受領が遅れる / supply chain 攻撃に弱い |
| `~x.y.z`（tilde）| 🟡 patch のみ自動。妥協案として使えるが next-auth など activeなパッケージでは不十分 |
| `x.y.z`（exact）| ✅ **採用**：security-critical / 認証 / AWS 系は exact pin |
| ロックファイル `package-lock.json` | ✅ **必ずコミット**し、CI は `npm ci` で同一バージョン保証 |
| バージョン上げ運用 | **Renovate / Dependabot で PR 化 → コードレビュー → マージ**。手動アップデート禁止 |

**実装時の `package.json` 例（イメージ）**：

```json
{
  "engines": { "node": ">=20.0.0 <23.0.0" },
  "dependencies": {
    "next": "14.2.25",
    "react": "18.3.1",
    "react-dom": "18.3.1",
    "next-auth": "5.0.0",
    "@aws-sdk/client-bedrock-runtime": "3.700.0",
    "@aws-sdk/client-dynamodb": "3.700.0",
    "@aws-sdk/lib-dynamodb": "3.700.0"
  },
  "devDependencies": {
    "typescript": "5.7.0"
  }
}
```

（具体バージョン番号は実装着手時の最新を採用）

---

## 5. CI / CD のセキュリティチェック

| チェック | ツール | PoC | Phase 2 |
|---|---|---|---|
| ロックファイル整合性 | `npm ci` | ✅ 必須 | ✅ |
| 既知 CVE スキャン | `npm audit --audit-level=high` | ✅ High 以上 fail | ✅ Critical/High すべて fail |
| 依存上げ自動化 | Dependabot / Renovate | 🟡 PoC は手動でも可 | ✅ 自動 PR |
| ライセンス互換性 | `license-checker` | 🟡 任意 | ✅ |
| サプライチェーン分析 | `socket.dev` / Snyk | ❌ Phase 2 | ✅ |
| SBOM 生成 | `cyclonedx-npm` | ❌ Phase 2 | ✅ |
| 署名検証 | npm provenance | 🟡 利用可なら On | ✅ |
| postinstall 制御 | `npm ci --ignore-scripts`（要検証）| 🟡 ベース動作確認後 On | ✅ |
| GitHub Push Protection | GitHub | ✅ | ✅ |
| Secret Scanning | GitHub | ✅ | ✅ |

**重要な運用**：
- `package-lock.json` の差分は **PR レビュー時に必ず確認**（怪しい依存の混入検知）
- `npm ci` 失敗時はバージョン整合性が崩れているサイン → 強制 `npm install` で誤魔化さない

---

## 6. クレデンシャル管理（shai-hulud 事件を踏まえて）

| 環境 | 推奨 |
|---|---|
| ローカル開発 | **`aws sso login` + `~/.aws/credentials` 一時クレデンシャル**。`.env.local` への AWS キー直書きは避ける。やむを得ず使う場合は **Cognito / Bedrock / DynamoDB の最小権限ポリシー**のみ |
| CI（GitHub Actions）| **OIDC で AWS IAM ロール引き受け**（長期キー禁止）|
| 本番 | **タスクロール / インスタンスプロファイル / Lambda 実行ロール**。長期キー絶対禁止 |
| `.env.local` | `.gitignore` に登録 + GitHub Push Protection で **secret commit を block** |
| キーローテーション | 万一漏洩した場合は即時無効化。**監査ログで利用追跡可能な状態を維持** |

---

## 7. PoC で守るべきベースライン

実装時に必ず実施する項目：

- [ ] `package.json` の `engines` で Node.js 20+ / <23 を強制
- [ ] `next` を 14.2.25+ または 15.2.3+ に**exact pin**（CVE-2025-29927 対応）
- [ ] `next-auth` を 5.x exact pin
- [ ] `@aws-sdk/*` パッケージを exact pin（モジュラー設計を活かして必要な client のみ）
- [ ] `package-lock.json` を必ずコミット
- [ ] `.gitignore` に `.env.local` / `.env*.local` を含む
- [ ] CI で `npm ci`（`npm install` 禁止）
- [ ] CI で `npm audit --audit-level=high` を実行し High 以上で fail
- [ ] GitHub の Dependabot を有効化（少なくとも `security` のみでも On）
- [ ] GitHub の Push Protection / Secret Scanning を有効化
- [ ] AWS クレデンシャルはローカルでも一時クレデンシャル / 本番は IAM ロール
- [ ] `.env.local.example` で必須環境変数を明示（実値を含めない）
- [ ] middleware の `matcher` 漏れに加え、API Route 内 `await auth()` の二重ガードを Code Review チェックリストに

---

## 8. Phase 2 で導入検討

| # | 項目 | 動機 |
|---|---|---|
| 1 | Renovate / Dependabot 自動 PR + auto-merge（patch のみ）| 月次手動運用の省力化 |
| 2 | `socket.dev` / `Snyk` サプライチェーン分析 | shai-hulud 等の異常 publish 検知 |
| 3 | SBOM 生成（CycloneDX）| 監査・脆弱性追跡 |
| 4 | npm provenance 検証 | publish 元の検証 |
| 5 | `--ignore-scripts` を CI / 本番ビルドで強制 | postinstall 経由の攻撃面削減 |
| 6 | プライベート npm レジストリ / 署名された vendored ミラー | 上流 npm の単一点障害回避 |
| 7 | コンテナイメージスキャン（Trivy / ECR Scan）| 本番デプロイ前の最終確認 |
| 8 | Renovate で **古い依存の age 制限**（例：published from 7 days 以下を block）| 直近 publish の悪意ある版を一時隔離 |

---

## 9. リスク評価マトリクス（バージョン管理視点）

| ID | リスク | 発生可能性 | 影響 | 対応 |
|---|---|---|---|---|
| V-1 | CVE-2025-29927 未対応 Next.js を採用 | 中（情報を知らないと起こる）| 高（middleware 認可バイパス）| §3.1 のバージョン下限を実装時必須要件に |
| V-2 | npm ワーム / 乗っ取りで AWS キー漏洩 | 低〜中 | 高 | §6 のクレデンシャル管理 + `.env.local` 禁止運用 |
| V-3 | caret pin で意図しない minor 上げ | 高（油断する）| 中 | §4 の exact pin 戦略 |
| V-4 | `package-lock.json` のコンフリクトを `--force` で解決 | 中 | 中 | PR レビューで lock 差分を必ず確認 |
| V-5 | Auth.js v4 のサンプルを v5 環境にコピペ | 中 | 中 | v5 公式ドキュメントのみ参照のルール化 |
| V-6 | AWS SDK v2 を誤って install | 低 | 中 | `package.json` レビューで `aws-sdk` 単体名を block |
| V-7 | Node.js 18 を本番採用（EOL）| 低 | 中 | `engines` で >= 20 を強制 |
| V-8 | postinstall で任意コード実行されるパッケージ追加 | 低〜中 | 高 | 新規依存追加時に Code Review |

---

## 10. 結論

**3 回目サイクルで採用した技術スタックそのものに、バージョン管理上の致命的問題はない**。ただし、以下 3 点は**実装着手前に決定すべき**：

1. **Next.js のバージョン下限（CVE-2025-29927 対応）**を 14.2.25+ / 15.2.3+ に確定
2. **AWS クレデンシャル運用**を IAM ロール一本化（shai-hulud 事件への構造的耐性）
3. **CI で `npm ci` + `npm audit` を必須化**（chalk-debug 事件への運用面耐性）

この 3 点を `package.json` / CI 設定 / 開発者ガイドに明文化することで、PoC レベルとして十分なバージョン管理体制を構築できる。残りは Phase 2 で **Dependabot 自動化 + サプライチェーン分析ツール導入** で段階的に強化する。

---

## 11. 関連ドキュメント

- [security-review.md](security-review.md) — A06 Vulnerable Components（本ドキュメントの起点）
- [application-design.md](application-design.md) — 採用技術スタックの全体像
- [components.md](components.md) — `@aws-sdk/*` 利用箇所
- [unit-of-work-dependency.md](unit-of-work-dependency.md) — 外部依存リスト
