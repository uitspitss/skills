# skills

個人用の Agent Skills 置き場。開発環境セットアップ用の `setup-dev:*` を管理している。

以前は chezmoi（`uitspitss/dotfiles`）で管理していたが、独立させて [vercel-labs/skills](https://github.com/vercel-labs/skills) CLI 経由でインストールする形に移行した。

## インストール

```sh
npx skills add uitspitss/skills -g --all
```

`-g` はユーザーレベル（グローバル）、`--all` は全スキル × 全エージェント一括。private リポジトリだが、`gh auth` 済みなら clone できる。

インストール先は `~/.agents/skills/setup-dev-*`（実体のコピー）で、`~/.claude/skills/` からはそこへ symlink が張られる。

## 更新フロー

**このリポジトリを編集しても即座には反映されない。** インストール時にコピーされるため、push してから update する。

```sh
git commit -am "..." && git push
npx skills update
```

## スキル一覧

| スキル | 内容 |
| --- | --- |
| `setup-dev:tooling` | 共通ツール（mise、pnpm、TypeScript、oxlint、oxfmt、lefthook、knip、worktrunk、dotenvx） |
| `setup-dev:tsconfig` | tsconfig.json の規約（target / module / moduleResolution / strict） |
| `setup-dev:next` | Next.js（App Router）の環境構築・更新 |
| `setup-dev:vite-react` | Vite + React SPA（TanStack Router / Query / Form） |
| `setup-dev:expo` | Expo / React Native（expo-router、NativeWind v4） |
| `setup-dev:monorepo-cloudflare` | Cloudflare 上のモノレポ（pnpm workspaces + Turborepo、Hono API） |
| `setup-dev:testing` | Vitest + Testing Library |
| `setup-dev:e2e` | @playwright/test による E2E |
| `setup-dev:storybook` | Storybook 10 + addon-vitest |
| `setup-dev:ci` | GitHub Actions CI、react-doctor、Cloudflare deploy |
| `setup-dev:dependabot` | `.github/dependabot.yml` の生成 |
| `setup-dev:portless` | portless による `.localhost` URL + HTTPS + 自動ポート割当 |

`tooling` と `tsconfig` は他のスキルから参照される土台。

## スキルを追加・編集するときの注意

`SKILL.md` の frontmatter は厳密な YAML としてパースされる。Claude Code 本体は寛容に読むが、skills CLI は弾く。

`description` の値が YAML の予約文字（`` ` `` `@` `&` `*` `!` `|` `>` `%` `{` `[` `#` `,` `-`）で始まる場合はダブルクォートで囲むこと。

```yaml
# NG — パースエラーでスキップされる
description: `.github/dependabot.yml` を生成する。

# OK
description: "`.github/dependabot.yml` を生成する。"
```

追加後は `npx skills add uitspitss/skills -l` で全件が `Found N skills` に出るか確認できる。
