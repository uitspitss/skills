---
name: setup-dev:next
description: Next.js（App Router）の開発環境をセットアップ・更新する。oxlint、oxfmt、Vitest、lefthook、knip、worktrunk、GitHub Actions CI。`next lint` / ESLint+Prettier からの移行と、メジャーアップグレードにも対応。
disable-model-invocation: true
---

# Next.js Development Environment Setup

新規プロジェクトのセットアップに加え、既存 Next.js プロジェクトのツール更新と Biome / ESLint+Prettier / `next lint` から oxlint+oxfmt への移行に対応する。`$ARGUMENTS` が指定されていればそのディレクトリ、なければカレントディレクトリで動作する。

**参照スキル:** 該当ステップでは共通スキルの内容に従うこと。

- `setup-dev:tooling` - mise, pnpm, TypeScript, oxlint, oxfmt, lefthook, knip, worktrunk, dotenvx（モード判定を含む）
- `setup-dev:tsconfig` - TypeScript 設定の共通方針
- `setup-dev:testing` - Vitest + Testing Library の設定
- `setup-dev:ci` - GitHub Actions の設定
- `setup-dev:storybook` - Storybook + addon-vitest（オプション）。`setup-dev:testing` の後に適用する
- `setup-dev:shadcn` - shadcn/ui の導入（オプション）。UI の書き方そのものは `shadcn` スキルが持つ
- `setup-dev:dependabot` - 依存更新自動化（オプション）
- `setup-dev:portless` - 複数 worktree のポート衝突対策（オプション）

## ステップ 0: 対象の確認（何よりも先に実行する）

```bash
# Next.js プロジェクトか
ls next.config.* 2>/dev/null
cat package.json 2>/dev/null | jq -r '.dependencies + .devDependencies | keys[]?' 2>/dev/null | grep -E '^next$'
# ルーターの種類
ls -d app pages 2>/dev/null
# バージョン
cat package.json 2>/dev/null | jq -r '.dependencies.next // .devDependencies.next'
```

| 検出結果 | どうするか |
|---|---|
| 何も無い（空ディレクトリ） | このスキルで新規作成 |
| `next` があり `app/` がある | App Router。このスキルをそのまま適用 |
| `next` があり `pages/` だけ | **Pages Router**。ツール類（oxlint/oxfmt/Vitest/lefthook/knip/CI）は適用できるが、本スキルの App Router 前提の記述（`next typegen`、`app/` のディレクトリ構成）は読み替えが必要。ユーザーに確認する |
| `vite` を検出 | このスキルではなく **`setup-dev:vite-react`** |
| `expo` / `react-native` を検出 | **`setup-dev:expo`** |
| pnpm workspace / `turbo.json` を検出 | **`setup-dev:monorepo-cloudflare`** |

**次に `setup-dev:tooling` の「モード判定」に従い、`fresh` / `migrate-from-biome` / `migrate-from-eslint-prettier` / `update` を決定する。** 以降の各ステップは新規セットアップ前提で書かれているが、`update` / `migrate-*` では「既に設定済みかチェック → 不足分のみ追加」で進める。

**既存プロジェクトに適用する場合、フレームワークが生成した設定を尊重する。** 特に `tsconfig.json` の `plugins` / `include` / `jsx` は Next.js の動作に必要（後述）。

## Prerequisites Check

- `mise` (runtime version manager)
- `node` (Node.js runtime) — **Next.js 16 は Node.js 20.9+ が必須**
- `ni` (@antfu/ni - package manager agent)

## Setup Steps

### 1. Configure mise

**`setup-dev:tooling` の「mise（ランタイムのバージョン管理）」セクションに従う。** `node` / パッケージマネージャ / `npm:@antfu/ni` を固定バージョンで `mise.toml` に書き、`mise install` する。

### 2. Create or initialize the project

新規の場合:

```bash
nlx create-next-app@latest . --app --ts --no-eslint --no-tailwind --use-pnpm
```

**`--no-eslint` を付ける理由**: このスキルは oxlint を使う。ESLint を入れてから消すのは二度手間で、`next lint` は Next.js 16 で削除済み（後述）。

**`--use-pnpm` は手順 1 で決めたパッケージマネージャに合わせる**（`--use-bun` / `--use-npm` / `--use-yarn`）。`create-next-app` 自身に渡すフラグなので `ni` の自動判別は効かず、間違えると別のロックファイルが生成される。

既存プロジェクトの場合はこのステップをスキップ。`package.json` に `"type": "module"` を追加する（ESM only パッケージの互換性のため）。プロジェクトに `.js` の設定ファイル（`next.config.js` 等）がある場合は `.mjs` / `.ts` に寄せるか、`"type": "module"` を見送る。

### 3. Install / upgrade Next.js and React

```bash
ni next@latest react@latest react-dom@latest
ni -D typescript @types/node @types/react @types/react-dom
```

**既存プロジェクトのメジャーアップグレード時は、必ず公式のアップグレードガイドを読む**（`https://nextjs.org/docs/app/guides/upgrading/version-<N>`）。codemod も用意されている:

```bash
nlx @next/codemod@canary upgrade latest
```

TypeScript のバージョン選定は **`setup-dev:tooling` の「TypeScript 7（ネイティブ版）のインストール」** に従う。Next.js 側の対応状況によっては最新が使えないことがあるので、`nr typecheck` と `nr build` を通してから確定させる。

### 4. Configure tsconfig.json

**Next.js が生成した `tsconfig.json` をベースにする。`setup-dev:tsconfig` の「Base compilerOptions」のうち、Next が管理していない項目（`target` 等）だけを揃える。**

**Next.js が管理していて手で変えてはいけないもの**:

| 項目 | 値 | 理由 |
|---|---|---|
| `jsx` | Next が決める（16 では `react-jsx`） | 手で書き換えても `next typegen` / `next build` が上書きする |
| `plugins` | `[{ "name": "next" }]` | エディタの型サポート |
| `include` | `next-env.d.ts`, `.next/types/**/*.ts`, `.next/dev/types/**/*.ts` | 型付きルートの生成物。**`exclude` に `.next` を足すとこれが落ちて型検査が壊れる** |
| `noEmit` | `true` | 出力は Next が持つ。`outDir` / `declaration` / `rootDir` は付けない |

**`types/css.d.ts` を追加する**:

```ts
// next の型定義は `*.module.css` しか宣言していない。
// 素のグローバル CSS（app/globals.css）の side-effect import は宣言が無いため、
// TypeScript 7 が TS2882 "Cannot find module or type declarations for
// side-effect import" で弾く。tsc 5.x は黙認していたが 7 は許さない。
declare module "*.css";
```

画像やその他のアセットを直接 import している場合も同様に宣言を足す。

### 4-b. (Optional) shadcn/ui

素の Tailwind でコンポーネントを積み上げる前に入れておくと後が楽。
**`setup-dev:shadcn` に従う。** Next.js 固有の差分だけここに書く。

- **`@/*` エイリアスは `create-next-app` が既に張っていることが多い。**
  `tsconfig.json` の `paths` を確認し、無ければ init の前に足す
  （shadcn の CLI はエイリアスを検出して `components.json` に焼き込むので、
  用意する前に init すると別のものを誤検出する）
- **`components.json` の `rsc` が `true` になる。** App Router では、生成された
  コンポーネントをそのまま使う分には問題ないが、**`useState` / `useEffect` /
  イベントハンドラ / ブラウザ API を使う自前のコンポーネントには `"use client"` が要る**。
  shadcn の生成物は必要なものに既に付いている
- Tailwind v4 なら `tailwind.config` は無い。init は `app/globals.css`
  （`components.json` の `tailwind.css` が指すファイル）を**上書きする**ので、
  自前のカスタムプロパティを書いていたら先に退避する
- ステップ 4 で足した `types/css.d.ts` の `declare module "*.css"` は
  shadcn を入れても引き続き必要

### 5. Install and Configure oxlint + oxfmt

**`setup-dev:tooling` の「oxlint」「oxfmt」セクションに従う。** 単一プロジェクト用（`-w` なし）。`migrate-*` モードの場合は `setup-dev:tooling` の `references/migration.md` を先に実行する。

**`.oxlintrc.json` に `nextjs` プラグインを足し、App Router で誤検知するルールを調整する**:

```json
{
  "$schema": "https://raw.githubusercontent.com/oxc-project/oxc/main/npm/oxlint/configuration_schema.json",
  "plugins": ["typescript", "react", "react-hooks", "import", "nextjs"],
  "categories": {
    "correctness": "error",
    "suspicious": "warn",
    "perf": "warn",
    "style": "off"
  },
  "rules": {
    "react/react-in-jsx-scope": "off",

    "nextjs/no-page-custom-font": "off",

    "nextjs/no-html-link-for-pages": "warn",

    "import/no-unassigned-import": ["warn", { "allow": ["**/*.css"] }]
  }
}
```

| ルール | 設定 | 理由 |
|---|---|---|
| `nextjs/no-page-custom-font` | `off` | Pages Router の `_document.js` を前提にしたルール。App Router では `app/layout.tsx` の `<head>` に font link を置くのが正しいので**誤検知**になる |
| `nextjs/no-html-link-for-pages` | `warn` | 内部リンクが `<a>` のままだと全ページリロードになる。`next/link` に寄せるべきだが、静的寄りの読み物サイトでは許容する判断もある。**プロジェクトの方針をユーザーに確認し、決めた理由を AGENTS.md に書く** |
| `import/no-unassigned-import` | `*.css` を許可 | `import "./globals.css"` は Next の定石 |

**`next lint` は使わない。** Next.js 16 で削除されており、`next build` も lint を実行しなくなった。ESLint から移行する場合は codemod がある:

```bash
nlx @next/codemod@canary next-lint-to-eslint-cli .
```

ただし本スキルは oxlint に寄せるので、`setup-dev:tooling` の `references/migration.md` の「ESLint + Prettier から oxlint + oxfmt への移行」に従って ESLint ごと外す。`next.config.*` の `eslint` オプションも削除する（16 で無効）。

**oxfmt の `.oxfmtrc.json`**: 既存の手書き CSS がある場合は `**/*.css` を除外対象に入れる（`setup-dev:tooling` の「フォーマット対象」の表を参照）。

### 6. Install and Configure Vitest + Testing Library

**`setup-dev:testing` の「React フロントエンド向け（full setup）」に従う。**

**`include` を必ず Next のディレクトリ構成に合わせる。** Next.js プロジェクトは `src/` を持たないことが多く、雛形の `src/**` をそのまま使うと1件もマッチせず、`passWithNoTests: true` と組み合わさって**テスト0件のまま CI が緑になる**:

```ts
// src/ を使わない構成
include: ["{app,components,lib}/**/*.{test,spec}.{ts,tsx}"],
```

`src/` を使う構成（`create-next-app --src-dir`）ならそのままでよい。

**tsconfig の `paths` エイリアスを vitest 側にも通す**:

```ts
import { fileURLToPath } from "node:url";

export default defineConfig({
  resolve: {
    // tsconfig の "@/*" -> "./*" を vitest でも解決させる
    alias: { "@": fileURLToPath(new URL(".", import.meta.url)) },
  },
  test: { /* ... */ },
});
```

**Server Component と route handler のテストについて**: `async` な Server Component は Testing Library の `render()` で直接描画できない。テストの対象は「純粋な表示コンポーネント」「`lib/` のロジック」「route handler の入力バリデーション」に絞り、データ取得を伴うページは E2E に回す。**その E2E 側は `setup-dev:e2e` を参照**（`@playwright/test`、`webServer` に `next dev` ではなく本番ビルドを起動、認証は setup project + `storageState`）。Vitest の `include` と Playwright の既定パターンが `.spec.ts` を取り合う事故も同スキルに書いてある。

### 7. Install and Configure knip

**`setup-dev:tooling` の「knip」セクションに従う。** knip は Next.js プラグインを自動検出するので `entry` は基本不要。`project` はソース配置に合わせる:

```json
{
  "$schema": "https://unpkg.com/knip@latest/schema.json",
  "project": ["{app,components,lib,types}/**/*.{ts,tsx}"],
  "rules": {
    "devDependencies": "warn",
    "exports": "warn"
  }
}
```

### 8. Install and Configure lefthook

**`setup-dev:tooling` の「lefthook」セクションに従う。** `typecheck` は `nr typecheck` 経由にすること（手順 10 の理由による）。

### 9. Configure worktrunk

**`setup-dev:tooling` の「worktrunk」セクションに従う。**

**`copy-ignored` を使う場合、`.worktreeinclude` を必ず作る。** これが無いと gitignore された全ファイルがコピーされ、**`.next` のビルドキャッシュが古い worktree から持ち込まれて Turbopack が壊れる**（CPU 張り付き・dev サーバー応答不能の実害あり）:

```text
# .worktreeinclude
.env*
```

`.next/` と `node_modules/` は**含めない**（`node_modules` は pnpm のハードリンクで `ni` が数秒で張り直す。コピーの方が遅い）。

#### Optional: portless

複数 worktree で同時に dev サーバーを起動するなら **`setup-dev:portless`** に従う。

### 10. Add npm Scripts to package.json

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "oxlint .",
    "lint:fix": "oxlint --fix .",
    "format": "oxfmt .",
    "format:check": "oxfmt --check .",
    "test": "vitest run",
    "test:watch": "vitest",
    "typegen": "next typegen",
    "typecheck": "next typegen && tsc --noEmit",
    "knip": "knip"
  }
}
```

**`typecheck` が `next typegen` を先に走らせる理由**: `next-env.d.ts` と `.next/types/` は gitignore された生成物で、クリーンなチェックアウト（CI、新しい worktree、`.next` を消した後）には存在しない。tsconfig がそれらを `include` しているため、生成せずに `tsc` を回すと型検査が落ちる。`next typegen` はフルビルドせずに型だけ生成する。

**`--turbopack` フラグは付けない。** Next.js 16 から Turbopack が `next dev` / `next build` の既定になっている。カスタム webpack 設定を持つプロジェクトは `next build` が**失敗する**ので、その場合のみ `--webpack` で opt-out するか、設定を Turbopack に移行する。

### 11. Create .gitignore

**`setup-dev:tooling` の「.gitignore（共通ベース）」に従い、Next.js 固有の項目を足す**:

```
# Next.js
.next/
out/
next-env.d.ts

# Vercel
.vercel/
```

- `.next/` は dev と build の両方の出力を含む（Next.js 16 は dev を `.next/dev` に分離した）
- `next-env.d.ts` は生成物。コミットしない代わりに、`typecheck` で必ず `next typegen` を通す（手順 10）
- 環境変数ファイル（`.env`, `.env.local` 等）の扱いは、dotenvx を使うかどうかで変わる。`setup-dev:tooling` の該当節を参照

### 12. Set up GitHub Actions

**`setup-dev:ci` の「CI Workflow」と「react-doctor Workflow」に従う。**

**`nr build` に環境変数が必要かを必ず確認する。** Next.js では以下のケースで build 時に外部リソースへアクセスする:

- DB クライアントをモジュールのトップレベルで初期化している（`lib/db.ts` で `throw` する類）
- ISR / SSG のページが build 時にデータを取りに行く

該当する場合は build ステップに env を渡す:

```yaml
      - run: nr build
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

**fork からの pull_request では secrets が渡らない**（空文字になる）。外部コントリビュータを受け入れるなら、build を `push` イベントに限定するか、ビルド専用のダミー値で通るようコードを変える。

#### Optional: Dependabot

**`setup-dev:dependabot`** に従って `.github/dependabot.yml` を作成する。

### 13. Deployment

Vercel が既定の想定。`DATABASE_URL` などの環境変数を Vercel 側に設定しておけば、`vercel --prod` あるいは GitHub 連携で公開できる。

Vercel 以外（Cloudflare Workers、self-host）に出す場合は、使っている機能（ISR、Image Optimization、Node API）がターゲットで動くかを個別に確認すること。

### 14. Create AGENTS.md

開発ルールの実体は `AGENTS.md` に書き、`CLAUDE.md` はそれを参照するだけにする（Claude 以外のエージェントからも同じルールを読めるようにするため）。

`CLAUDE.md`:

```markdown
@AGENTS.md
```

`AGENTS.md`:

```markdown
# 開発ルール

## このプロジェクトの構成

Next.js <version> (App Router)。Vite ではないので、Vite 前提の手順をそのまま持ち込まないこと。

## コード変更後の検証

- `nr lint` (oxlint)
- `nr format:check` (oxfmt)
- `nr typecheck` (next typegen && tsc --noEmit)

`nr typecheck` が `next typegen` を先に走らせるのは、`next-env.d.ts` と `.next/types/` が
gitignore された生成物で、クリーンな環境には存在しないため。

## テスト

- TDD（テスト駆動開発）で実装する
- テストを先に書き、実装はテストが通るように行う
- `nr test` (vitest run) / `nr test:watch`
- DOM テストは happy-dom + Testing Library。`vitest.setup.ts` で `cleanup` を明示登録している
- async な Server Component は render() できない。表示コンポーネントとロジックに絞る

## Next.js まわりの注意

- ビルドと dev は Turbopack がデフォルト（`--turbopack` フラグは不要）
- `tsconfig.json` の `jsx` は Next が管理する。手で書き換えても上書きされる
- `next lint` は使わない（Next.js 16 で削除済み）。lint は oxlint
- route handler に `export const runtime = "edge"` を足さない（Next.js 16 で deprecated。
  ページの静的生成も無効化される）

## ファイル命名規則

- kebab-case を使用する (例: `my-component.tsx`, `use-auth.ts`)

## リンタ設定で意図的に緩めているもの

`.oxlintrc.json` を参照。<プロジェクトで実際に設定した内容と理由を書く>

## 環境変数

<必須の環境変数と、build にも必要かどうかを書く>
```

### 15. Create README.md

プロジェクト名と説明に加えて、以下を含める。

- **Getting Started**: `mise install` → `ni` → `nr dev`。必須の環境変数があれば明記する
- **開発コマンド**: 上記 scripts の表
- **Project Structure**: `app/` のルーティング構成
- **Tech Stack**: 使用ライブラリ

### 16. Final Verification

```bash
nr typecheck
nr lint
nr format:check
nr test
nr build
nr knip
```

**`nr test` の出力の「Test Files」が0件でないことを目視する**（手順 6 の `include` が効いていない場合、`passWithNoTests` で緑になってしまう）。

**`nr build` に環境変数が必要な場合**、ローカルで一度は実データを使って通しておく。ダミー値では「コンパイルは通るが静的生成で落ちる」という状態になり、CI で初めて発覚する。

`update` / `migrate-*` モードでは、移行前後で `nr lint` / `nr format:check` の差分を一覧化してユーザーに確認すること。

## Important Notes

- ユーザー確認なしに既存の設定ファイルを上書きしない（`setup-dev:tooling` の「上書きルール」を参照）
- **`tsconfig.json` の Next.js 管理項目を触らない**（手順 4 の表）。特に `exclude` に `.next` を足すと型付きルートが落ちる
- **メジャーアップグレードでは公式のアップグレードガイドを読む。** Next.js は 15 → 16 で async Request API の同期アクセス削除、`middleware` → `proxy` 改名、Turbopack 既定化、`next lint` 削除、`next/image` の各種デフォルト変更などを入れている。プロジェクトが該当するかを1つずつ確認する
- oxlint の `nextjs` プラグインは Pages Router 前提のルールを含む。App Router で誤検知したものは**理由をコメントに残して off にする**。無言で off にすると、次に見た人が「なぜ切ってあるのか」を判断できない
