# LM Navigator — プロジェクトコンテキスト

## プロジェクト概要

LM Navigatorは、ListeningMind（検索インテント分析ツール）の最適な使い方をAIがガイドするWebアプリケーション。ユーザーが「調査目的」を入力するだけで、Gemini APIが最適なListeningMind機能とキーワード候補を提案し、ワンクリックでListeningMindの調査画面に遷移できる。

**所有者:** K&F Consulting LLC.
**用途:** マーケティングコンサルティング業務でのクライアント向け・社内向けツール

## 技術構成

```
ブラウザ（単一HTML）
  ↓ POST
Cloudflare Workers（APIプロキシ + レート制限）
  lm-navigator-api.yamagishimktgoffice.workers.dev
  ↓
Gemini API
```

- **フロントエンド:** 単一HTMLファイル（HTML/CSS/JS完結、フレームワーク不使用）
- **ホスティング:** GitHub Pages
- **APIプロキシ:** Cloudflare Workers（1日20回のレート制限付き）
- **AI:** Gemini API（システムプロンプトでJSON形式の出力を強制）
- **フォント:** Syne（英字見出し用） + Noto Sans JP（本文用）

## UI設計方針

### 現行デザイン（v2）
- **テーマ:** ライトテーマ（白ベース + グレー背景）。Google/Facebook Ads風のダッシュボードUI
- **カラー:** アクセントは深い紫 `#4f5bd5`、成功は `#16a46c`、警告は `#e0930b`
- **レイアウト:** サイドバー（非表示状態で保持） + メインエリア（中央寄せ max-width: 880px）
- **トップバー:** sticky固定、高さ52px。タイトルのみ表示
- **フッター:** `© K&F Consulting LLC. All Rights Reserved.`

### サイドバー（将来のツール拡張用）
- HTMLは残してあるが `display: none` で非表示
- `.sidebar` に `.open` クラスを追加すると表示される
- `.main-wrapper` にも `.sidebar-open` クラスを追加して `margin-left: 220px` を適用する
- 将来追加予定のツール: Persona KW Research、ニーズ整理表、SEO記事ドラフト

### フロー表示
- 4ステップのプログレスバー: カテゴリー選択 → 目的を入力 → AIが提案 → 調査実行
- ステップ進行に応じて `updateFlow(step)` で動的に色・アイコンが変わる
- 結果表示時は「推奨機能」ラベル（`#resultTop`）にスクロール。`scroll-margin-top: 64px` でstickyヘッダー分をオフセット

### レスポンシブ
- 900px以下: サイドバー幅縮小、カテゴリーグリッド2列化
- 700px以下: カテゴリーグリッド2列、補足フィールド1列化

## データ構造

### CATEGORIES（8件）
分析カテゴリーの定義。各カテゴリーには `id`, `icon`, `name`, `desc`, `aux`（補足入力フィールド定義）を持つ。

| id | name | 用途 |
|---|---|---|
| market | 市場・需要調査 | 規模・トレンド把握 |
| insight | 消費者インサイト発掘 | ニーズ・CEP発見 |
| journey | 購買ジャーニー分析 | 意思決定プロセス追跡 |
| competitor | 競合・ブランド分析 | ブランドスイッチ把握 |
| content | コンテンツ・SEO戦略 | 記事テーマ・KW選定 |
| creative | 広告・クリエイティブ戦略 | 訴求軸・コピーの根拠 |
| product_dev | 新商品・サービス開発 | 未充足ニーズ発見 |
| persona | ペルソナ設計 | ターゲット人物像の具体化 |

### FEATURES（5件）
ListeningMindの機能定義。URLとパラメータ形式を管理。

| 機能名 | URL | param | multi |
|---|---|---|---|
| インテントファインダー | /ja/query | keywords | true（複数KW同時投入可） |
| クラスターファインダー | /ja/cluster | keyword | false（1つ選択） |
| パスファインダー | /ja/path/pthv | keyword | false |
| ペルソナビュー | /ja/path/prsnv | keyword | false |
| ロードビュー | /ja/path/rdv | keyword | false |

### SYSTEM_PROMPT
Gemini APIに渡すシステムプロンプト。以下を含む：
- ListeningMind全5機能の詳細な使い分け基準
- JSON出力フォーマット仕様（`recommended_features`, `keywords`, `notes`, `action_guides`）
- キーワード生成ルール（消費者語のみ、禁止ワードリスト、purposeの書き方）
- 機能別操作ガイド情報（action_guide生成用の参照データ）

### API応答のJSONスキーマ
```json
{
  "recommended_features": [
    { "feature": "機能名", "reason": "理由50字以内", "priority": 1 }
  ],
  "keywords": [
    { "keyword": "検索語句", "purpose": "意図20字以内", "applicable_features": ["機能名"] }
  ],
  "notes": "補足またはempty",
  "action_guides": {
    "機能名": "Step1:〜Step2:〜形式の操作ガイド（600字以内）"
  }
}
```

## ListeningMind MCP ツールの仕様（参考情報）

Claude.aiでListeningMind MCPツールを使用する際のナレッジ：

- `keyword_info` と `intent_finder` は `keywords`（配列）を受け取る
- `path_finder` と `cluster_finder` は `keyword`（単数の文字列）を受け取る
- `cluster_finder`: `data_type: "communities"`, `hop: 2`, `limit: 500` で消費者検索コミュニティを返す。`data_type: "all"` はより広いネットワークデータ
- 代表KW = `keyword_info` 結果から `volume_avg` が最も高いもの（直接返却されるフィールドではない）
- `path_finder` のエントリ: 対象KWで終わる = インバウンドパス、対象KWで始まる = アウトバウンドパス。両方向を明示的にラベル付けする
- `path_finder` 単体では定性的主張の根拠として不十分。`cluster_finder` と `intent_finder` を並行実行して実証的裏付けを取る
- KW文字列は `web_search` や `path_finder` で厳密にそのまま使用する。手動で変形した語句は信頼性が低い

## ターゲットニーズ整理表ワークフロー（関連スキル）

LM Navigatorと連携して使用される分析ワークフロー：
1. `cluster_finder` → クラスター構造の把握
2. `keyword_info` → 検索ボリューム・CPC等の定量データ取得
3. `web_search` → 上位コンテンツの定性分析
4. `path_finder` → 検索経路の確認

出力形式: クラスター別 × 3分析軸のカード型整理表

## コーディング規約

- 単一HTMLファイルで完結させる（外部JSファイル分割しない）
- CSS変数を `:root` で定義し一貫して使用する
- IDベースのDOM操作（`getElementById`）
- `async/await` + `try/catch` でAPI呼び出し
- テンプレートリテラルでHTML生成
- アニメーションは `fadeUp` キーフレーム + `animation-delay` でスタガー表現

## 変更履歴

### v2（現行）
- ダークテーマ → ライトテーマに変更
- ダッシュボード型レイアウト導入（サイドバー + トップバー + フッター）
- サイドバーはLM Navigatorのみで非表示状態（将来拡張用にHTML保持）
- フローバー（プログレスバー）を動的更新に変更
- 使い方ガイドをコンテンツ最上部にインラインテキストで配置
- 結果表示時のスクロール先を「推奨機能」ラベルに修正（sticky header対応）
- フッターに `© K&F Consulting LLC. All Rights Reserved.` を追加

### v1（初版）
- ダークテーマ（`#0a0a0f` 背景）
- 単一カラムレイアウト（サイドバーなし）
- Syne + Noto Sans JP のタイポグラフィ
- Gemini API経由のAI提案機能
- 7カテゴリー → 8カテゴリー（ペルソナ設計追加）
