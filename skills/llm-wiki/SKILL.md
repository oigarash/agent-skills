---
name: llm-wiki
description: Operate an LLM-maintained personal wiki at /Users/oigarash/notes following Karpathy's LLM Wiki pattern. Use when the user mentions "LLM Wiki", "wiki ingest", "wiki query", "wiki lint", "notes wiki", "ノートWiki", "メモをingest", "Wikiに追加", "Wikiで調べて", or wants to drop a source (PDF/URL/text/chat log) into their notes for incremental knowledge-base maintenance. Vault is Obsidian-compatible.
---

# LLM Wiki Skill

Karpathy の LLM Wiki パターン（https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f）に沿って、`/Users/oigarash/notes` に置かれた個人 Wiki を維持する。Vault は Obsidian 互換。

## 事前確認（全オペ共通）

1. `/Users/oigarash/notes/AGENTS.md` を読み、運用ルールを確認
2. `/Users/oigarash/notes/wiki/index.md` を読み、既存ページの全体像を把握
3. 対象操作（ingest / query / lint）をユーザに確認、または文脈から判定

## オペレーション

### ingest

新しいソースを Wiki に取り込み、関連ページ群を更新する。
詳細手順: `references/ingest-procedure.md`

トリガ例:
- 「このPDFをingestして」
- 「inboxに入れたメモを処理して」
- 「このURLをWikiに追加して」

### query

Wiki を使って質問に答え、新規知見があれば Wiki に還元する。

手順:
1. `wiki/index.md` から関連ページを特定
2. 必要なら `wiki/pages/` と `wiki/topics/` を Grep で補足検索
3. 該当ページを読み、引用・出典明記で回答
4. 回答過程で得た新しい整理・観点を関連ページに追記、または新規ページ化
5. `wiki/log.md` に1行追記: `YYYY-MM-DD HH:MM [query] "<質問要旨>" -> <参照ページ>`

トリガ例:
- 「Wikiで〇〇について調べて」
- 「メモにあった△△の件、まとめて」

### lint

Wiki の健全性チェック。破壊的編集はせず、レポートとして出力する。
詳細手順: `references/lint-checklist.md`

トリガ例:
- 「Wikiをlintして」
- 「メモの矛盾をチェックして」

## ページ作成・更新ルール

必ず `references/page-template.md` のテンプレートに従う。

**TL;DR ファースト（必須）**: すべての調査・ingest ページは冒頭に `## TL;DR` を置き、ここだけ読めば対象の全体像が掴めるようにする。処理フロー・ワークフロー・構造がある対象なら俯瞰図（ASCII or 箇条書き）を TL;DR に含める。詳細は後続セクションに展開する。既存ページを更新する際も、TL;DR を欠いていたら最初に補う。

Obsidian 互換のため:

- ファイル名はノートタイトルそのまま（日本語・スペース可、禁則文字 `\/:*?"<>|` のみ回避）
- フロントマターは `aliases` / `tags` / `created` / `updated` / `sources` を含む YAML
- リンクは Obsidian wikilink `[[ノート名]]` のみ。Markdown link は使わない
- 画像/PDF 埋め込みは `![[file]]`
- タグのネストは `/` 区切り（例: `project/wiki`）

## 参照ファイル

- `references/page-template.md` — ページテンプレート
- `references/ingest-procedure.md` — ingest 詳細手順
- `references/lint-checklist.md` — lint チェック項目

## ログ書式（log.md 追記）

```
YYYY-MM-DD HH:MM [ingest|query|lint] <1行要約>
```

日時は `date '+%Y-%m-%d %H:%M'` で取得。
