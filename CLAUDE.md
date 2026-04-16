# Zenn Content Repository

## プロジェクト概要
Zenn記事を通じて以下2サイトへの集客を行うコンテンツマーケティングリポジトリ。

### 集客先
- **SEO CHECK**: https://seo.codequest.work — 無料SEO診断ツール（45項目チェック、サイト全体診断、改善コード自動生成）
- **CodeQuest.work**: https://codequest.work — Web制作・SEOツール開発のフリーランスサイト

### 筆者プロフィール
Web制作・SEOツール開発を行うフリーランス。技術記事と実用記事の両方を書く。

## 記事ルール

### Zennフロントマター
```yaml
---
title: "記事タイトル（60文字以内推奨）"
emoji: "絵文字1つ"
type: "tech" or "idea"
topics: ["トピック1", "トピック2"]  # 最大5つ
published: false
publish_date: 2026-04-18
---
```

### 公開予約ルール（必須）
- 記事は必ず `published: false` + `publish_date: YYYY-MM-DD` で作成する
- GitHub Actions（`.github/workflows/scheduled-publish.yml`）が毎朝JST 7:00に実行され、当日以前の`publish_date`の記事を自動公開する
- 公開日の決め方:
  - **前回の公開日から3日以上空ける**（1日2記事以上はスパム扱いのリスク）
  - 既存の予約記事と重複しないよう、先に `grep -r 'publish_date:' articles/` で確認する
- 即時公開したい場合は `published: true` にして `publish_date` を付けずにpushする
- Qiitaと同じ記事を書く場合、**同じ日に公開する**（Zenn/Qiitaで1記事ずつ = 合計2記事/日）

### 記事構成パターン
1. **tech記事** — 技術的な実装・設計の解説。コード例を含む
2. **idea記事** — ノウハウ・活用法・提案手法の解説。ビジネス視点を含む

### CTA（集客導線）ルール
- 記事末尾に必ず筆者紹介 + サービスリンクを配置
- 記事内容に関連する場合のみ、本文中でも自然にSEO CHECKに言及してよい
- 押し売りにならないよう、あくまで「使ったツール」「関連ツール」として紹介
- フォーマット:
```markdown
---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/) で活動中。
[SEO CHECK](https://seo.codequest.work/) — ディレクターも使っているSEO診断ツールを公開中。
```

### 文体・スタイル
- 「です・ます」調
- 技術用語は正確に、ただし初心者にもわかる補足を添える
- 箇条書き・表・コードブロックを積極的に使い、読みやすく
- 1記事2000〜5000文字が目安

## スキル一覧

| スキル | ファイル | 用途 |
|--------|----------|------|
| zenn-seo-writer | `.claude/skills/zenn-seo-writer.md` | SEO最適化されたZenn記事の執筆 |
| zenn-article-reviewer | `.claude/skills/zenn-article-reviewer.md` | 記事のSEO・集客観点レビュー |
| zenn-content-strategy | `.claude/skills/zenn-content-strategy.md` | コンテンツ戦略・キーワード選定 |
| zenn-cta-optimizer | `.claude/skills/zenn-cta-optimizer.md` | CTA・集客導線の最適化 |
