---
name: setup-dev:monorepo-cloudflare
description: Cloudflare 上のモノレポ開発環境をセットアップ・更新する。pnpm workspaces + Turborepo、Hono API（Workers + D1 + R2）、Vite+React および / または Expo、Drizzle ORM、oxlint、oxfmt、Vitest、lefthook。
disable-model-invocation: true
---

# Monorepo Cloudflare Full-Stack Development Environment Setup

新規モノレポのセットアップに加え、既存モノレポのツール更新と Biome / ESLint+Prettier から oxlint+oxfmt への移行にも対応する。`$ARGUMENTS` が指定されていればそのディレクトリ、なければカレントディレクトリで動作する。

**最初に `setup-dev:tooling` の「モード判定」セクションに従い、`fresh` / `migrate-from-biome` / `migrate-from-eslint-prettier` / `update` のどれで動作するかを決定すること。** 各ステップは新規セットアップ前提で書かれているが、`update` / `migrate-*` モードでは「既に設定済みかチェック → 不足分のみ追加 / 必要な置換のみ実施」の方針で進める。

**参照スキル:** このスキルは以下の共通スキルを参照する。該当ステップでは共通スキルの内容に従うこと。

- `setup-dev:tsconfig` - TypeScript 設定の共通方針
- `setup-dev:tooling` - oxlint, oxfmt, lefthook, knip, worktrunk, dotenvx の設定（モード判定を含む）
- `setup-dev:testing` - Vitest + Testing Library の設定
- `setup-dev:ci` - GitHub Actions の設定
- `setup-dev:vite-react` - apps/web のセットアップ（フロントエンドを Web で作る場合）
- `setup-dev:expo` - apps/mobile のセットアップ（フロントエンドを Expo / React Native で作る場合）
- `setup-dev:shadcn` - shadcn/ui の導入（オプション）
- `setup-dev:e2e` - Playwright での E2E（オプション）。Cloudflare 固有の節がある
- `setup-dev:dependabot` - 依存更新自動化（オプション）

**構成:**
- **モノレポ管理**: pnpm workspaces + Turborepo
- **フロントエンド**: 以下のいずれか / 両方
  - `apps/web` — Vite + React + TanStack Router + TanStack Query
  - `apps/mobile` — Expo + expo-router + NativeWind + TanStack Query
- **バックエンド** (`apps/api`): Hono + Drizzle ORM (Cloudflare Workers + D1 + R2)
- **共有パッケージ** (`packages/shared`): フロントエンド・バックエンド間の共有型・ユーティリティ

**最初にどのフロントエンドをセットアップするかユーザーに確認すること。** `web` のみ / `mobile` のみ / 両方、いずれも有効。両方の場合は両アプリが `packages/shared` と `apps/api` を共有する。

**ローカル開発**: Docker 不要。`wrangler dev` が D1・R2 のローカルエミュレーションを提供する（`.wrangler/state/` に保存）。

## Prerequisites Check

Before starting, verify that the following tools are installed:
- `mise` (runtime version manager)
- `node` (Node.js runtime)
- `ni` (@antfu/ni - package manager agent)

If any are missing, inform the user and suggest installation commands.

## Setup Steps

### 1. Configure mise

**`setup-dev:tooling` の「mise（ランタイムのバージョン管理）」セクションに従う。** `node` / `pnpm` / `npm:@antfu/ni` を固定バージョンで `mise.toml` に書く。

モノレポではリポジトリルートの `mise.toml` 1つで全ワークスペースをカバーする。各 app / package に個別の `mise.toml` は置かない。

Run `mise install` to install the pinned versions.

### 2. Initialize Root package.json and pnpm workspaces

ルートの `package.json` を作成:
```json
{
  "name": "project-name",
  "private": true,
  "type": "module",
  "packageManager": "pnpm@<実バージョン>",
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev",
    "lint": "turbo run lint",
    "typecheck": "turbo run typecheck",
    "test": "turbo run test",
    "format": "oxfmt .",
    "format:check": "oxfmt --check .",
    "knip": "knip"
  },
  "devDependencies": {
    "turbo": "^<実バージョン>"
  }
}
```

**`"latest"` と書かない。** `ni -D turbo -w`（ステップ 3）で入れれば `^x.y.z` が
書かれるので、この雛形を手打ちで埋めるときも実バージョンにする。理由は
**`setup-dev:tooling` の「原則: バージョンは固定する」** を参照。

**`packageManager` が要る理由**: Turborepo はこのフィールドを起動時に検証し、無いと `Could not resolve workspaces. Missing packageManager field in package.json` で起動拒否する。

**`package.json` に `pnpm` フィールドを書かない。** pnpm 11 以降は読まれず、警告を出して無視される。pnpm の設定は下の `pnpm-workspace.yaml` に置く。pnpm 10 以前を使う場合の書き方を含め、詳細は **`setup-dev:tooling` の「pnpm の設定（モノレポ / 単一プロジェクト共通）」** を参照。

`pnpm-workspace.yaml` を作成（ワークスペース定義と pnpm 設定を兼ねる）:
```yaml
packages:
  - "apps/*"
  - "packages/*"

managePackageManagerVersions: false

allowBuilds:
  esbuild: true
  sharp: true
  lefthook: true
  workerd: true
  msgpackr-extract: true   # apps/mobile を含む場合（Metro が依存）

catalog:
  hono: ^4.12.25
  zod: ^4.0.0
  drizzle-orm: ^0.45.2
  react: ^19.0.0
  react-dom: ^19.0.0
```

**`managePackageManagerVersions: false`**: `packageManager` があると pnpm はそのバージョンへの自動切替を試み、mise / システムの pnpm パスと衝突して `Failed to switch pnpm to v<X.Y.Z>` で起動失敗することがある。切替を抑止し、mise が pin する pnpm をそのまま使う。

**`allowBuilds`**: pnpm 10+ はデフォルトで `postinstall` をスキップする。Metro が依存する `msgpackr-extract` のネイティブ build を許可しないと Mobile 側の `expo export` / `expo start` 時に `Unexpected end of MessagePack data` で失敗する。**pnpm 10 以前では `package.json` の `pnpm.onlyBuiltDependencies` に配列で書く**（形が違う）。

`catalog` で共通パッケージのバージョンを一元管理する。各 `package.json` では `"catalog:"` プロトコルでバージョンを参照する。

**catalog にも `latest` を書かない。** 上のバージョンは例なので、実際は
`npm view <pkg> version` で現在の値を確認して書く。`latest` のままだと
`pnpm install` のたびに解決が動き、ロックファイルの差分が
無関係な PR に混ざる（**`setup-dev:tooling` の「原則: バージョンは固定する」**）。

### 3. Install and Configure Turborepo

```bash
ni -D turbo -w
ni -D typescript -w
```

`typescript` は各 package の `typecheck` スクリプト (`tsc --noEmit`) で使う。ルートに入れておけば pnpm のシンボリックリンク経由で全 package から `tsc` バイナリが解決される。詳細は **`setup-dev:tooling` の「TypeScript 7（ネイティブ版）のインストール」** 参照。

`turbo.json` を作成:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "ui": "tui",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["$TURBO_DEFAULT$", ".env*"],
      "outputs": ["dist/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {
      "dependsOn": ["^lint"]
    },
    "typecheck": {
      "dependsOn": ["^typecheck"]
    },
    "test": {
      "dependsOn": ["^build"]
    }
  }
}
```

**turbo のタスクはパッケージ単位でしか回らない。** `lint` / `typecheck` / `test` は
「そのスクリプトを持っているパッケージ」だけが対象になる。`packages/*` に
`lint` スクリプトを置き忘れると、**そのパッケージのコードは CI の lint を
永久に素通りする**（エラーにならないので気づけない）。

導入後に `nr lint` の出力で「Tasks: N successful」の N がパッケージ数と
合っているかを確認すること。合っていなければどこかにスクリプトが無い。

```bash
# 各ワークスペースのスクリプト有無を一覧する
cat */*/package.json | jq -r '.name, (.scripts | keys)'
```

#### `wrangler dev` も assets のディレクトリを要求する

`assets.directory` が存在しないと **`wrangler dev` は起動前に落ちる**。
クリーンチェックアウトで README どおり `nr dev` した人が最初に踏む。

```
✘ [ERROR] The directory specified by the "assets.directory" field does not exist
```

dev では SPA を Vite が配るので**中身は空でよい**。dev スクリプトで作る。

```jsonc
"dev": "mkdir -p ../web/dist && wrangler dev"
```

E2E の `webServer` で web を先に build している構成だと**そちらでは再現しないので、
dev だけが壊れていることに気づけない**。セットアップ後に `rm -rf apps/web/dist` して
`nr dev` が上がるか一度確かめること。

#### Worker が SPA を配る構成では build に循環ができる

`apps/api` の `wrangler.jsonc` が `assets.directory: "../web/dist"` を指す構成
（本番を 1 Worker にまとめる形）だと、**`wrangler deploy --dry-run` は web の
ビルド成果物が無いと落ちる**。

```
✘ [ERROR] The directory specified by the "assets.directory" field does not exist:
  /path/to/apps/web/dist
```

turbo は依存グラフ上 `api -> web` を知らないので、並列に走らせて api が先に終わると
落ちる。**`@repo/api#build` に明示的な依存を書く**。

```jsonc
"@repo/api#build": { "dependsOn": ["@repo/web#build"], "outputs": ["dist/**"] }
```

ただしこれだけだと **`web -> api-client -> api -> web` の循環**になる
（web が `@repo/api-client` を、api-client が `AppType` のために `@repo/api` を
依存に持つため）。

```
The cycle can be broken by removing any of these sets of dependencies:
  @repo/web#build -> @repo/api-client#build
```

`packages/*` はソース消費で `build` スクリプトを持たないので、**web の build に
先行タスクは無い**。`^build` を外して切る。

```jsonc
"@repo/web#build": { "dependsOn": [], "outputs": ["dist/**"] }
```

デプロイの順序も同じ理由で決まる。**migration → web build → deploy**。
migration を deploy の後に置くと、新しいコードが一瞬だけ古いスキーマに当たる。

### 4. Create Directory Structure

セットアップ対象のフロントエンドに応じてディレクトリを作成する:

```
apps/
  web/           # Vite + React フロントエンド（任意）
  mobile/        # Expo + React Native フロントエンド（任意）
  api/           # Hono + Drizzle バックエンド
packages/
  shared/        # 共有型・ユーティリティ
```

```bash
# 必要なものだけ作成
mkdir -p apps/api packages/shared
mkdir -p apps/web        # web を含める場合
mkdir -p apps/mobile     # mobile を含める場合
```

---

## packages/shared のセットアップ

### 5. Initialize packages/shared

`packages/shared/package.json`:
```json
{
  "name": "@repo/shared",
  "private": true,
  "type": "module",
  "exports": {
    ".": {
      "types": "./src/index.ts",
      "default": "./src/index.ts"
    }
  },
  "scripts": {
    "typecheck": "tsc --noEmit"
  }
}
```

`packages/shared/tsconfig.json` — **`setup-dev:tsconfig` の「Base compilerOptions」+「モノレポの shared パッケージ向け追加設定」に従う。**

`packages/shared/src/index.ts` を作成（初期エクスポート用の空ファイルまたは簡単な型定義）:
```ts
export type AppEnv = {
  DATABASE_URL: string;
};
```

---

## apps/api（Hono + Drizzle）のセットアップ

**→ `references/api-setup.md` を読んで Step 6〜11 を実行する。**

内容: package.json、Hono エントリポイント、レイヤードアーキテクチャ（routes/services/repositories）、Drizzle スキーマ・マイグレーション設定、Wrangler 設定、TypeScript・Vitest 設定。

---

## フロントエンドのセットアップ

### 12. Set up frontend apps (web and/or mobile)

ユーザーに「web のみ」「mobile のみ」「両方」を確認し、対応するセクションを適用する。両方を選んだ場合は両方適用する。

#### モノレポ配下では除外する共通項目

`apps/web` でも `apps/mobile` でも、以下はルートレベルで管理するため**個別アプリでは設定しない**:

- mise の設定（ルートの `mise.toml` で管理）
- oxlint の設定（ルートの `.oxlintrc.json` で管理）
- oxfmt の設定（ルートの `.oxfmtrc.json` で管理。markdown を `ignorePatterns` で除外する。詳細は `setup-dev:tooling`）
- lefthook の設定（ルートの `lefthook.yml` で管理）
- knip の設定（ルートの `knip.json` で管理）
- worktrunk の設定（ルートの `.config/wt.toml` で管理）
- .gitignore（ルートで管理）
- GitHub Actions（ルートで管理）
- CLAUDE.md（ルートで管理）
- README.md（ルートで管理）
- Cloudflare Workers デプロイ設定（`apps/api` 側で管理）

各 setup-dev スキルの該当ステップはスキップする。

---

### 12-a. apps/web (Vite + React) — オプション

`apps/web` ディレクトリに対して **`setup-dev:vite-react` スキルの内容を適用する。**（Next.js で作る場合は代わりに **`setup-dev:next`** を適用し、以下の「適用時の差分・追加事項」を同様に反映する）

**適用時の差分・追加事項:**

#### パッケージ名とワークスペース依存

- `package.json` の `name` は `"@repo/web"` にする
- `dependencies` に `"@repo/shared": "workspace:*"`、`"hono": "catalog:"`、`"zod": "catalog:"` を追加
- `devDependencies` に `"@repo/api": "workspace:*"` を追加
- catalog で管理しているパッケージ（`react`, `react-dom` 等）は `"catalog:"` プロトコルを使う

#### Vite の proxy 設定

`vite.config.ts` の `server` に API プロキシを追加:
```ts
server: {
  proxy: {
    "/api": {
      target: "http://localhost:8787",
      changeOrigin: true,
    },
  },
},
```

#### tsconfig.json のパス設定

**`setup-dev:tsconfig` の「モノレポのパス参照設定」に従い** `paths` と `references` を追加。

#### Hono RPC クライアント

`apps/web/src/lib/api.ts` を作成し、API の型安全なクライアントを設定:
```ts
import type { AppType } from "@repo/api";
import { hc } from "hono/client";

export const api = hc<AppType>("/api");
```

**重要**: `@repo/api` の `AppType` を型としてのみインポートする（`import type`）。これによりフロントエンドにサーバーコードがバンドルされることなく、型安全な API クライアントが使える。

#### shadcn/ui を入れる場合

**`setup-dev:shadcn` に従う。** モノレポ固有の要点だけ:

- **CLI には `-c apps/web` が要る。** ルートで実行すると `monorepo_root` で止まる。
  `init` だけでなく `add` / `info` にも要る
- `@/*` エイリアスを `tsconfig.json` / `vite.config.ts` / `vitest.config.ts` の
  **3 箇所すべて**に張ってから init する。モノレポは `@repo/*` の paths が既にあるので、
  `@/*` が無いと CLI がそちらを誤検出する
- `src/components/ui/**` は生成物。knip の `ignore` に入れる（使う前でも未使用として並ぶ）

#### Storybook を入れる場合

**`setup-dev:storybook` の「monorepo の場合」に従う。** 要点だけ:

- `.storybook` は `apps/web` に**1つだけ**置き、`packages/ui` のストーリーは
  `stories` のオブジェクト形式（`{ directory, files, titlePrefix }`）で拾う。
  パッケージごとに立てると設定と CI が丸ごと二重になる
- ストーリーを置く `packages/*` には `@storybook/react-vite` と `storybook` を
  devDependency として足す。pnpm は非ホイストなので、**そのパッケージ単体の
  `tsc --noEmit` が `Meta` や `storybook/test` を解決できない**
- `turbo.json` に `storybook`（`cache: false` / `persistent: true`）と
  `build-storybook`（`outputs: ["storybook-static/**"]`）を足し、ルートの
  `package.json` から `turbo run` で呼ぶ。`pnpm --filter` を直書きしない

---

### 12-b. apps/mobile (Expo + React Native) — オプション

`apps/mobile` ディレクトリに対して **`setup-dev:expo` スキルの内容を適用する。** create-expo-app は対象ディレクトリに対して実行する:

```bash
nlx create-expo-app@latest apps/mobile --template default@next --no-install
```

**適用時の差分・追加事項:**

#### パッケージ名とワークスペース依存

- `package.json` の `name` は `"@repo/mobile"` にする
- `dependencies` に `"@repo/shared": "workspace:*"`、`"hono": "catalog:"`、`"zod": "catalog:"` を追加
- `devDependencies` に以下を追加:
  - `"@repo/api": "workspace:*"` — Hono RPC の型を参照するため
  - `"@cloudflare/workers-types": "^<実バージョン>"` — `AppType` 経由で D1/R2 ambient 型が必要になるため。pnpm の strict resolution では `apps/api` の devDep を transitively には参照できないので、mobile 側でも明示的に持つ。**`apps/api` と同じバージョンを書く**（ズレると ambient 型が食い違う）
- catalog で管理しているパッケージ（`react`, `react-dom` 等）は `"catalog:"` プロトコルを使う

#### tsconfig.json のパス設定

**`setup-dev:tsconfig` の「モノレポのパス参照設定」に従い** `paths` と `references` を追加。`setup-dev:expo` の `expo/tsconfig.base` 継承との両立は、`compilerOptions.paths` をマージする形で書く:

```json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "jsx": "react-jsx",
    "moduleResolution": "Bundler",
    "types": ["@cloudflare/workers-types"],
    "paths": {
      "@/*": ["./src/*"],
      "@repo/shared": ["../../packages/shared/src"],
      "@repo/api": ["../api/src/index.ts"]
    }
  },
  "references": [{ "path": "../../packages/shared" }],
  "include": [
    "**/*.ts",
    "**/*.tsx",
    ".expo/types/**/*.ts",
    "expo-env.d.ts",
    "nativewind-env.d.ts",
    "types.d.ts"
  ]
}
```

**`@repo/api` を paths に登録する理由**: `apps/mobile/src/lib/api.ts` で `import type { AppType } from "@repo/api"` を解決するため。`apps/api/src/index.ts` の `typeof app` を直接参照する形なので、ファイルパスを paths で直接ポイントする。pnpm の workspace 解決だけでは TypeScript が型を見つけられない場合がある。

**`types: ["@cloudflare/workers-types"]` を追加する理由**: `apps/api/src/index.ts` の `Bindings` 型が `D1Database` / `R2Bucket`（Cloudflare の ambient 型）を参照しているため、mobile 側から `AppType` を import すると Cloudflare 型の宣言が必要になる。これを満たすため、mobile の devDependencies に `@cloudflare/workers-types` を追加し、tsconfig の types で明示的に読み込ませる。

> 設計上はやや汚染的（mobile が Cloudflare 型を持つ）だが、Hono RPC の型推論を活かす（`typeof app` を export する）以上、現状はこの妥協が現実的。将来、`AppType` を Cloudflare 型に依存しない形にリファクタリングできれば外せる。

#### Hono RPC クライアント

`apps/mobile/src/lib/api.ts` を作成:
```ts
import type { AppType } from "@repo/api";
import { hc } from "hono/client";

const baseUrl = process.env.EXPO_PUBLIC_API_URL ?? "http://localhost:8787";

export const api = hc<AppType>(baseUrl);
```

**Web との違い**: Vite のプロキシ機能に相当するものが Expo にはないため、API ベース URL を `EXPO_PUBLIC_API_URL` 環境変数で切り替える。実機 / Android Emulator から接続する場合は `localhost` ではなく LAN IP / `10.0.2.2` を指定する。

#### Metro の monorepo 対応

Expo の Metro バンドラはデフォルトで `node_modules` の解決を現在のディレクトリ起点で行うため、pnpm workspaces のシンボリックリンクを正しく追えないことがある。`apps/mobile/metro.config.js` を以下のように調整する:

```js
const { getDefaultConfig } = require("expo/metro-config");
const { withNativeWind } = require("nativewind/metro");
const path = require("node:path");

const projectRoot = __dirname;
const workspaceRoot = path.resolve(projectRoot, "../..");

const config = getDefaultConfig(projectRoot);

// モノレポのルート node_modules も解決対象にする
config.watchFolders = [workspaceRoot];
config.resolver.nodeModulesPaths = [
  path.resolve(projectRoot, "node_modules"),
  path.resolve(workspaceRoot, "node_modules"),
];
// pnpm の strict resolution では .pnpm/<pkg>/node_modules/<dep> 階層を辿る必要があるため
// 階層解決を有効化する。Expo 公式の monorepo ガイドは npm/yarn 用に true を推奨するが、
// pnpm では false が必須(true にすると expo-router の deep 依存が UnableToResolveError になる)
config.resolver.disableHierarchicalLookup = false;

module.exports = withNativeWind(config, { input: "./src/global.css" });
```

**`disableHierarchicalLookup = false` を選ぶ理由**: Expo 公式は `true` を推奨するが、これは npm / yarn 想定。pnpm では各パッケージが `.pnpm/<pkg>@<ver>/node_modules/<pkg>/` の階層で配置され、依存先は階層を辿らないと見つからない。`true` にすると `apps/mobile/node_modules` 直下にしか symlink されていない peer 依存 (`@expo/metro-runtime`, `@expo/log-box` 等) が transitive に解決できず、Metro が `UnableToResolveError` を投げる。

#### apps/mobile dependencies に Expo Router の peer を明示

`expo-router` の transitive 依存は、pnpm の strict mode では **apps/mobile に明示しないと** `apps/mobile/node_modules` 直下に symlink されない。Metro 階層解決を有効化しても、最低限以下は明示が必要:

```json
{
  "dependencies": {
    "@expo/metro-runtime": "~56.0.8"
  }
}
```

`@expo/metro-runtime` は expo-router の entry が require するが、peer dep 扱いで apps/mobile に明示する必要がある。これを省くと Metro 起動時に `entry-classic.js` の `@expo/metro-runtime` が解決できず即 fail する。バージョンは Expo SDK と合わせる（SDK 56 なら `~56.0.x`）。

#### EAS の monorepo 対応

`eas.json` の `cli.appVersionSource` は `remote` を推奨。`build.production` に `node` バージョンの明示を入れておくと CI が安定:

```json
{
  "build": {
    "production": {
      "node": "20.19.0",
      "autoIncrement": true
    }
  }
}
```

EAS Build 側は `apps/mobile` をワーキングディレクトリとして自動認識するが、ビルド時の `pnpm install` で workspace を解決させるため、ルートに `pnpm-workspace.yaml` がコミットされていることを確認。

#### Turborepo タスクへの組み込み

`turbo.json` の `dev` タスクは `cache: false, persistent: true` のままで OK。`apps/mobile` の `dev` も `expo start` を持続実行するため同じ設定で動く。

`build` タスクは `apps/mobile` では `expo export` に相当する。`apps/mobile/package.json` の `build` スクリプトを `"expo export --output-dir dist"` にしておけば `turbo run build` が正しく動く。ネイティブビルド（IPA/APK 生成）は EAS Build に委譲する想定で、ローカル turbo の対象外。

---

## ルートレベルの共通設定

### 13. Install and Configure oxlint + oxfmt (Root Level)

**`setup-dev:tooling` の「oxlint」「oxfmt」セクションに従う。** モノレポ用（`-w` 付き）のインストールを使用。`migrate-from-biome` / `migrate-from-eslint-prettier` モードの場合は、`setup-dev:tooling` の `references/migration.md` の該当セクションを先に実行する。

### 14. Install and Configure lefthook

**`setup-dev:tooling` の「lefthook」セクションに従う。** モノレポ用（`-w` 付き）のインストールを使用。

### 15. Install and Configure knip

**`setup-dev:tooling` の「knip」セクションに従う。** モノレポ用の `knip.json` を使用。

### 16. Configure worktrunk

**`setup-dev:tooling` の「worktrunk」セクションに従う。**

#### Optional: portless（複数 worktree でのポート衝突対策）

モノレポ + 複数 worktree で `apps/web` と `apps/api` を同時に起動する運用では、ポート衝突が起きやすい。これを避けたい場合は **`setup-dev:portless` スキルに従って portless を設定する。** Vite + Wrangler の連携設定（CORS / プロキシ / `.dev.vars` 自動生成 / worktrunk pre-start フック追加）まで一通りカバーされている。

不要なら省略可。後から導入しても OK。

### 17. Create .gitignore

**`setup-dev:tooling` の「.gitignore（共通ベース）」+「モノレポ向け追加項目」に従う。**

### 18. Set up dotenvx

**`setup-dev:tooling` の「dotenvx」セクションに従う。**

ルートに `.env` を作成し、暗号化する:
```bash
# .env にプロジェクトの環境変数を記述
# CLOUDFLARE_ACCOUNT_ID=...
# CLOUDFLARE_DATABASE_ID=...
# CLOUDFLARE_D1_TOKEN=...

na exec dotenvx encrypt
```

### 19. Configure mise with @antfu/ni

手順 1 で `npm:@antfu/ni` も一緒に固定済みなら、このステップは不要。入れていなければ **`setup-dev:tooling` の「mise」セクション**に従って追加する。

`mise.toml` に `"npm:@antfu/ni" = "<固定バージョン>"` があれば、`mise install` で `ni`/`nr`/`nci`/`nlx`/`na` が使えるようになる。CI でも `jdx/mise-action@v2` が自動でインストールする。

---

## GitHub Actions

### 19-b. (Optional) E2E

bindings 越しの経路（D1 の認可、R2 のファイル、Durable Object のチャット）が
仕様そのものなら、それを通す E2E を置く価値が高い。**`setup-dev:e2e` に従う**
（Cloudflare 向けの節がある）。モノレポ固有の要点だけ:

- **`e2e/` を workspace パッケージにする。** ルート直下の素のディレクトリだと
  turbo の `typecheck` / `lint` から外れ、E2E のコードだけ検査されない
- `webServer` は `vite preview` ではなく **`wrangler dev`**。
  状態は `--persist-to` で開発用と分ける
- `nr test` には含めない。CI も別ジョブにする

### 20. Set up CI Workflow

**`setup-dev:ci` の「CI Workflow」に従う。**

### 21. Set up Deploy Workflow (Optional)

**`setup-dev:ci` の「Cloudflare Deploy Workflow（モノレポ）」に従う。**

`apps/mobile` を含む場合、モバイルのデプロイ（バイナリ生成・配布）は **EAS Workflows 側**で別途定義する。GitHub Actions では `apps/mobile` の `nr typecheck` / `nr lint` / `nr test` / `nr export` は CI でカバーし、バイナリは EAS Build に委譲するのが標準パターン。詳細は `setup-dev:expo` のステップ 17 を参照。

### 22. Set up react-doctor Workflow

**`setup-dev:ci` の「react-doctor Workflow」に従う。**

### 23. Set up Dependabot (Optional)

依存更新を自動化したい場合は **`setup-dev:dependabot` スキルに従って `.github/dependabot.yml` を作成する。** モノレポ向けのテンプレ（`apps/*` / `packages/*` 別 entry、pnpm catalog の制約、Expo SDK 紐づきパッケージの ignore 設定）が用意されている。後から導入しても OK。

---

## CLAUDE.md と README.md

### 24. Create CLAUDE.md

```markdown
# 開発ルール

## コード変更後の検証

コードを変更したら、以下を実行してエラーがないことを確認する:

- `nr lint` (turbo 経由で全パッケージの oxlint)
- `nr format:check` (oxfmt でフォーマット差分の検出)
- `nr typecheck` (turbo 経由で全パッケージの tsc --noEmit)

## 開発サーバー

- `nr dev` で全アプリを同時起動
- Claude Code から実行する場合は `na exec turbo run dev --ui=stream` を使う

## モノレポ構成

セットアップ済みのアプリのみ記載:
- `apps/web` - Vite + React フロントエンド（含めた場合）
- `apps/mobile` - Expo / React Native モバイルアプリ（含めた場合）
- `apps/api` - Hono + Drizzle API サーバー
- `packages/shared` - 共有型・ユーティリティ

## ファイル命名規則

- kebab-case を使用する (例: `my-component.tsx`, `use-auth.ts`)
- PascalCase は使わない

## API アーキテクチャ（レイヤード）

- `routes/` → リクエスト/レスポンスの変換のみ。ロジックは services に委譲
- `services/` → ビジネスロジック。DB や repository を関数引数で受け取る（DI コンテナ不要）
- `repositories/` → Drizzle の DB 操作をラップ
- DI は関数引数で行う（tsyringe 等の DI コンテナは使わない）

## テスト

- TDD（テスト駆動開発）で実装する
- テストを先に書き、実装はテストが通るように行う
- services のテストでは repository をモックして単体テストする

## パッケージ間の依存関係

- `@repo/shared` は `apps/web` / `apps/mobile` / `apps/api` から参照可能
- `@repo/api` の `AppType` は web / mobile 双方から `import type` でのみ参照する

## 推奨 Claude Code スキル

このプロジェクトでは以下のスキルの使用を推奨:
- tanstack-router - ファイルベースルーティング（web 側）
- tanstack-query - サーバー状態管理
- cloudflare - Cloudflare プラットフォーム開発
- wrangler - Cloudflare Workers CLI
- frontend-design - フロントエンド UI 作成
- expo:expo-native-ui - Expo Router でのネイティブ UI 構築（mobile を含む場合）
- expo:expo-data-fetching - データ取得パターン（mobile を含む場合）
- expo:eas-workflows - EAS Workflows（mobile を含む場合）
```

### 25. Create README.md

プロジェクト名と簡単な説明を含む README.md を生成する。以下のセクションを含めること:

#### Getting Started

```bash
# ランタイムのインストール
mise install

# 依存関係のインストール
ni

# D1 データベースの作成（初回のみ）
wrangler d1 create project-name-db
# 出力された database_id を apps/api/wrangler.jsonc に設定

# マイグレーションの生成と適用
na --filter @repo/api run db:generate
na --filter @repo/api run db:migrate:local

# 開発サーバーの起動（API + Web / Mobile を turbo で同時起動）
nr dev

# 個別に起動したい場合
na --filter @repo/web run dev       # Web のみ
na --filter @repo/mobile run dev    # Mobile のみ
na --filter @repo/mobile run ios    # iOS シミュレータ起動（mobile を含む場合）
```

#### Available Scripts

| コマンド | 説明 |
|---|---|
| `nr dev` | 全アプリの開発サーバーを起動（turbo） |
| `nr build` | 全パッケージをビルド |
| `nr lint` | 全パッケージの oxlint チェック |
| `nr typecheck` | 全パッケージの型チェック |
| `nr test` | 全パッケージのテスト実行 |
| `nr format` | oxfmt でフォーマット |
| `nr format:check` | oxfmt でフォーマット差分の検出のみ |
| `nr knip` | 未使用コード・依存関係の検出 |
| `na --filter @repo/api run db:generate` | マイグレーション生成 |
| `na --filter @repo/api run db:migrate:local` | ローカル DB にマイグレーション適用 |
| `na --filter @repo/api run db:studio` | Drizzle Studio を起動 |
| `na --filter @repo/mobile run ios` | iOS シミュレータでビルド（mobile を含む場合） |
| `na --filter @repo/mobile run android` | Android Emulator でビルド（mobile を含む場合） |

#### Project Structure

```
├── apps/
│   ├── api/                 # Hono API サーバー (Cloudflare Workers)
│   │   ├── src/
│   │   │   ├── db/
│   │   │   │   └── schema.ts
│   │   │   └── index.ts
│   │   ├── drizzle/
│   │   │   └── migrations/
│   │   ├── drizzle.config.ts
│   │   └── wrangler.jsonc
│   ├── web/                 # React フロントエンド (Vite) — 含めた場合
│   │   ├── src/
│   │   │   ├── lib/api.ts   # Hono RPC クライアント
│   │   │   ├── routes/
│   │   │   │   ├── __root.tsx
│   │   │   │   └── index.tsx
│   │   │   └── main.tsx
│   │   └── index.html
│   └── mobile/              # Expo / React Native アプリ — 含めた場合
│       ├── src/
│       │   ├── app/         # expo-router ファイルベースルート
│       │   │   ├── _layout.tsx
│       │   │   └── index.tsx
│       │   ├── lib/api.ts   # Hono RPC クライアント
│       │   └── global.css   # Tailwind エントリ
│       ├── tailwind.config.js
│       ├── babel.config.js
│       ├── metro.config.js
│       ├── app.json
│       └── eas.json
├── packages/
│   └── shared/              # 共有型・ユーティリティ
│       └── src/
│           └── index.ts
├── turbo.json
├── .oxlintrc.json
├── lefthook.yml
├── knip.json
└── pnpm-workspace.yaml
```

#### Tech Stack

使用しているライブラリ・ツールの一覧を簡潔に記載する。

---

## Final Steps

### 26. Install Dependencies and Verify

```bash
ni
nr typecheck
nr lint
nr format:check
nr build
nr knip
```

Report the results to the user. `update` / `migrate-*` モードでは、移行前後で `nr lint` と `nr format:check` の差分を一覧化してユーザーに確認すること。

## Hono RPC で踏むもの

### エラーのステータスをリテラル型のままにする

`c.json(body, status)` の `status` を `ContentfulStatusCode` のような広い型にすると、
**クライアントで `res.ok` による絞り込みが効かなくなる**。成功レスポンスとエラーが
union のまま残り、`res.json()` の戻りに `{ error: string }` が混ざる。

```
Type '{ id: string; ... } | { error: string }' is not assignable to type '{ id: string; ... }'
```

エラー写像を関数に切り出すときに起きやすい。**戻り値のリテラル union を保つこと。**

```ts
function toHttpError(e: AppError): { body: ApiError; status: 400 | 404 } { /* ... */ }
```

### zValidator のエラーボディは自前のエラー形式と別物

素の `zValidator` は zod の `SafeParseError` をそのまま返すので、**API のエラー形式が
2 種類**になる（zod 形式と自前の形式）。クライアントが分岐できない。

薄いラッパを作って形を揃える。

```ts
export function validate<T extends ZodType, Target extends keyof ValidationTargets>(
  target: Target, schema: T,
) {
  return zValidator(target, schema, (result, c) => {
    if (!result.success) {
      return c.json({ code: "VALIDATION_FAILED", message: result.error.issues[0]?.message }, 400);
    }
  });
}
```

**業務上の上限をワイヤスキーマに書かないこと。** `title` の最大長やファイルサイズの
上限を zod 側にも書くと、**ドメインに届く前に zValidator が弾き、ドメインが用意した
メッセージがクライアントに出ない**（zod の英語メッセージになる）。
`packages/schema` は形式（uuid か、int か、正か）だけを見て、業務ルールはドメインに置く。
重複も同時に消える。

## R2 に本文を流すときの制約

### `put` は長さが確定したストリームしか受け付けない

「アップロードの実バイト数を数えて上限で打ち切る」を `TransformStream` でやると落ちる。

```
TypeError: Provided readable stream must have a known length
(request/response body or readable half of FixedLengthStream)
```

受け付けるのは**リクエストボディそのもの**か `FixedLengthStream` の readable だけで、
変換を 1 段挟むと長さの情報が消える。

上限の担保は `Content-Length` で行う。**リクエストボディは HTTP の層でその長さに
収まることが保証される**ので、これで実バイト数も抑えられる。長さが無いリクエスト
（chunked）は受け付けない（411）。

```ts
if (declaredLength === undefined) return err({ type: "LengthRequired" });   // 411
if (declaredLength > attachment.size) return err({ type: "ContentTooLarge" }); // 413
await storage.put(key, body, contentType);   // body は変換しない
```

**「アップロード URL 発行時にサイズを検証した」で終わらせないこと。** そこで見ているのは
クライアントの**申告値**であって実際のバイト列ではない。小さい size を申告してから
巨大な本文を PUT すれば上限を迂回できる。

### 親を消しても R2 のオブジェクトは残る

D1 の FK cascade が消すのは**メタデータの行だけ**。R2 のオブジェクトは残り、
参照する行が消えているので**発見も削除もできない孤児**になる。削除のたびに積もる。

集約が分かれていれば 1 トランザクションにはできない（できても D1 と R2 は別ストア）。
**本体 → メタデータの順で消す**。逆順にすると、途中で失敗したときに孤児が残る。

```ts
const attached = await attachments.listByThread(id, ownerId);
await Promise.all(attached.map((a) => storage.delete(a.storageKey)));  // 先に本体
await repo.deleteById(id, ownerId);                                    // 後でメタデータ
```

## 相対 URL を返すか絶対 URL を返すか

`uploadUrl` / `downloadUrl` のようにサーバーが URL を返す API では、**絶対 URL を
サーバー側で組み立てない。**

dev は Vite（:5173）が wrangler（:8787）へプロキシしているので、サーバーが
`c.req.url` を基に絶対 URL を作ると **:8787 を指す**。ブラウザがそこへ直接 PUT すると
クロスオリジンになり、セッション cookie が載らない。**どのオリジンから見えているかは
クライアントしか知らない。**

相対パスを返し、解決はクライアントの責務にする。ブラウザは何もしなくてよく、
Expo / Node は共有クライアント側のヘルパで解決する。

```ts
// packages/api-client
export function resolveApiUrl(baseUrl: string, pathOrUrl: string): string {
  if (/^https?:\/\//.test(pathOrUrl)) return pathOrUrl;
  if (baseUrl.startsWith("/")) return pathOrUrl;   // Web は baseUrl が "/"
  return new URL(pathOrUrl, baseUrl).toString();
}
```

## Workers AI のモデルはツール呼び出しの可否で選ぶ

function calling に対応していないモデルは、**ツール呼び出しを JSON テキストとして
本文にそのまま吐く**。エラーは出ず、ツールが実行されないまま会話が進む。

```
{"type": "function", "name": "readThreadFile", "parameters": {"name": "problem.txt"}}
```

`@cf/meta/llama-3.3-70b-instruct-fp8-fast` で確認済み。モデルを差し替えたら、
**ツールが実際に実行されるところまで**確認すること（応答が返るだけでは足りない）。

## Important Notes

- ユーザー確認なしに既存の設定ファイルを上書きしない（`setup-dev:tooling` の「上書きルール」を参照）
- `update` モードの場合: 既に入っているパッケージや scripts はスキップし、不足分だけ追加する。`oxlint` / `oxfmt` のバージョンが古いだけなら `nu oxlint oxfmt -w` でアップグレード
- `migrate-from-biome` モードの場合: `setup-dev:tooling` の `references/migration.md` の「Biome から oxlint + oxfmt への移行」を最初に実行し、その後でこのスキルの未適用ステップを反映。各 `apps/*` の `package.json` の `lint` script も同時に置換する（`api-setup.md` 参照）
- `migrate-from-eslint-prettier` モードの場合: `setup-dev:tooling` の `references/migration.md` の「ESLint + Prettier から oxlint + oxfmt への移行」を最初に実行。モノレポではルートと各 `apps/*` / `packages/*` のそれぞれに ESLint / Prettier 設定が散在しているケースが多いため、すべて検出・置換する。`.vscode/` のフォーマッタ設定や CI の `eslint`/`prettier` 直接呼び出しも合わせて置換
- プロジェクト固有の設定（カスタムルール、turbo タスクの追加、catalog の独自エントリ等）は**保持する**。新規セットアップで提示する設定はあくまで雛形なので、既存の意図を上書きしない
- D1 データベースの作成は `wrangler d1 create` が必要。database_id をユーザーに確認すること
- `@repo/api` の `AppType` はフロントエンドで型のみ参照する（`import type`）。web / mobile どちらでも同じ扱い
- Hono の RPC 型推論を有効にするため、`app` はメソッドチェーンで定義すること
- `apps/mobile` を含む場合: Expo の Metro は monorepo 向けの追加設定が必要（ステップ 12-b 参照）。`apps/mobile/metro.config.js` の `watchFolders` / `nodeModulesPaths` を忘れずに設定する
- `apps/mobile` を含む場合: `EXPO_PUBLIC_API_URL` で API ベース URL を切り替える。Vite のプロキシ機能に相当するものはモバイルにないため、ローカル / 実機 / ステージング / 本番ごとに `.env.*` で URL を切り替える運用になる
