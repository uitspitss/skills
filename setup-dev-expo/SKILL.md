---
name: setup-dev:expo
description: Expo / React Native の開発環境をセットアップ・更新する。expo-router、NativeWind v4（Tailwind v3）、TanStack Query、Hono RPC、oxlint、oxfmt、Vitest、lefthook、EAS。Biome / ESLint+Prettier からの移行にも対応。
disable-model-invocation: true
---

# Expo / React Native Development Environment Setup

新規 Expo プロジェクトのセットアップに加え、既存プロジェクトのツール更新と Biome / ESLint+Prettier から oxlint+oxfmt への移行にも対応する。`$ARGUMENTS` が指定されていればそのディレクトリ、なければカレントディレクトリで動作する。

**最初に `setup-dev:tooling` の「モード判定」セクションに従い、`fresh` / `migrate-from-biome` / `migrate-from-eslint-prettier` / `update` のどれで動作するかを決定すること。** 以降の各ステップは新規セットアップ前提で書かれているが、`update` / `migrate-*` モードでは「既に設定済みかチェック → 不足分のみ追加 / 必要な置換のみ実施」の方針で進める。

**参照スキル:** このスキルは以下の共通スキルを参照する。該当ステップでは共通スキルの内容に従うこと。

- `setup-dev:tsconfig` - TypeScript 設定の共通方針
- `setup-dev:tooling` - oxlint, oxfmt, lefthook, knip, worktrunk, dotenvx の設定（モード判定を含む）
- `setup-dev:testing` - Vitest の設定（モバイルでは下記の差分を適用）
- `setup-dev:ci` - GitHub Actions の設定
- `setup-dev:dependabot` - 依存更新自動化（オプション）

## Why Expo は他フレームワークと違うのか

Expo は Metro バンドラと React Native ランタイムを前提とするため、Vite/Node 系プロジェクトと噛み合わない箇所がある。以下を踏まえてセットアップする:

- **Babel が必須**: NativeWind と Reanimated は Babel プラグインで動作する。Vite/SWC ではなく Babel に設定を入れる
- **Metro 設定が必須**: NativeWind v4 は Metro transformer を差し込む必要がある
- **Vitest はユーティリティ層のみ**: RN コンポーネントテストは Metro 前提の API（`Stylesheet`, ネイティブモジュール）に依存するため、Vitest で動かすには `react-native-web` 経由のブリッジが必要。テストはまず純粋な hooks/utilities 中心で書き、コンポーネントテストは「必要になってから」入れる
- **mise の `node` のみ管理**: iOS シミュレータ・Android Emulator は Xcode / Android Studio 側で別管理
- **expo install を優先**: バージョン互換性を Expo SDK が解決するため、`expo install` を使う。`ni` でも入るが SDK 整合性は保証されない

## Prerequisites Check

Before starting, verify that the following tools are installed:
- `mise` (runtime version manager)
- `node` (Node.js runtime)
- `ni` (@antfu/ni - package manager agent)

iOS シミュレータを使う場合（推奨フロー）:
- **Xcode** がインストール済み（`xcode-select -p` で確認）
- **CocoaPods** がインストール済み（`which pod` で確認、無ければ `brew install cocoapods`）。`expo run:ios` の Pod install で必須
- **iOS Simulator runtime** がインストール済み。**Expo SDK の preview/next を使う場合は最新の iOS runtime が必要** で、未インストールだと `xcodebuild` が "iOS X.Y is not installed" で失敗する。Xcode > Settings > Components （または Platforms タブ）から該当 iOS バージョンの Simulator runtime を取得する（数 GB、5-30 分）

Android Emulator を使う場合:
- **Android Studio** + 該当 SDK Platform + AVD（仮想デバイス）
- `which adb` で adb が利用可能か確認

ネイティブビルドが不要なら Expo Go アプリ（実機）または Web 出力でも開発可能。ユーザーにどの形態で開発するか確認する。**ただし Expo SDK の preview/next channel は Expo Go では動かない**（Expo Go アプリは安定 SDK 専用）ので、preview/next を使うなら Development Build (`expo run:ios/android`) が必須。

## Setup Steps

### 1. Create Expo Project

create-expo-app の `default@next` テンプレートを使う。`--no-install` でロックファイル生成を抑止し、後で pnpm に統一する:

```bash
nlx create-expo-app@latest <dir> --template default@next --no-install
cd <dir>
```

**`default@next` を選ぶ理由**: SDK 最新世代向けの「全部入り」テンプレで、`expo-router`、`experiments.typedRoutes`、`experiments.reactCompiler`、`src/app/` 構造、TypeScript 設定 が **最初から全部 ON**。`blank-typescript` を選ぶと最小スキャフォールド (App.tsx 1 枚) になり、これらを全部手動で設定する手間が増える。Welcome 画面やサンプル UI は後で消せるが、Expo の最新スタイルの初期セットを手に入れる方が学習コストとしても安い。

**update / migrate モードの場合**: このステップはスキップ。既存の `app.json` または `app.config.ts` の存在で「Expo プロジェクトである」ことを確認してから、次のステップ以降に進む。

#### `default@next` テンプレが生成するもの（SDK 56 next 時点）

create-expo-app の最新テンプレートは「最小スキャフォールド」ではなく、**かなり豊富な初期セット**を生成する:

- `src/app/_layout.tsx` / `src/app/index.tsx` / `src/app/explore.tsx`（tab レイアウト）
- `src/components/` 配下に多数のサンプル UI（themed-text, themed-view, animated-icon, app-tabs, ui/collapsible 等）
- `src/global.css`（CSS variables のみ。Tailwind ディレクティブはまだ無い）
- `app.json` の `plugins` に `expo-router` 同梱、`experiments.typedRoutes: true` と `experiments.reactCompiler: true` が既に ON
- `CLAUDE.md` と `AGENTS.md`（Expo SDK の versioned docs を読むよう指示）が同梱

これらは「ベース」として尊重し、**全部消して書き直さない**。後続のステップ（NativeWind 設定、TanStack Query 追加など）は **既存生成物に対する差分** として適用する。具体的には:

- `app/` 配下を新規作成するステップは、SDK 56 next では `src/app/` が既にあるので**スキップ**して内容追記に切り替える
- `global.css` も既存ファイルに Tailwind ディレクティブを追記する形にする
- `_layout.tsx` は既存内容（`AppTabs` を使う tab レイアウト）を尊重しつつ、`QueryClientProvider` と `GestureHandlerRootView` でラップする

**Welcome 画面を最初から消したい場合**: テンプレに同梱されている `npm run reset-project` (`scripts/reset-project.js`) を走らせると Welcome 関連の components/画面が `src/app-example/` に退避され、最小の `src/app/index.tsx` だけが残る。

### 2. Configure mise

**`setup-dev:tooling` の「mise（ランタイムのバージョン管理）」セクションに従う。** `node` / `pnpm` / `npm:@antfu/ni` を固定バージョンで `mise.toml` に書き、`mise install` する。

**Expo では Node バージョンを勝手に上げない。** Expo SDK は動作確認済みの Node バージョンを持っているので、`node@lts` の解決結果がそれより新しい場合は SDK 側の要求に合わせて固定する（`nlx expo-doctor` が不整合を検出する）。

`package.json` に `"packageManager": "pnpm@<version>"` フィールドを追加して、ロックファイル方言を pnpm に固定する。

### 3. Install Expo Required Packages

```bash
na exec expo install expo-router expo-image expo-image-picker \
  react-native-gesture-handler react-native-reanimated \
  react-native-safe-area-context react-native-screens
```

**バージョン指定なしで `expo install` を使う**: Expo SDK が各パッケージの互換バージョンを管理しているため、`expo install` に解決を任せるのが最も堅牢。**手動で `~17.0.x` のような旧バージョン体系を書かない**。Expo SDK 56 以降は `expo-*` パッケージのバージョンが **SDK と同期した命名 (`~56.0.x`)** に揃ったため、過去ドキュメントや AI が出力する `~14.x` `~17.x` のような旧番号は SDK と非互換になる。

**`na exec` を使う理由**: `nlx`（= `pnpm dlx`）は registry から expo CLI を一時取得してしまい遠回り。`na exec` なら local の `node_modules/.bin/expo` を使う。

#### モード判定との関係

- **fresh モード**: 上記コマンドをそのまま実行
- **update モード**: `na exec expo install --check` で SDK との不整合を検出し、`--fix` で自動修正。手動でバージョンを書かない
- create-expo-app の `default@next` テンプレートには `expo-router` 等が同梱されている場合がある。重複インストールは Expo 側で吸収されるので問題ない

#### pnpm 利用時: Expo Router の peer 依存を明示

`expo-router` の entry は `@expo/metro-runtime` を require するが、**pnpm の strict mode では `apps/<name>/node_modules` 直下に symlink されないと Metro が解決できない**。default@next テンプレに含まれるかどうかに関わらず、明示的に追加する:

```bash
na exec expo install @expo/metro-runtime
```

`expo install` 経由で入れることで SDK 整合バージョン (`~<sdk-major>.0.x`) が自動選択される（SDK 56 なら `~56.0.x`）。これを省くと dev server 起動時に `UnableToResolveError` で bundle が即落ちる（npm/yarn では hoist で偶然解決されるが pnpm では失敗する罠）。

### 4. Configure app.json (expo-router, bundle identifiers, scheme)

**注**: `default@next` テンプレでは `app.json` の `plugins` に `expo-router` 同梱、`experiments.typedRoutes` も既に ON。`src/app/` 配下にもサンプルルートが生成済み。一方、**bundle identifier / package は未設定** なので必ず追加する。下記の構造を `app.json` にマージする:

```json
{
  "expo": {
    "scheme": "my-app",
    "ios": {
      "bundleIdentifier": "com.<owner>.<appslug>"
    },
    "android": {
      "package": "com.<owner>.<appslug>"
    },
    "plugins": ["expo-router"],
    "experiments": {
      "typedRoutes": true,
      "reactCompiler": true
    }
  }
}
```

#### `ios.bundleIdentifier` / `android.package` を必ず設定する

`expo run:ios` / `expo run:android` を実行する前に必須。未設定だと `CommandError: Required property "ios bundleIdentifier" is not found in the project app.json` で即落ちる。

**選び方の規約**:
- **Reverse-DNS 形式**（`com.<owner>.<appslug>`）
- iOS: ASCII 英数字 + ハイフン + ドット OK
- Android: ハイフン **不可**、各セグメント英字始まり必須
- **両 OS で同じ値を共有する** のが運用的に最も楽 → ハイフン避け、camelCase か単語結合
- 例: `com.example.myapp`
- **後から変えると面倒**（既存ビルドが別アプリ扱い、TestFlight / Play Store の app record と紐づく）。**最初に決めて固定**

#### `scheme` と `typedRoutes`

- `scheme`: deep link スキーム（`my-app://path` で開ける）。bundle identifier と独立に設定
- `typedRoutes: true` で `expo-router` がルートの型を自動生成する（TanStack Router の `routeTree.gen.ts` に相当）
- `reactCompiler: true` で React Compiler 有効化（SDK 53+ 推奨）

**ファイルベースルーティングの初期構造**（`default@next` テンプレが生成済み。以下は最小形の参考）:

```
src/app/
  _layout.tsx        # ルートレイアウト
  index.tsx          # "/" 画面
```

`src/app/_layout.tsx`:
```tsx
import { Stack } from "expo-router";
import { GestureHandlerRootView } from "react-native-gesture-handler";

export default function RootLayout() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <Stack />
    </GestureHandlerRootView>
  );
}
```

`src/app/index.tsx`:
```tsx
import { Text, View } from "react-native";

export default function Home() {
  return (
    <View className="flex-1 items-center justify-center bg-white">
      <Text className="text-xl">Hello Expo</Text>
    </View>
  );
}
```

**なぜ `GestureHandlerRootView` を最上位に**: `react-native-gesture-handler` はネイティブビューを Bridge 経由で挿入するため、ルートに置かないとジェスチャーが動かない箇所が出る。expo-router の `Stack` を直接 default export する書き方もあるが、明示する方が将来の拡張で躓きにくい。

### 5. Install and Configure NativeWind v4

```bash
ni nativewind "tailwindcss@^3.4" react-native-css-interop
```

**Tailwind は v3 系を使う**: 2026 年 8 月時点で、`nativewind@4.x`（latest 4.2.6）は **Tailwind v3 のみ対応**。`nativewind` 自身の peer は `tailwindcss: >3.3.0` と緩いが、ランタイム依存の `react-native-css-interop`（0.2.6）の peer が `tailwindcss: ~3` で v4 を弾く。`tailwindcss@4` を入れると Metro 起動時に「NativeWind only supports Tailwind CSS v3」で失敗する。

Tailwind v4 対応は `nativewind@5` 側で進んでいるが、**まだ `preview` タグ（5.0.0-preview.x）で latest ではない**。`npm view nativewind dist-tags` で `latest` が 5.x になったら、その時点で両方を上げる。

```bash
na exec tailwindcss init
```

`tailwind.config.js` を編集:
```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ["./src/**/*.{ts,tsx}"],   // src/app も src/components もこれで拾える
  presets: [require("nativewind/preset")],
  theme: { extend: {} },
  plugins: [],
};
```

`src/global.css`（テンプレが生成済み。CSS variables のみ）の**先頭に追記**する:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

`src/app/_layout.tsx` の先頭で `import "../global.css";`（`src/app/` から見て `src/global.css`）。

`babel.config.js` を作成（存在しなければ）:
```js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: [
      ["babel-preset-expo", { jsxImportSource: "nativewind" }],
      "nativewind/babel",
    ],
    plugins: ["react-native-worklets/plugin"],
  };
};
```

**順序が重要**: worklets plugin は `plugins` 配列の**最後**に置く必要がある。worklet を含む関数を Babel が AST 変換するため、他のプラグインの後で走らせる。これを守らないと `worklet` 系のコード（Reanimated の `useAnimatedStyle` 等）が動かない。

**`react-native-worklets/plugin` vs `react-native-reanimated/plugin`**: Reanimated 4 以降は worklet 処理が独立した `react-native-worklets` パッケージに分離された（Expo SDK 54+）。**SDK 54 以前** のテンプレでは `react-native-reanimated/plugin` を使う。プロジェクトの reanimated バージョンを確認し、4.x 以降なら worklets plugin、3.x 以前なら reanimated plugin。

`metro.config.js` を作成（存在しなければ）:
```js
const { getDefaultConfig } = require("expo/metro-config");
const { withNativeWind } = require("nativewind/metro");

const config = getDefaultConfig(__dirname);
module.exports = withNativeWind(config, { input: "./src/global.css" });
```

**`input` はテンプレの実配置に合わせる。** `default@next` は `src/global.css` を生成する。ここを間違えると Tailwind のクラスが一切効かないが、エラーは出ない。

`nativewind-env.d.ts` を作成（型補完用）:
```ts
/// <reference types="nativewind/types" />
```

`tsconfig.json` の `include` に `nativewind-env.d.ts` が含まれることを確認。

**CSS / CSS module の type-only 宣言を追加**: SDK 56 next の create-expo-app テンプレは `import "../global.css"` のような side-effect import や `*.module.css` の named import を使う。`tsc --noEmit` でこれらが「型なし」と判定されてエラーになるため、`types.d.ts` を作成:

```ts
declare module "*.css";

declare module "*.module.css" {
  const classes: { [key: string]: string };
  export default classes;
}
```

`tsconfig.json` の `include` に `types.d.ts` を追加する。

**なぜ NativeWind v4 + Tailwind v3 を選ぶか**: NativeWind は Tailwind のクラス名を RN の style オブジェクトに変換する層で、`react-native-css-interop` が Tailwind のビルド出力形式に強く依存する。したがって「Tailwind の最新版」ではなく「NativeWind が対応している Tailwind」に合わせるのが唯一の正解になる。v4 の CSS-first 設定（`@theme` ディレクティブ）を使いたくても、NativeWind 5 が latest になるまでは選べない。

### 6. Install and Configure TanStack Query

```bash
ni @tanstack/react-query zod
```

`src/app/_layout.tsx` に `QueryClientProvider` を追加:
```tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { Stack } from "expo-router";
import { GestureHandlerRootView } from "react-native-gesture-handler";
import "../global.css";

const queryClient = new QueryClient();

export default function RootLayout() {
  return (
    <QueryClientProvider client={queryClient}>
      <GestureHandlerRootView style={{ flex: 1 }}>
        <Stack />
      </GestureHandlerRootView>
    </QueryClientProvider>
  );
}
```

### 7. Set up Hono RPC Client

API（別プロジェクトまたはモノレポ内 `@repo/api`）と型安全に通信するため、Hono RPC クライアントを作る:

```bash
ni hono
```

`src/lib/api.ts`:
```ts
import type { AppType } from "@repo/api"; // モノレポなら workspace、単体プロジェクトなら型定義を別途持ち込む
import { hc } from "hono/client";

const baseUrl = process.env.EXPO_PUBLIC_API_URL ?? "http://localhost:8787";

export const api = hc<AppType>(baseUrl);
```

**なぜ `EXPO_PUBLIC_API_URL` を使うか**: Expo は `EXPO_PUBLIC_*` プレフィックスが付いた環境変数だけクライアントバンドルに含める。これを使えば dev/staging/production で同じコードのまま差し替えられる。

**実機で `localhost` が解決できない問題**: iOS シミュレータは localhost を解決できるが、Android Emulator は `10.0.2.2`、実機は LAN IP が必要。`.env.local` で `EXPO_PUBLIC_API_URL` を環境ごとに切り替える。

`@repo/api` をモノレポで持っていない場合は、AppType を別ファイルで暫定定義するか、Hono RPC のジェネリック型を `any` で諦めて REST 呼び出しのみにする選択もある。

### 8. Install and Configure TypeScript

create-expo-app の `default@next` には TypeScript 設定が同梱されている。差分として以下を反映する。

`tsconfig.json` — **`setup-dev:tsconfig` の「Base compilerOptions」+「React フロントエンド向け追加設定」をベースとし、Expo 固有の差分を以下のように適用:**

- `extends`: `"expo/tsconfig.base"`（Expo が提供する基底を使う）
- `types` から `"vite/client"` を除外（Expo は Vite 非依存）
- `include` に `"nativewind-env.d.ts"`, `".expo/types/**/*.ts"`, `"expo-env.d.ts"` を追加

```json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "jsx": "react-jsx",
    "moduleResolution": "Bundler",
    "paths": { "@/*": ["./src/*"] }
  },
  "include": [
    "**/*.ts",
    "**/*.tsx",
    ".expo/types/**/*.ts",
    "expo-env.d.ts",
    "nativewind-env.d.ts"
  ]
}
```

**なぜ `expo/tsconfig.base` を extends するか**: Expo SDK は内部で `react-native` の型シムや `metro-config` のエイリアスを設定している。これを自力で再現すると SDK 更新ごとに追従が必要になるため、基底を継承するのが安全。

### 9. Install and Configure oxlint + oxfmt

**`setup-dev:tooling` の「oxlint」「oxfmt」セクションに従う。** 単一プロジェクト用（`-w` なし）でインストール。

**Expo 固有の差分:**
- `.oxlintrc.json` の `plugins` から `"import"` を一旦外すか、`ignorePatterns` に `.expo/`, `android/`, `ios/` を追加する。Expo のネイティブ生成物が import 解決を阻害するため
- create-expo-app が同梱する `eslint-config-expo` は削除する。oxlint と二重で走るとフォーマット衝突が起きやすい

`migrate-from-biome` / `migrate-from-eslint-prettier` モードの場合は、`setup-dev:tooling` の `references/migration.md` の該当セクションを先に実行する。create-expo-app の `eslint.config.js` は migrate-from-eslint-prettier の対象。

### 10. Install and Configure Vitest (Utilities and Logic Only)

**`setup-dev:testing` の「React フロントエンド向け（full setup）」をベースに、Expo 固有の差分を適用する。**

#### 何をテストするか

Expo プロジェクトでは Vitest は **ユーティリティ層とビジネスロジックのみ** をカバーする:

- 純粋関数、フォーマッタ、バリデータ
- `@tanstack/react-query` の `queryFn`、Hono クライアントのラッパー
- カスタム hooks（`react-native` の API を直接触らないもの）

RN コンポーネント（`<View>`, `<Text>` 等）のテストは、Metro と React Native のネイティブモジュールに依存するため Vitest では正しく動かない。これは **意図的な切り分け**: コンポーネントは Storybook / Expo Go の実機確認で見るか、本当に必要になってから `jest-expo` を併設する。

#### vitest.config.ts

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "happy-dom",
    include: ["src/**/*.test.ts", "src/**/*.test.tsx"],
    setupFiles: ["vitest.setup.ts"],
    server: {
      deps: {
        // RN モジュールが誤って解決されないようにする
        inline: [/^react-native/],
      },
    },
  },
  resolve: {
    alias: {
      // RN コンポーネントを Web 用に解決させたい場合のみ有効化
      // "react-native": "react-native-web",
    },
  },
});
```

`vitest.setup.ts`:
```ts
import "@testing-library/jest-dom/vitest";
```

**RN コンポーネントテストを後から入れたい場合**: `react-native-web` をエイリアスする方法（上記コメント部分を有効化 + `@testing-library/react` で書く）、もしくは `jest-expo` を併設する方法のいずれかを選ぶ。スキルは前者のヒントを残し、後者は専用設定が必要なので別途検討。

### 11. Install and Configure knip

**`setup-dev:tooling` の「knip」セクションに従う。** ただし以下を `knip.json` に追加して、Expo の特殊エントリを誤検出させない:

```json
{
  "entry": ["src/app/**/*.{ts,tsx}", "metro.config.js", "babel.config.js", "tailwind.config.js"],
  "ignore": [".expo/**", "android/**", "ios/**"]
}
```

`src/app/` 配下は expo-router がエントリとして使う。明示しないと「未参照ファイル」として誤検出される。

### 12. Install and Configure lefthook

**`setup-dev:tooling` の「lefthook」セクションに従う。**

### 13. Configure worktrunk

**`setup-dev:tooling` の「worktrunk」セクションに従う。**

#### Optional: portless

Expo の Metro dev server は既定で `8081` を使う。複数 worktree で並行起動するとここで衝突する。`PORTLESS=0` で逃げる手もあるが、安定運用したい場合は **`setup-dev:portless` スキルに従って portless を設定し、Metro を portless 配下で起動する。**

`apps/mobile/package.json`（モノレポ時）または `package.json` の `dev` スクリプト例:
```json
{
  "scripts": {
    "dev": "portless run --name mobile -- sh -c 'expo start --port $PORT'"
  }
}
```

`sh -c` 経由で `$PORT` を渡すのは wrangler と同じ理由（Expo CLI は `PORT` 環境変数を直接読まないことがある）。

実機 / シミュレータから接続する場合は portless の `.localhost` URL を解決できる必要があるので、portless 不要なら省略可。

### 14. Directory Structure

```
src/
  app/                # expo-router のファイルベースルーティング
    _layout.tsx
    index.tsx
  components/         # 共通コンポーネント
  lib/
    api.ts            # Hono RPC クライアント
  global.css          # Tailwind エントリ
assets/               # 画像・フォント等
tailwind.config.js
babel.config.js
metro.config.js
nativewind-env.d.ts
types.d.ts
app.json (または app.config.ts)
```

`default@next` テンプレが生成するのがこの `src/` 配下に寄せた形。`src/app/` と `src/` の他ディレクトリを分けるのは、expo-router の自動ルートと純粋な実装コードを区別するため。

**ルート直下の `app/` にしない。** expo-router は `app/` と `src/app/` の両方を認識するので、両方あるとどちらが使われているか分からなくなる。テンプレに合わせて `src/app/` 一本にする。

### 15. Add npm Scripts to package.json

`default@next` テンプレが生成する scripts は **`"ios": "expo start --ios"`**（既存 dev client への attach モード）だが、これは **初回の native build を含まない** ため初学者には罠になる。`nr ios` で「ネイティブビルド + インストール + 起動」が走るほうが期待値に近いので、上書きする:

```json
{
  "scripts": {
    "start": "expo start",
    "dev": "expo start",
    "ios": "expo run:ios",
    "android": "expo run:android",
    "ios:start": "expo start --ios",
    "android:start": "expo start --android",
    "web": "expo start --web",
    "prebuild": "expo prebuild",
    "export": "expo export",
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

| script | 内容 | いつ使う |
|---|---|---|
| `nr ios` (= `expo run:ios`) | prebuild → pod install → Xcode build → install → 起動 | **初回 / native 設定 (app.json plugins, native dep 追加) 変更後** |
| `nr ios:start` (= `expo start --ios`) | dev server を起動し、既存 dev client を立ち上げる | 2 回目以降の日常開発（rebuild 不要時） |
| `nr dev` (= `expo start`) | interactive dev server。`i` キーで iOS、`a` で Android、`w` で Web | 2 回目以降の日常開発（最も便利） |

**初回フローの厳守**: `app.json` の bundle identifier や plugins を変更したら、必ず `nr ios` で **rebuild + install**。`nr ios:start` は app.json 変更を反映できないので、初回および native 設定変更後にこれを使うと "No development build for this project is installed" エラーになる。

**`build` ではなく `export`**: Expo の本番アセットは `expo export` で生成する。ネイティブビルドは EAS Build または `expo run:ios/android` 側の管轄。

### 16. Create .gitignore

**`setup-dev:tooling` の「.gitignore（共通ベース）」に従う。** Expo 固有の追加項目:

```
# Expo
.expo/
.expo-shared/
dist/
web-build/

# Native build artifacts
ios/Pods/
ios/build/
android/build/
android/app/build/
android/.gradle/
*.jks
*.p8
*.p12
*.key
*.mobileprovision
*.orig.*

# Metro cache
.metro-health-check*

# EAS local artifacts
*.apk
*.aab
*.ipa
```

`ios/` と `android/` のディレクトリ全体をどう扱うかは方針次第:
- **CNG (Continuous Native Generation) 方式**（推奨）: `expo prebuild` で毎回生成するので両方とも `.gitignore` 入りにする
- **永続化方式**: 既存のネイティブコードをカスタムする場合はコミットする

新規セットアップでは CNG を前提とし、`ios/` と `android/` を `.gitignore` に入れる。

### 17. Configure EAS

```bash
nlx eas-cli@latest init
```

`eas.json` の雛形（`eas init` の生成物に近い）:
```json
{
  "cli": { "version": ">= 13.0.0", "appVersionSource": "remote" },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": { "distribution": "internal" },
    "production": { "autoIncrement": true }
  },
  "submit": { "production": {} }
}
```

`.eas/workflows/build.yml` の雛形（GitHub Actions ではなく EAS Workflows）:
```yaml
name: Build
on:
  push:
    branches: [main]
jobs:
  build_ios:
    type: build
    params:
      platform: ios
      profile: preview
  build_android:
    type: build
    params:
      platform: android
      profile: preview
```

**EAS Workflows と GitHub Actions の使い分け**: EAS Workflows は Expo クラウド側で実行され、シミュレータ/実機向けバイナリ生成に最適化されている。型チェック・lint・テストといった汎用 CI は GitHub Actions（次のステップ）で回す。両者を並行させるのが標準パターン。

詳しい EAS Workflow の書き方は **`expo:eas-workflows` スキル**を参照する。

### 18. Set up GitHub Actions

**`setup-dev:ci` の「CI Workflow」と「react-doctor Workflow」に従う。**

Expo 固有の差分: CI ジョブで `nr build` の代わりに `nr export` を実行する（ネイティブビルドは EAS 側）:

```yaml
# .github/workflows/ci.yml の build ステップ
- run: nr export
```

`build` をスクリプトとして残す場合は `"build": "expo export"` のエイリアスにしても良い。

#### Optional: Dependabot

依存更新を自動化したい場合は **`setup-dev:dependabot` スキルに従って `.github/dependabot.yml` を作成する。** Expo SDK 紐づきパッケージ（`expo-*`, `react-native-*`）の ignore 設定を含むテンプレが用意されている。SDK 互換を保つため、これらを Dependabot に勝手に更新させない設定が重要。

### 19. Configure mise with @antfu/ni

ステップ 2 で実施済み。再確認のみ。

### 20. Create AGENTS.md

開発ルールの実体は `AGENTS.md` に書き、`CLAUDE.md` はそれを参照するだけにする（Claude 以外のエージェントからも同じルールを読めるようにするため）。テンプレート同梱の `CLAUDE.md` / `AGENTS.md` がある場合は、内容を `AGENTS.md` に集約してから `CLAUDE.md` を差し替える。

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
- Vitest はユーティリティ層とロジックを対象とする
- RN コンポーネントの見た目は実機 / Expo Go / シミュレータで確認

## ファイル命名規則

- kebab-case を使用する (例: `my-component.tsx`, `use-auth.ts`)
- expo-router の `app/` 配下はルート名がそのまま URL になるため、用途に合わせた名前を付ける（例: `src/app/settings/profile.tsx`）

## API 通信

- `process.env.EXPO_PUBLIC_API_URL` で API ベース URL を切り替える
- Hono RPC クライアントは `src/lib/api.ts` で集約
- `@repo/api` の `AppType` は `import type` でのみ読み込む（実装はバンドルに含めない）

## ネイティブビルドと CNG

- `ios/` `android/` は `.gitignore`。必要なら `expo prebuild` で再生成
- ネイティブ設定の変更は `app.json` の `plugins` または `expo-build-properties` で表現する

## 推奨 Claude Code スキル

このプロジェクトでは以下のスキルの使用を推奨:
- expo:expo-native-ui - Expo Router でのネイティブ UI 構築
- expo:expo-data-fetching - データ取得パターン
- expo:eas-workflows - EAS Workflows
- tanstack-query - サーバー状態管理
- frontend-design - UI 作成
```

### 21. Create README.md

プロジェクト名と簡単な説明を含む README.md を生成する。以下のセクションを含めること:

#### Getting Started

```bash
# ランタイムのインストール
mise install

# 依存関係のインストール
ni

# 開発サーバーの起動（Expo Go / シミュレータ）
nr dev

# iOS シミュレータで起動
nr ios

# Android Emulator で起動
nr android
```

#### Available Scripts

| コマンド | 説明 |
|---|---|
| `nr dev` | Metro dev server を起動 |
| `nr ios` | iOS シミュレータでビルド・起動 |
| `nr android` | Android Emulator でビルド・起動 |
| `nr web` | Web 出力を起動 |
| `nr export` | プロダクション用アセットを出力 |
| `nr prebuild` | ネイティブコードを生成 |
| `nr lint` | oxlint でコードチェック |
| `nr lint:fix` | oxlint で自動修正 |
| `nr format` | oxfmt でフォーマット |
| `nr format:check` | oxfmt でフォーマット差分の検出のみ |
| `nr test` | テストを実行 |
| `nr typecheck` | TypeScript の型チェック |
| `nr knip` | 未使用コード・依存関係の検出 |

#### Project Structure

```
src/app/              # expo-router ファイルベースルート
src/lib/api.ts        # Hono RPC クライアント
src/global.css        # Tailwind エントリ
tailwind.config.js
babel.config.js
metro.config.js
app.json              # Expo 設定
eas.json              # EAS 設定
```

#### Tech Stack

使用しているライブラリ・ツールの一覧を簡潔に記載する。

### 22. Final Verification

Run the following to verify everything is working:
```bash
nr typecheck
nr lint
nr format:check
nr test
nr export
nr knip
```

`nr export` はネイティブビルドを必要としないので CI でも回せる。実機 / シミュレータでの動作確認は手動で行う。

Report the results to the user。`update` / `migrate-*` モードでは、移行前後で `nr lint` / `nr format:check` の差分を一覧化してユーザーに確認すること。

## Important Notes

- ユーザー確認なしに既存の設定ファイルを上書きしない（`setup-dev:tooling` の「上書きルール」を参照）
- `update` モードの場合: 既に入っているパッケージや scripts はスキップし、不足分だけ追加する。Expo SDK のメジャー更新が絡む場合は **`expo:expo-upgrade` スキル**を参照
- `migrate-from-biome` モードの場合: `setup-dev:tooling` の `references/migration.md` の「Biome から oxlint + oxfmt への移行」を最初に実行し、その後でこのスキルの未適用ステップを反映
- `migrate-from-eslint-prettier` モードの場合: create-expo-app が同梱する `eslint.config.js` と `eslint-config-expo` を `setup-dev:tooling` の手順で oxlint に置換する。VSCode のフォーマッタ設定や CI の `eslint`/`prettier` 直接呼び出しも合わせて置換
- Reanimated の Babel プラグインは `babel.config.js` の `plugins` 配列の**最後**に置く。順序が違うと worklet が機能しない
- NativeWind v4 に対して Tailwind は **v3 系**を使う（`nativewind@5` が latest になるまで v4 は選べない。ステップ 5 参照）
- **pnpm を使う場合 (単独 / monorepo どちらも)**: `metro.config.js` の `resolver.disableHierarchicalLookup` は **`false`** にする。Expo 公式 monorepo ガイドは `true` を推奨するが、これは npm/yarn 想定で、pnpm の `.pnpm/<pkg>/node_modules/<dep>` 階層解決を阻害する。`true` のままだと dev server 起動時に `UnableToResolveError` ( `@expo/metro-runtime` / `@expo/log-box` 等の expo-router 依存) で bundle が失敗する
- **pnpm を使う場合**: `expo-router` の peer 依存である `@expo/metro-runtime` を `apps/<name>/package.json` の dependencies に明示する (`~<sdk-major>.0.x`)。明示しないと pnpm の strict mode で symlink されず Metro が見つけられない
- `EXPO_PUBLIC_*` 環境変数だけがクライアントバンドルに含まれる。シークレットを誤って `EXPO_PUBLIC_` プレフィックスで定義しない
- ネイティブビルドは EAS Build に委譲する想定。ローカルで `expo run:ios/android` する場合は Xcode / Android Studio のセットアップが別途必要
- プロジェクト固有の設定（カスタム `app.config.ts`、`plugins`、独自エイリアス）は**保持する**。新規セットアップで提示する設定はあくまで雛形なので、既存の意図を上書きしない
