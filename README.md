# agent-skills

Personal Agent Skills monorepo. Each skill lives under `skills/<name>/SKILL.md`, optionally with
a `references/` directory for templates and deeper documentation.

For the behavior, triggers, and usage of any individual skill, see that skill's `SKILL.md`.

## Skills

- [cmux](skills/cmux/SKILL.md) — cmux のトポロジ／ルーティング制御（window/workspace/pane/surface の inspect・focus・move・reorder）
- [customize-claude-code](skills/customize-claude-code/SKILL.md) — Claude Code カスタマイズの実践リファレンス（hooks / settings.json / CLAUDE.md / シェルスクリプトの罠）
- [cxj-dify-maintenance](skills/cxj-dify-maintenance/SKILL.md) — cxj-dify プラットフォームの運用保守・Admin API 管理（鍵ローテーション／ヘルスチェック／LLMプロバイダ設定／プラグイン管理）
- [etrade-kakutei-sinkoku](skills/etrade-kakutei-sinkoku/SKILL.md) — E*TRADE (Morgan Stanley at Work) から日本の確定申告データを収集・計算・出力（RSU／ESPP／配当の外国税額控除）
- [k8s-gitlab-cicd-nextjs](skills/k8s-gitlab-cicd-nextjs/SKILL.md) — Next.js を GitLab CI/CD (Kaniko) で Kubernetes へデプロイする構成テンプレート
- [llm-wiki](skills/llm-wiki/SKILL.md) — LLM 保守型パーソナル Wiki（Karpathy の LLM Wiki パターン、Obsidian 互換）の ingest／query／lint 運用
- [skill-feedback](skills/skill-feedback/SKILL.md) — Skill 使用時に気づいた改善点を Issue/PR/ローカル patch に振り分ける
