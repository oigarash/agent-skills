# agent-skills

Personal Agent Skills monorepo. Each skill lives under `skills/<name>/SKILL.md`, optionally with
a `references/` directory for templates and deeper documentation.

For the behavior, triggers, and usage of any individual skill, see that skill's `SKILL.md`.

## Skills

- [agent-skill-install](skills/agent-skill-install/SKILL.md) — グローバル Agent Skill の導入・整理ルール（正本は home repo、グローバルには npx 管理か symlink で参照を置き実体を置かない／作成先の振り分け／監査）
- [cmux](skills/cmux/SKILL.md) — cmux のトポロジ／ルーティング制御（window/workspace/pane/surface の inspect・focus・move・reorder）
- [customize-claude-code](skills/customize-claude-code/SKILL.md) — Claude Code カスタマイズの実践リファレンス（hooks / settings.json / CLAUDE.md / シェルスクリプトの罠）
- [cxj-dify-maintenance](skills/cxj-dify-maintenance/SKILL.md) — cxj-dify プラットフォームの運用保守・Admin API 管理（鍵ローテーション／ヘルスチェック／LLMプロバイダ設定／プラグイン管理）
- [etrade-kakutei-sinkoku](skills/etrade-kakutei-sinkoku/SKILL.md) — E*TRADE (Morgan Stanley at Work) から日本の確定申告データを収集・計算・出力（RSU／ESPP／配当の外国税額控除）
- [k8s-gitlab-cicd-nextjs](skills/k8s-gitlab-cicd-nextjs/SKILL.md) — Next.js を GitLab CI/CD (Kaniko) で Kubernetes へデプロイする構成テンプレート
- [llm-wiki](skills/llm-wiki/SKILL.md) — LLM 保守型パーソナル Wiki（Karpathy の LLM Wiki パターン、Obsidian 互換）の ingest／query／lint 運用
- [skip-side-quests](skills/skip-side-quests/SKILL.md) — 実装中の本筋外の問題を保留し、重大な方針変更や環境ブロッカーでは停止・報告する手動 Skill
- [skill-feedback](skills/skill-feedback/SKILL.md) — Skill 使用時に気づいた改善点を Issue/PR/ローカル patch に振り分ける
- [touch-grass](skills/touch-grass/SKILL.md) — 計画時の質問を現実的なユーザー判断に絞り、実装詳細や仮想的な corner case の深掘りを抑える手動 Skill
