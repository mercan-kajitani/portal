[README.md](https://github.com/user-attachments/files/25510132/README.md)
# Mercari Commons — リバースドキュメンテーション

> **目的**：このドキュメントは、`index.html` のプロトタイプを起点に、エンジニアが本実装を進めるためのリバースドキュメントです。「何がどこにあり、どういう意図で設計されているか」を UI→仕様の順で記述します。

---

## 1. プロジェクト概要

| 項目 | 内容 |
|---|---|
| **プロダクト名** | Mercari Commons |
| **タグライン** | CIRCULATE VALUE · EXPAND POSSIBILITY |
| **コンセプト** | メルカリ社員が毎朝開くイントラポータル。ミッション「あらゆる価値を循環させ、あらゆる人の可能性を広げる」を体現した情報の流通拠点 |
| **主な対象ユーザー** | メルカリグループ全社員 |
| **アクセス制御** | @mercari.com ドメインのGoogleアカウントのみ（Cloudflare Access 等で認証） |
| **現在の状態** | 静的HTMLプロトタイプ（`index.html` 単一ファイル） |

---

## 2. ページ構成マップ

```
┌─────────────────────────────────────────────────────────┐
│  HEADER：ロゴ + Universal Search + ユーザーアバター       │
├─────────────────────────────────────────────────────────┤
│  TRENDING BAR：急上昇キーワード（タグ）                   │
├─────────────────────────────────────────────────────────┤
│  今日の一言 BOARD：LED電光掲示板風・引用テキスト           │
├──────────────────────────────┬──────────────────────────┤
│  MAIN COLUMN (左 / 約70%)    │  SIDEBAR (右 / 約30%)    │
│                              │                          │
│  ① Resonance Feature         │  ⑤ クイックアクセス       │
│     大型バナー（今週の主役）   │     （経費・カレンダー等） │
│                              │                          │
│  ② All-hands アーカイブ       │  ⑥ 今週のメルカリ数字     │
│     直近3回の動画＋議事録      │     （KPIメトリクス）      │
│                              │                          │
│  ③ トピックス タイムライン     │  ⑦ メディアティッカー     │
│     カテゴリタブ＋記事一覧     │     （外部報道ローテ）     │
│                              │                          │
│  ④ みんなの反応               │  ⑧ オウンドメディア       │
│     絵文字リアクション＋コメント│     （4媒体最新記事）     │
└──────────────────────────────┴──────────────────────────┘
```

---

## 3. コンポーネント仕様

### 3-1. HEADER — Universal Search

| 要素 | 説明 |
|---|---|
| **ロゴ** | "Mercari Commons"。左上固定。クリックでトップへ戻る |
| **検索窓** | Drive / Slack / Notion / GitHub / mercan を横断する Universal Search。現状はUI only。Glean API または社内検索基盤と接続する想定 |
| **ソースバッジ** | 検索対象サービスを視覚化するラベル。クリックでソース絞り込みに使う（未実装） |
| **日付表示** | `new Date().toLocaleDateString('ja-JP')` で自動生成 |
| **アバター** | ログインユーザーのイニシャル表示。SSO連携後は Google Profile 画像に置き換え |

**本実装時の接続先**
```
GET /api/search?q={query}&sources=drive,slack,notion,github,mercan
→ { results: [ { title, url, source, snippet, date } ] }
```

---

### 3-2. TRENDING BAR — 急上昇キーワード

| 要素 | 説明 |
|---|---|
| **タグ一覧** | 社内で検索・閲覧が急増しているキーワードをタグ表示 |
| **クリック動作** | タグをクリック → 検索窓に自動入力して検索実行 |
| **更新頻度** | リアルタイム（現状は静的ダミーデータ） |

**本実装時の接続先**
```
GET /api/trending-keywords?limit=8&window=24h
→ { keywords: [ { label, count, delta } ] }
```

---

### 3-3. 今日の一言 BOARD

| 要素 | 説明 |
|---|---|
| **デザイン** | 黒地＋琥珀色テキスト（LED電光掲示板風）|
| **テキスト** | 「今最も読まれているページ」から1文を抜粋・スクロール表示 |
| **ソース表示** | 引用元の記事名と媒体を右端に表示 |
| **ローテーション** | 30秒ごとに次の引用に切り替わる（`setInterval`） |
| **スクロールアニメ** | CSS `@keyframes quoteScroll`（28秒で右→左） |

**本実装時のデータモデル**
```json
{
  "quotes": [
    {
      "text": "引用テキスト（1〜2文）",
      "source_title": "記事タイトル",
      "source_media": "mercan",
      "source_url": "https://...",
      "view_rank": 1
    }
  ]
}
```

**編集フロー（案）**：EBチームが週次でNotion/CMS上で登録 → APIで配信

---

### 3-4. RESONANCE FEATURE — 大型バナー

| 要素 | 説明 |
|---|---|
| **目的** | 今週「最も熱量が高い」コンテンツを1本だけ大フィーチャー |
| **選出基準** | 閲覧数×リアクション数×コメント数の複合スコア（またはEBチームが手動指定） |
| **コンテンツ** | タイトル・リード文・著者・媒体・日付・熱量インジケーター |
| **クリック動作** | モーダルで本文プレビューを表示。「続きを読む」で外部URLへ |
| **バリューバッジ** | Go Bold / All for One / Be a Pro のいずれかを手動タグ付け |
| **LIVE STORY バッジ** | 今週のコンテンツであることを示す。先週以前は非表示にする |

**本実装時のデータモデル**
```json
{
  "resonance": {
    "id": "string",
    "value_tag": "go_bold | all_for_one | be_a_pro",
    "title": "string",
    "lead": "string（200字以内）",
    "author_name": "string",
    "author_team": "string",
    "media": "mercan | engineering_blog | ...",
    "url": "https://...",
    "published_at": "ISO8601",
    "heat_score": 0.0,
    "is_live": true
  }
}
```

---

### 3-5. ALL-HANDS アーカイブ

| 要素 | 説明 |
|---|---|
| **表示件数** | 直近3回を固定表示 |
| **各アイテム** | 動画サムネイル・タイトル・日付・尺・議事録リンクのセット |
| **動画** | 社内動画基盤（Vimeo / YouTube Private / Wistia 等）へのリンク |
| **議事録** | Google Docs / Notion のURL |
| **権限** | 全社員が閲覧可能。動画は社内SSO認証済みユーザーのみ |

**本実装時のデータモデル**
```json
{
  "allhands": [
    {
      "id": "string",
      "title": "2026 Feb All-hands — Q1 OKR進捗と新戦略",
      "date": "2026-02-14",
      "duration_min": 42,
      "video_url": "https://...",
      "minutes_url": "https://...",
      "is_recent": true
    }
  ]
}
```

---

### 3-6. トピックス タイムライン

タブで6カテゴリに分類。カテゴリの定義は以下の通り。

| タブ名 | `data-cat` | 定義 |
|---|---|---|
| すべて | `all` | 全カテゴリを表示 |
| 社内 🏢 | `internal` | 全社向けメッセージ・D&I活動・社内施策の報告など |
| オウンドメディア ✍️ | `owned` | メルカリが運営する4媒体の最新記事（下記参照）|
| プレスリリース 📣 | `press` | jp-news.mercari.com 発信の公式プレスリリース |
| メディア掲載 PR経由 📰 | `pr-media` | PR室が働きかけて掲載されたアーンドメディア記事 |
| 外部言及 🔍 | `organic` | PR関与なし。外部ライター・ユーザーが自発的に言及した記事・投稿 |

**オウンドメディア 4媒体**

| 媒体名 | URL | テーマ |
|---|---|---|
| Mercari Engineering Blog | https://engineering.mercari.com/blog/ | 技術・エンジニアリング |
| Mercari Design Note | https://note.com/mercari_design | デザイン・UX |
| メルカリニュース | https://jp-news.mercari.com/ | 公式ニュース・IR |
| merpoli | https://merpoli.mercari.com/ | ポリシー・トラスト・サステナビリティ |

**各トピックアイテムのデータモデル**
```json
{
  "id": "string",
  "category": "internal | owned | press | pr-media | organic",
  "media_name": "日本経済新聞",
  "value_tag": "go_bold | all_for_one | be_a_pro | null",
  "headline": "string",
  "url": "https://...",
  "published_at": "ISO8601",
  "reactions": { "🔥": 112, "💡": 67 },
  "comment_count": 14
}
```

**タブ切り替え**：JSの `switchTab()` がすべてのアイテムを `data-cat` で show/hide。タブ選択時に `tabDescriptions` オブジェクトからカテゴリ定義文を表示。

---

### 3-7. みんなの反応（リアクション＋コメント）

| 要素 | 説明 |
|---|---|
| **絵文字リアクション** | Slackライクな絵文字ボタン。クリックでトグル（カウント増減） |
| **「＋」ボタン** | ランダム絵文字を追加（本実装では絵文字ピッカーUIに置き換え） |
| **コメント入力** | Enter or ボタンで投稿。アバター付き吹き出しで一覧表示 |
| **編集者ピックアップ** | `📌 編集者ピックアップ` バッジ。EBチームが手動で付与する想定 |
| **対象コンテンツ** | 現状は Resonance Feature 固定。本実装では各記事ごとにスレッドを持つ |

**本実装時の接続先（例）**
```
POST /api/reactions   { content_id, emoji }
POST /api/comments    { content_id, body }
GET  /api/comments    { content_id } → { comments: [...] }
PATCH /api/comments/:id/pin  （編集者ピックアップ）
```

---

### 3-8. サイドバー — クイックアクセス

| ボタン | リンク先（本実装時） |
|---|---|
| 💴 経費精算 | 社内経費システムURL |
| 📅 カレンダー | Google Calendar |
| 🏖️ 休暇申請 | 勤怠管理システムURL |
| 🎁 福利厚生 | 福利厚生ポータルURL |
| 🤝 リファラル採用 | 社内採用ツールURL |
| 📚 社内Wiki | Notion / Confluence URL |

**頻度の高いリンクを上位に。ユーザーごとにカスタマイズできるよう「ピン留め」機能を将来追加検討。**

---

### 3-9. サイドバー — 今週のメルカリ数字

| 指標 | 現状 | 本実装時 |
|---|---|---|
| MAU | ダミー（5秒ごとに微変動でライブ感演出） | データ基盤APIから取得（公開可能な指標のみ） |
| GMV | ダミー | 同上 |
| NPS | ダミー | 同上 |
| インシデント解決率 | ダミー | モニタリング基盤から取得 |

**注意**：開示可能な指標の範囲を法務・IR部門と確認の上で実装すること。

---

### 3-10. サイドバー — メディアティッカー

| 要素 | 説明 |
|---|---|
| **デザイン** | 黒地・外部メディア掲載情報をローテーション |
| **更新頻度** | 6秒ごとにフェードで切り替え（`setInterval`） |
| **内容** | 媒体名＋見出しのセット |
| **本実装** | `pr-media` カテゴリのフィードと連動させる |

---

## 4. コンテンツ管理フロー（編集チーム向け）

```
EBチーム（編集者）
  │
  ├─ Resonance Feature の選出・更新（週1回）
  ├─ 「今日の一言」の登録・管理（週次）
  ├─ コメントの編集者ピックアップ（随時）
  └─ トピックスへの記事登録（随時）
        ↓
      CMS（Notion / Contentful / 自社CMS 等）
        ↓
      API → Mercari Commons フロントエンド
```

---

## 5. バリューバッジ仕様

メルカリの3バリューに対応するタグ。記事の「メッセージの核」に応じて付与。

| バッジ | 意味 | 配色 | 使用例 |
|---|---|---|---|
| `Go Bold` | 大胆な意思決定・技術挑戦・新規開拓 | 黒地・赤文字 | 生成AI活用、新機能リリース、大型ピボット |
| `All for One` | 組織横断の連携・D&I・チームワーク | 青地・青文字 | D&I施策、All-hands、チーム協働事例 |
| `Be a Pro` | 専門性・品質・プロとしての姿勢 | 緑地・緑文字 | エンジニアブログ、SRE事例、設計判断 |

バッジなし＝バリューに紐付けにくい純粋な情報提供コンテンツ（プレスリリース等）。

---

## 6. レスポンシブ設計

### ブレークポイント

| 名称 | 幅 | 主な変更 |
|---|---|---|
| Desktop | > 900px | 2カラム（メイン70% + サイドバー30%） |
| Tablet | ≤ 900px | 1カラム。サイドバーがメインの下に移動 |
| Mobile | ≤ 600px | 下記の詳細参照 |
| Small Mobile | ≤ 380px | フォントサイズ・タップ領域をさらに最適化 |

### モバイル（≤600px）の主な変更内容

| コンポーネント | 変更内容 |
|---|---|
| **Header** | 高さ52px、日付非表示、ロゴサブテキスト非表示、検索フォント16px（iOS zoom防止） |
| **Trending Bar** | 横スクロール対応（スクロールバー非表示）、タグを折り返さない |
| **今日の一言** | ソース表示（右側）を非表示、テキスト12px |
| **Resonance** | 高さ220px、リード文非表示、熱量バー非表示、タイトル16px |
| **Topics タブ** | 横スクロール（スクロールバー非表示）、タブ12px |
| **サムネイル** | 64×50px（通常80×60px） |
| **クイックアクセス** | 3カラム（通常2カラム） |
| **モーダル** | 画面下からシート形式で表示（`border-radius: 16px 16px 0 0`、高さ最大92vh）|
| **Toast通知** | 画面下端・横幅フル・中央揃え |
| **Hover効果** | `@media (hover: none)` でタッチデバイスの意図しないhoverを抑制 |

### iOS対応の注意点

- `<input>` の `font-size` を `16px` に設定（これを下回るとSafariが自動ズームする）
- `-webkit-overflow-scrolling: touch` で慣性スクロールを有効化
- `scrollbar-width: none` + `::-webkit-scrollbar { display: none }` でスクロールバーを非表示
- モーダルはボトムシート形式にして親指で操作しやすく

## 7. 技術スタック（現状 → 本実装案）


| レイヤー | 現状 | 本実装候補 |
|---|---|---|
| フロントエンド | 純HTML/CSS/JS（単一ファイル） | Next.js / React |
| スタイリング | Inline CSS | Tailwind CSS or CSS Modules |
| 状態管理 | なし | Zustand / Jotai |
| 認証 | なし（プロトタイプ） | Cloudflare Access + Google OAuth（@mercari.com 限定）|
| CMS | なし | Contentful / Notion API / 自社CMS |
| 検索 | なし | Glean API / Elasticsearch |
| リアクション・コメント | ローカルJS | Firebase Realtime DB / 自社API |
| KPIデータ | ダミー | 社内データ基盤API（BigQuery等） |
| ホスティング | GitHub Pages（プロトタイプ） | Cloudflare Pages / Vercel（本番）|

---

## 8. 未実装・今後の検討事項

- [ ] ユーザー認証（SSO連携）
- [ ] 検索機能（Universal Search）の実装
- [ ] コメント・リアクションの永続化（DB連携）
- [ ] Resonance Feature の自動スコアリング
- [ ] トピックス記事のCMS管理画面
- [ ] KPIメトリクスのリアルタイム化
- [ ] プッシュ通知（重要ニュース・Resonance更新）
- [ ] モバイルアプリ版（PWA化）
- [ ] 「今日の一言」の編集者登録UI
- [ ] クイックアクセスのユーザーカスタマイズ（ピン留め）
- [ ] タグ・バリューバッジによる横断フィルタリング

---

## 9. ファイル構成（現状）

```
mercari-pulse/
├── index.html    # 全コンポーネントを含む単一ファイル（プロトタイプ）
└── README.md     # 本ドキュメント
```

**本実装時の推奨構成（Next.js の場合）**
```
mercari-commons/
├── app/
│   ├── page.tsx              # トップページ
│   ├── layout.tsx            # 共通レイアウト
│   └── api/
│       ├── search/route.ts   # Universal Search
│       ├── topics/route.ts   # トピックス一覧
│       ├── reactions/route.ts
│       └── comments/route.ts
├── components/
│   ├── Header/
│   ├── TrendingBar/
│   ├── QuoteBoard/           # 今日の一言
│   ├── ResonanceFeature/
│   ├── AllHandsArchive/
│   ├── TopicsTimeline/
│   ├── ReactionBar/
│   ├── CommentSection/
│   └── Sidebar/
├── lib/
│   ├── cms.ts                # CMS接続
│   └── analytics.ts          # アクセス解析
└── README.md
```

---

*Last updated: 2026-02-24 | Author: EBチーム*
*このドキュメントは `index.html` プロトタイプから自動・手動で生成されたリバースドキュメントです。*
