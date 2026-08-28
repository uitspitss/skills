---
name: setup-dev:vite-react
description: Vite + React SPA の開発環境をセットアップ・更新する。TanStack Router / Query / Form、oxlint、oxfmt、Vitest、lefthook、react-doctor CI。Next.js は setup-dev:next、Expo は setup-dev:expo、モノレポは setup-dev:monorepo-cloudflare を使う。
disable-model-invocation: true
---

# TypeScript/React Development Environment Setup

新規プロジェクトのセットアップに加え、既存プロジェクトのツール更新と Biome / ESLint+Prettier から oxlint+oxfmt への移行にも対応する。`$ARGUMENTS` が指定されていればそのディレクトリ、なければカレントディレクトリで動作する。

## ステップ 0: フレームワーク判定（何よりも先に実行する）

**このスキルの手順 3〜7 と 13 は Vite プロジェクトを前提にしている。他のフレームワークのプロジェクトに適用すると、既存のルーティング・API・レンダリング戦略を丸ごと捨てて作り直すことになる。** 対象ディレクトリが本当に Vite（あるいは空）かを最初に確認すること。

```bash
# 別フレームワークの設定ファイル
ls next.config.* nuxt.config.* astro.config.* svelte.config.* remix.config.* app.json 2>/dev/null
# 依存の確認
cat package.json 2>/dev/null | jq -r '.dependencies + .devDependencies | keys[]?' 2>/dev/null \
  | grep -E '^(next|nuxt|astro|@remix-run/.*|@sveltejs/kit|expo|react-native|@angular/core)$'
```

| 検出結果 | どうするか |
|---|---|
| 何も無い / `vite` がある | このスキルをそのまま実行 |
| `next` を検出 | このスキルではなく **`setup-dev:next`** を使う |
| `expo` / `react-native` を検出 | このスキルではなく **`setup-dev:expo`** を使う |
| pnpm workspace / turbo.json を検出 | このスキルではなく **`setup-dev:monorepo-cloudflare`** を使う |
| `nuxt` / `astro` / `@remix-run/*` / `@sveltejs/kit` を検出 | 専用スキルが無い。**手順を進めずユーザーに確認する**（下記） |

### 専用スキルの無いフレームワークを検出したときの確認

「検出したフレームワーク」と「そのまま進めた場合に壊れるもの」を具体的に挙げ、次の3択をユーザーに提示する:

1. **ツール類のみ適用（推奨）** — 手順 3〜7 と 13 を飛ばし、手順 8〜12・14〜21（oxlint/oxfmt, Vitest, knip, lefthook, worktrunk, CI, .gitignore, AGENTS.md, README）だけを適用する。既存のフレームワークはそのまま
2. **Vite + TanStack へ全面移行** — 既存フレームワークを捨てて作り直す。何が失われるか（サーバーコンポーネント、API ルート、SSR/ISR、ファイルベースルーティング等）を列挙してから合意を取る
3. **移行は検討だけ** — ツール類を入れ、移行プランは文書で提示するに留める

**1 を選ばれた場合、各ステップの「Vite 前提」の記述は読み替えが必要になる**。特に:

- `tsconfig.json` — フレームワーク公式の設定（`jsx: "preserve"`、`plugins`、生成物の `include` 等）を尊重する。`setup-dev:tsconfig` の「既存プロジェクトの tsconfig を触るときの原則」を参照
- Vitest の `include` — `src/` を持たない構成が多い。`setup-dev:testing` の注意を必ず読む
- knip の `project` — 同上
- npm scripts — `dev` / `build` / `preview` はフレームワークのものを残す。追加するのは lint / format / test / typecheck / knip だけ

**最初に `setup-dev:tooling` の「モード判定」セクションに従い、`fresh` / `migrate-from-biome` / `migrate-from-eslint-prettier` / `update` のどれで動作するかを決定すること。** 以降の各ステップは新規セットアップ前提で書かれているが、`update` / `migrate-*` モードでは「既に設定済みかチェック → 不足分のみ追加 / 必要な置換のみ実施」の方針で進める。

**参照スキル:** このスキルは以下の共通スキルを参照する。該当ステップでは共通スキルの内容に従うこと。

- `setup-dev:tsconfig` - TypeScript 設定の共通方針
- `setup-dev:tooling` - oxlint, oxfmt, lefthook, knip, worktrunk, dotenvx の設定（モード判定を含む）
- `setup-dev:testing` - Vitest + Testing Library の設定
- `setup-dev:ci` - GitHub Actions の設定
- `setup-dev:storybook` - Storybook + addon-vitest（オプション）。Next.js で検証されたスキルなので、Vite 側の差分は末尾の節を読むこと
- `setup-dev:shadcn` - shadcn/ui の導入（オプション）。UI の書き方そのものは `shadcn` スキルが持つ
- `setup-dev:e2e` - Playwright での E2E（オプション）。`setup-dev:testing` とは別物
- `setup-dev:dependabot` - 依存更新自動化（オプション）

## Prerequisites Check

Before starting, verify that the following tools are installed:
- `mise` (runtime version manager)
- `node` (Node.js runtime)
- `ni` (@antfu/ni - package manager agent)

If any are missing, inform the user and suggest installation commands.

## Setup Steps

### 1. Configure mise

**`setup-dev:tooling` の「mise（ランタイムのバージョン管理）」セクションに従う。** `node` / パッケージマネージャ / `npm:@antfu/ni` を固定バージョンで `mise.toml` に書き、`mise install` する。

このスキルではパッケージマネージャはユーザーの好みに合わせる（`pnpm` or `bun`）。

### 2. Initialize package.json

If `package.json` doesn't already exist, create one:

```bash
ni --init
```

`package.json` に `"type": "module"` を追加する（ESM only パッケージの互換性のため）。

### 3. Install Vite + React + TypeScript

```bash
ni react react-dom
ni -D typescript @types/node @types/react @types/react-dom vite @vitejs/plugin-react babel-plugin-react-compiler
```

Create `vite.config.ts`:
```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [
    react({
      babel: {
        plugins: [["babel-plugin-react-compiler"]],
      },
    }),
  ],
});
```

Create `tsconfig.json` — **`setup-dev:tsconfig` の「Base compilerOptions」+「React フロントエンド向け追加設定」に従う。**

### 4. Install and Configure TanStack Router

```bash
ni @tanstack/react-router
ni -D @tanstack/router-plugin
```

`vite.config.ts` に TanStack Router プラグインを追加:

```ts
import { TanStackRouterVite } from "@tanstack/router-plugin/vite";

export default defineConfig({
  plugins: [TanStackRouterVite(), react()],
});
```

ファイルベースルーティングのディレクトリ構造を作成:

```
src/
  routes/
    __root.tsx      # ルートレイアウト
    index.tsx       # "/" ページ
  routeTree.gen.ts  # 自動生成（コミットする）
```

`src/routes/__root.tsx`（TanStack DevTools の UI コンポーネントもここに配置。**本番バンドルから除外するため dev のみで動的ロード**）:
```tsx
import { createRootRoute, Outlet } from "@tanstack/react-router";
import { lazy, Suspense } from "react";

const TanStackDevtools = import.meta.env.DEV
  ? lazy(() =>
      import("@tanstack/react-devtools").then((m) => ({
        default: m.TanStackDevtools,
      })),
    )
  : () => null;

export const Route = createRootRoute({
  component: () => (
    <>
      <Outlet />
      {import.meta.env.DEV && (
        <Suspense fallback={null}>
          <TanStackDevtools />
        </Suspense>
      )}
    </>
  ),
});
```

**なぜ動的 import + DEV ガード**: 静的 import だと本番ビルドに `@tanstack/react-devtools` のコードが含まれ、UI が露出する/バンドルが膨らむ可能性がある。`import.meta.env.DEV` は Vite が build 時に定数置換するため、本番では `lazy()` の枝ごと dead code elimination で除去される。

`src/routes/index.tsx`:
```tsx
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/")({
  component: () => <div>Hello World</div>,
});
```

`src/main.tsx` でルーターを作成:
```tsx
import { createRouter, RouterProvider } from "@tanstack/react-router";
import { routeTree } from "./routeTree.gen";

const router = createRouter({ routeTree });

// ... ReactDOM.createRoot で <RouterProvider router={router} /> をレンダリング
```

### 5. Install and Configure TanStack Query

```bash
ni @tanstack/react-query zod
```

`src/main.tsx` で `QueryClient` と `QueryClientProvider` をセットアップ:
```tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

const queryClient = new QueryClient();

// <QueryClientProvider client={queryClient}> で <RouterProvider /> をラップ
```

### 6. Install and Configure TanStack Form

```bash
ni @tanstack/react-form @tanstack/zod-form-adapter
```

TanStack Form は zod と連携してフォームバリデーションを提供する。`@tanstack/zod-form-adapter` を使い、zod スキーマでバリデーションを定義する。

### 6-b. (Optional) shadcn/ui

素の Tailwind でコンポーネントを積み上げる前に入れておくと後が楽。
**`setup-dev:shadcn` に従う。**

**ステップ 3 の直後（`@/*` エイリアスを張った状態）で入れること。** shadcn の CLI は
import エイリアスを検出して `components.json` に焼き込むので、用意する前に init すると
別のエイリアスを誤検出する。

### 7. Install react-scan, react-grab, and @tanstack/devtools

```bash
ni -D react-scan react-grab @tanstack/devtools-vite @tanstack/react-devtools
```

開発時のみ有効化する。`src/main.tsx` のエントリポイントに以下を追加:

```ts
if (import.meta.env.DEV) {
  import("react-scan").then(({ scan }) => {
    scan({ enabled: true });
  });
  // react-grab: UI 要素をホバー → ⌘C で HTML / コンポーネント / ソース位置 (file:line:col) を
  // コーディングエージェント（Claude Code 等）に渡せる。副作用 import するだけで有効化。
  import("react-grab");
}
```

**react-scan と react-grab の役割の違い**: どちらも aidenybai 作・bippy ベースで「dev のみ・要素オーバーレイ」という基盤は共通だが、機能は重複しない。react-scan は不要な再レンダリングを可視化する**パフォーマンス**ツール、react-grab は要素のコンテキストを **AI エージェントに渡す**ツール。用途が直交するので両方入れてよい。

**本番に絶対含めない**: react-grab は React 内部 (Fiber) とソース位置を露出するため、必ず `import.meta.env.DEV` ガード内に置く。Vite が build 時に枝ごと dead code elimination で除去する。

`vite.config.ts` に `@tanstack/devtools-vite` プラグインを追加（**dev サーバー時のみ適用**）:

```ts
import { devtools } from "@tanstack/devtools-vite";

export default defineConfig({
  plugins: [
    { ...devtools(), apply: "serve" },
    react(),
  ],
});
```

`apply: "serve"` で dev サーバー時のみプラグインを有効化し、本番 build には devtools-vite が一切関与しないようにする。

### 8. Install and Configure oxlint + oxfmt

**`setup-dev:tooling` の「oxlint」「oxfmt」セクションに従う。** 単一プロジェクト用（`-w` なし）でインストール。`migrate-from-biome` / `migrate-from-eslint-prettier` モードの場合は、`setup-dev:tooling` の `references/migration.md` の該当セクション（「Biome から〜」または「ESLint + Prettier から〜」）を先に実行すること。

### 9. Install and Configure Vitest + Testing Library

**`setup-dev:testing` の「React フロントエンド向け（full setup）」に従う。**

`vitest.config.ts` を別ファイルにする場合、**`vite.config.ts` の `resolve.alias` は
引き継がれない。** ステップ 3 以降で `@/*` エイリアスを張っているなら
`vitest.config.ts` にも同じ alias を書くこと（詳細は `setup-dev:testing` の
「`vitest.config.ts` は `vite.config.ts` を継承しない」）。

### 9-b. (Optional) Storybook

コンポーネントカタログを作る／ストーリーをブラウザ実機テストとして CI で回すなら
**`setup-dev:storybook` の「Vite SPA の場合」に従う。** ステップ 9 を先に済ませてから。

- framework は `@storybook/react-vite`
- `base` を `/` 以外にしている場合、`viteFinal` で打ち消す必要がある
- `index.html` の `<html class="dark">` のようなルート属性は preview に伝わらない

### 9-c. (Optional) E2E

サーバーを起動して確かめるものは Vitest ではなく Playwright に置く。
**`setup-dev:e2e` に従う。**

`setup-dev:testing` の `include` と Playwright の既定パターンは **`.spec.ts` を
取り合う**。E2E を入れる予定があるなら、ステップ 9 の時点で
`include` をディレクトリ明示にしておくと後で直さずに済む。

### 10. Install and Configure knip

**`setup-dev:tooling` の「knip」セクションに従う。** 単一プロジェクト用の `knip.json` を使用。

### 11. Install and Configure lefthook

**`setup-dev:tooling` の「lefthook」セクションに従う。**

### 12. Configure worktrunk

**`setup-dev:tooling` の「worktrunk」セクションに従う。**

#### Optional: portless（複数 worktree でのポート衝突対策）

複数 worktree で同時に dev サーバーを起動する運用を想定する場合は、**`setup-dev:portless` スキルに従って portless を設定する。** worktrunk のブランチ名を `.localhost` サブドメインに自動付加することでポート衝突を防ぐ。

不要なら省略可。後から導入しても OK。

### 13. Create Directory Structure

```
src/
  main.tsx
  routes/
    __root.tsx
    index.tsx
index.html
```

`index.html` は Vite のエントリポイントとして作成し、`<div id="root">` と `<script type="module" src="/src/main.tsx">` を含める。

### 14. Add npm Scripts to package.json

```json
{
  "scripts": {
    "build": "vite build",
    "dev": "vite",
    "preview": "vite preview",
    "lint": "oxlint .",
    "lint:fix": "oxlint --fix .",
    "format": "oxfmt .",
    "format:check": "oxfmt --check .",
    "test": "vitest run",
    "test:watch": "vitest",
    "typecheck": "tsc --noEmit",
    "knip": "knip"
  }
}
```

### 15. Create .gitignore

**`setup-dev:tooling` の「.gitignore（共通ベース）」に従う。**

### 16. Set up GitHub Actions

**`setup-dev:ci` の「CI Workflow」と「react-doctor Workflow」に従う。**

#### Optional: Dependabot

依存更新を自動化したい場合は **`setup-dev:dependabot` スキルに従って `.github/dependabot.yml` を作成する。** 単一プロジェクト向けのテンプレ（npm + github-actions）が用意されている。後から導入しても OK。

### 17. (Optional) Cloudflare Workers Deployment

ユーザーが Cloudflare Workers へのデプロイを希望する場合に設定する。

```bash
ni -D wrangler @cloudflare/workers-types
```

Create `wrangler.jsonc`:
```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "project-name",
  "compatibility_date": "2026-03-01",
  "assets": {
    "directory": "dist"
  }
}
```

`tsconfig.json` の `types` に `@cloudflare/workers-types` を追加。

npm scripts に追加:
```json
{
  "scripts": {
    "deploy": "vite build && wrangler deploy",
    "deploy:preview": "vite build && wrangler dev"
  }
}
```

**`setup-dev:ci` の「Cloudflare Deploy Workflow（単一プロジェクト）」に従い** `.github/workflows/deploy.yml` を作成。

`.gitignore` に `.wrangler/` を追加。

### 18. Configure mise with @antfu/ni

`mise use` で ni をインストール対象に追加:

手順 1 で `npm:@antfu/ni` も一緒に固定済みなら、このステップは不要。入れていなければ **`setup-dev:tooling` の「mise」セクション**に従って追加する。

`mise.toml` に `"npm:@antfu/ni" = "<固定バージョン>"` があれば、`mise install` で `ni`/`nr`/`nci`/`nlx`/`na` が使えるようになる。CI でも `jdx/mise-action@v2` が自動でインストールする。

### 19. Create AGENTS.md

プロジェクトの開発ルールは `AGENTS.md` に書き、`CLAUDE.md` はそれを参照するだけにする（Claude 以外のエージェントからも同じルールを読めるようにするため）。

`CLAUDE.md`:

```markdown
@AGENTS.md
```

`AGENTS.md`:

```markdown
# 開発ルール

## コード変更後の検証

コードを変更したら、以下を実行してエラーがないことを確認する:

- `nr lint` (oxlint)
- `nr format:check` (oxfmt)
- `nr typecheck` (tsc --noEmit)

## テスト

- TDD（テスト駆動開発）で実装する
- テストを先に書き、実装はテストが通るように行う

## ファイル命名規則

- kebab-case を使用する (例: `my-component.tsx`, `use-auth.ts`)
- PascalCase は使わない

## 推奨 Claude Code スキル

このプロジェクトでは以下のスキルの使用を推奨:
- tanstack-router - ファイルベースルーティング
- tanstack-query - サーバー状態管理
- frontend-design - フロントエンド UI 作成
```

### 20. Create README.md

プロジェクト名と簡単な説明を含む README.md を生成する。以下のセクションを含めること:

#### Getting Started

```bash
# ランタイムのインストール
mise install

# 依存関係のインストール
ni

# 開発サーバーの起動
nr dev
```

#### Available Scripts

| コマンド | 説明 |
|---|---|
| `nr dev` | 開発サーバーを起動 |
| `nr build` | プロダクションビルド |
| `nr preview` | ビルド結果をプレビュー |
| `nr lint` | oxlint でコードチェック |
| `nr lint:fix` | oxlint で自動修正 |
| `nr format` | oxfmt でフォーマット |
| `nr format:check` | oxfmt でフォーマット差分の検出のみ |
| `nr test` | テストを実行 |
| `nr test:watch` | テストをウォッチモードで実行 |
| `nr typecheck` | TypeScript の型チェック |
| `nr knip` | 未使用コード・依存関係の検出 |

#### Project Structure

```
src/
  main.tsx              # エントリポイント
  routes/
    __root.tsx          # ルートレイアウト
    index.tsx           # "/" ページ
```

#### Tech Stack

使用しているライブラリ・ツールの一覧を簡潔に記載する。

### 21. Final Verification

Run the following to verify everything is working:
```bash
nr typecheck
nr lint
nr format:check
nr test
nr build
nr knip
```

Report the results to the user。`update` / `migrate-*` モードでは、移行前後で `nr lint` / `nr format:check` の差分を一覧化してユーザーに確認すること。

## Important Notes

- ユーザー確認なしに既存の設定ファイルを上書きしない（`setup-dev:tooling` の「上書きルール」を参照）
- `update` モードの場合: 既に入っているパッケージや scripts はスキップし、不足分だけ追加する。`oxlint` / `oxfmt` のバージョンが古いだけなら `nu oxlint oxfmt` でアップグレード
- `migrate-from-biome` モードの場合: `setup-dev:tooling` の `references/migration.md` の「Biome から oxlint + oxfmt への移行」を最初に実行し、その後でこのスキルの未適用ステップを反映
- `migrate-from-eslint-prettier` モードの場合: `setup-dev:tooling` の `references/migration.md` の「ESLint + Prettier から oxlint + oxfmt への移行」を最初に実行（ESLint だけ / Prettier だけ片方の場合も該当節のみ実行）。`.vscode/` のフォーマッタ設定や CI の `eslint`/`prettier` 直接呼び出しも合わせて置換する
- プロジェクト固有の設定（カスタムルール、独自エイリアス、追加プラグイン）は**保持する**。新規セットアップで提示する設定はあくまで雛形なので、既存の意図を上書きしない
