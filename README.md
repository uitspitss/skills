# skills

> [!WARNING]
> **個人用です。汎用的なスキル集ではありません。**
>
> - **Agent Skills の書き方を学ぶための習作**として作っています。完成品ではありません。
> - 筆者のプロジェクトで毎回やるセットアップを固めただけの、**ボイラープレートに近い内容**です。
>   選定・規約はすべて筆者の好みに寄っています。
> - **ほとんどがバイブコーディングで書かれており、中身の精度は保証しません。**
>   レビューや動作検証を通した管理はしていません。

TypeScript / React 系プロジェクトの開発環境を、決まった構成で立ち上げ・更新するための Agent Skills 置き場。

共通の土台は mise + pnpm + TypeScript + oxlint + oxfmt + Vitest + lefthook。
そこに Next.js / Vite SPA / Expo / Cloudflare モノレポといったスタック別のスキルが乗る。

## 前提

- Node.js（`npx` が使えること）
- [mise](https://mise.jdx.dev/) と pnpm — 各スキルが生成する設定はこの2つ前提
- Claude Code などの Agent Skills 対応エージェント

## インストール

```sh
npx skills add uitspitss/skills -g -s '*' -a claude-code -y
```

[vercel-labs/skills](https://github.com/vercel-labs/skills) CLI 経由でインストールする。
`-g` はユーザーレベル（グローバル）、`-s '*'` は全スキル、`-a` は対象エージェント。

他のエージェントにも入れるならカンマ区切り（`-a claude-code,codex`）。有効な識別子の一覧は
`-a bogus` など不正な値を渡すと表示される。

`--all`（= `-s '*' -a '*' -y`）は使わないこと。Eve と PromptScript がグローバルインストールに
非対応なため、必ずその2つで失敗表示が出る。Claude Code へのインストール自体は成功しているので
ノイズでしかないが、`-a` で絞れば消える。

インストール先は `~/.agents/skills/setup-dev-*`（実体のコピー）で、`~/.claude/skills/` からは
そこへ symlink が張られる。

## 使い方

全スキルに `disable-model-invocation: true` を付けてあるため、エージェントが勝手に起動することはない。
使うときは明示的に呼ぶ。

```
/setup-dev:next
```

新規セットアップ・既存プロジェクトの更新・他ツールからの移行を、スキル側がプロジェクトの状態を見て
判定する。

## 収録スキル

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

`tooling` と `tsconfig` は他のスキルから参照される土台。スタック別スキル（`next` など）を呼べば
必要に応じて自動で参照される。

## 更新

```sh
npx skills update
```

**このリポジトリを編集しても即座には反映されない。** インストール時にコピーされるため、
push してから update する。

```sh
git commit -am "..." && git push
npx skills update
```

## 開発メモ

自分用の落とし穴メモ。fork して使う場合も同じ制約にかかる。

### ディレクトリ名にコロンを使わない

**スキルのディレクトリ名は `setup-dev-next` のようにハイフンにする**（frontmatter の
`name: setup-dev:next` はコロンのままでよい）。`npx skills update` は内部で
「リポジトリ + スキルのフォルダ名」を連結したソースで add を再実行するため、
フォルダ名にコロンがあると scp 形式のリモート（`host:path`）と解釈されて clone が落ちる:

```
skills add uitspitss/skills/setup-dev:next --skill setup-dev:next -g -y
fatal: 'uitspitss/skills/setup-dev:next' does not appear to be a git repository
→ ✗ Failed to update setup-dev:next
```

ディレクトリを改名すると lock の `skillPath` がずれるので、一度 `npx skills add ... -a claude-code -y`
で入れ直すこと。以降は `npx skills update` が通る。

### frontmatter は厳密な YAML

`SKILL.md` の frontmatter は厳密な YAML としてパースされる。Claude Code 本体は寛容に読むが、
skills CLI は弾く。

`description` の値が YAML の予約文字（`` ` `` `@` `&` `*` `!` `|` `>` `%` `{` `[` `#` `,` `-`）で
始まる場合はダブルクォートで囲むこと。

```yaml
# NG — パースエラーでスキップされる
description: `.github/dependabot.yml` を生成する。

# OK
description: "`.github/dependabot.yml` を生成する。"
```

追加後は `npx skills add uitspitss/skills -l` で全件が `Found N skills` に出るか確認できる。
