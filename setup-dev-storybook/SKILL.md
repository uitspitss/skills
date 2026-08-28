---
name: setup-dev:storybook
description: Storybook 10 + addon-vitest をセットアップし、ストーリーを実ブラウザテストとして走らせる。Next.js / Vite SPA / TanStack Start、vitest projects 分割、サーバー専用モジュールのモック、optimizeDeps、CI。他の setup-dev スキルから参照される。
disable-model-invocation: true
---

# Storybook Setup (Storybook 10 + addon-vitest)

**参照スキル:**

- `setup-dev:testing` - Vitest + Testing Library の基本設定。**先にこちらを済ませておくこと**
- `setup-dev:tooling` - oxlint / oxfmt / knip / lefthook
- `setup-dev:ci` - GitHub Actions

このスキルは以下で実地検証済み:

- **Next.js App Router**（pnpm 単一プロジェクト）
- **Vite SPA**（pnpm monorepo）
- **TanStack Start + Bun + Biome**（単一リポジトリ）

Vite SPA 固有の差分は末尾の「Vite SPA の場合」、monorepo 固有の差分は「monorepo の場合」を読む。

### パッケージマネージャ

本文は `@antfu/ni`（`ni` / `nr` / `na` / `nlx` / `nun` / `nu`）で書いてある。これらは
プロジェクトのパッケージマネージャを自動判別するので、**通常は読み替え不要**。
使い分けは `setup-dev:tooling` の「ni / nr / na / nlx の使い分け」を参照。

自動判別が効かない箇所は2つだけ:

| 箇所 | 理由 | どうするか |
|---|---|---|
| `storybook init` の `--package-manager` | init 自身に渡すフラグ。**間違えると別のロックファイルを作りに行く** | 実際の PM 名を書く（`pnpm` / `bun` / `npm` / `yarn`） |
| `na exec` | **bun では効かない**（`bun exec` は `node_modules/.bin` を PATH に載せない） | bun のプロジェクトでは scripts に逃がして `nr` で呼ぶ |

`nu`（update）も bun には対応する意味論が無いので、bun では `ni -D <pkg>@^N.N.N` で
バージョンを直接指定する。

---

## ステップ 0: 対象の確認

```bash
cat package.json | jq -r '.dependencies + .devDependencies | keys[]?' | grep -E '^(next|vite|expo)$'
ls vitest.config.* 2>/dev/null
cat package.json | jq -r '.devDependencies.vitest'
```

| 検出結果 | どうするか |
|---|---|
| `next` がある | このスキル。framework は `@storybook/nextjs-vite` |
| `vite` だけ | 末尾の「Vite SPA の場合」を先に読む。framework は `@storybook/react-vite` |
| workspace / monorepo | 末尾の「monorepo の場合」も読む |
| `expo` / `react-native` | **このスキルは使わない。** `setup-dev:expo` の方針（コンポーネントは実機確認）に従う |
| `vitest` が無い | 先に `setup-dev:testing` |

`@storybook/nextjs-vite` の peer は `next: ^14.1 || ^15 || ^16`、`vite: ^5 || ^6 || ^7 || ^8`。Vitest 4 が持ってくる Vite で足りる。

---

## ステップ 1: init

対話を挟まず、フレームワークとビルダーを明示して流す。`--type` を省くと
コンポーネントライブラリの選択（Base UI / React Aria / Radix）で止まる。

```bash
nlx storybook@latest init --yes --no-dev --type nextjs --builder vite --package-manager pnpm
```

init が入れるもの・触るもの:

| | 内容 |
|---|---|
| 生成 | `.storybook/main.ts`, `.storybook/preview.tsx`, `stories/`（サンプル）, `vitest.shims.d.ts` |
| 書き換え | `vitest.config.ts`（projects 構成に変換）, `package.json`, `.gitignore` |
| 追加 addon | `addon-vitest`, `addon-a11y`, `addon-docs`, `@chromatic-com/storybook`, `addon-mcp` |
| 追加依存 | `@vitest/browser-playwright`, `@vitest/coverage-v8`, `playwright`, `vite` |
| その他 | **Playwright のブラウザバイナリを自動ダウンロードする**（数十秒〜数分かかる） |

最後に「`storybook ai setup` を実行しろ」と出るが、あれは AI エージェント向けの
汎用手順書で、このスキルの内容と重複する。**このスキルがある場合は読まなくてよい。**

### vitest とブラウザ系パッケージのバージョンを揃える

**`@vitest/browser-playwright` / `@vitest/coverage-v8` の peer は `vitest` の
完全固定バージョン**（`"vitest": "4.1.10"` のように範囲ではない）。ここがずれると
以後の add / remove のたびに unmet peer 警告が出続ける。

まず「何に合わせるか」を実物で確認する。推測しない。

```bash
grep -A4 '"peerDependencies"' node_modules/@vitest/browser-playwright/package.json
cat node_modules/vitest/package.json | jq -r .version
```

ずれ方は 2 通りあり、**どちらを動かすかは状況で変わる**:

| 症状 | 直し方 |
|---|---|
| init が `"4.1.0"` の完全固定で入れた／プロジェクトの vitest の方が新しい | ブラウザ系を `^<vitest の実体>` に書き換える |
| init が `"^4.0.18"`（プロジェクトの caret に合わせた形）で入れたが、解決は 4.1.10。**vitest 本体だけ 4.0.18 に据え置かれている** | **`vitest` の方を上げる**（`ni -D vitest@^4.1.10 @vitest/ui@^4.1.10`） |

後者は caret のせいで package.json だけ見ても気づけない。**必ず解決済みバージョン
（`na ls --depth 0`、bun なら `bun pm ls`）と peer の実物を突き合わせる。**
`@vitest/ui` を入れている場合は道連れで上げること。

### 要らない addon を落とす

```bash
nun @chromatic-com/storybook @storybook/addon-mcp
```

- `@chromatic-com/storybook` は Chromatic（有償の視覚回帰サービス）のアカウントが無いと機能しない
- `@storybook/addon-mcp` は AI エージェント向け。要否をユーザーに確認してから決める

残すのは `addon-vitest` / `addon-a11y` / `addon-docs` の 3 つ。
落とした addon は `.storybook/main.ts` の `addons` からも消すこと。

### サンプルを消す

```bash
rm -rf stories
```

`stories/` の雛形（Button/Header/Page）は独自の CSS を持っていて、実プロジェクトの
デザインと無関係な見た目が混ざる。**ストーリーは対象と同じ場所に置く**方針にするので不要。

---

## ステップ 2: `.storybook/main.ts`

init が吐く `getAbsolutePath()` ラッパーは monorepo / Yarn PnP 向け。
pnpm の単一プロジェクトでは要らないので、素の文字列に落として読みやすくする。

```ts
import type { StorybookConfig } from "@storybook/nextjs-vite";

const config: StorybookConfig = {
  // ストーリーは対象と同じ場所に置く（components/foo.stories.tsx）
  stories: ["../components/**/*.stories.tsx", "../app/**/*.stories.tsx"],
  addons: ["@storybook/addon-vitest", "@storybook/addon-a11y", "@storybook/addon-docs"],
  framework: "@storybook/nextjs-vite",
};

export default config;
```

**glob の起点が複数あるとサイドバーの階層が揃わない。** 上の例だと
`components/foo.stories.tsx` は最上位に、`app/settings/forms.stories.tsx` は
`SETTINGS` グループの下に出る。

ストーリー側で `title` を明示すると autotitle を捨てることになる。**`stories` の
オブジェクト形式（`{ directory, files, titlePrefix }`）を使えば autotitle を
維持したままグループにまとめられる**ので、そちらを先に検討する。

```ts
stories: [
  "../src/**/*.stories.tsx",
  { directory: "../../../packages/ui/src", files: "**/*.stories.tsx", titlePrefix: "ui" },
],
```

これで `packages/ui/src/button.stories.tsx` は `ui/button` に入る。

**`stories` を変えたら dev サーバーを再起動する。** main.ts の `stories` は
HMR で反映されず、変更後のパスを開くと "Couldn't find story matching ..." になる。

### shadcn/ui などの vendored なコンポーネントを除外しない

`components/ui/**` を「生成物だから」と外したくなるが、外さないほうがよい。

- ソースを自分たちで持つのが shadcn の前提で、実際に手を入れることになる
- そのプロジェクトの配色でどう出るかは upstream のドキュメントからは分からない
- ローカルで足したバリアントは、そもそも upstream に載っていない

---

## ステップ 3: `.storybook/preview.tsx`

init が作ったものを**編集する**（上書きしない）。足すのは 2 つ。

```tsx
import type { Preview } from "@storybook/nextjs-vite";
import "../app/globals.css";

const preview: Preview = {
  parameters: {
    controls: { matchers: { color: /(background|color)$/i, date: /Date$/i } },
    a11y: { test: "todo" },
  },

  // アプリ本体のコンテナ幅・余白に合わせる。
  // 折り返しが本番と違うとストーリーを見る意味が薄れる
  decorators: [
    (Story) => (
      <div className="mx-auto max-w-[660px] px-6 py-6">
        <Story />
      </div>
    ),
  ],
};

export default preview;
```

**グローバル CSS の import を忘れない。** Tailwind v4 は設定を CSS に持つので、
これが無いとユーティリティが 1 つも効かず、全ストーリーが素の HTML に見える。
PostCSS 設定（`postcss.config.mjs`）はプロジェクトルートのものを Vite が自動で拾う。

### `<link>` で読んでいるフォント

アプリが `app/layout.tsx` の `<head>` で Web フォントを読んでいる場合、
preview には伝わらない。`.storybook/preview-head.html` に同じ `<link>` を置く。

```html
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
<link href="https://fonts.googleapis.com/css2?family=..." rel="stylesheet" />
```

`next/font` を使っているなら不要（`@storybook/nextjs-vite` が面倒を見る）。

**効いているかを `document.fonts.check()` で判定しない。** Google Fonts は
unicode-range でサブセットに分割されており、**1 つでも未ロードのサブセットがあると
`check()` は false を返す**。読み込めているのに false になるので、そのまま
「preview-head.html が効いていない」と誤診する。実際に見るのはこちら:

```js
// preview の iframe の中で
[...document.fonts].filter((f) => f.status === "loaded" && f.family === "M PLUS 1").length
[...document.querySelectorAll("link")].some((l) => l.href.includes("fonts.googleapis.com"))
```

`loaded` が 1 つでもあれば `preview-head.html` は届いている。

---

## ステップ 4: vitest を 2 プロジェクトに分ける

init が `vitest.config.ts` を projects 構成に書き換えるが、整形が崩れるうえ
`setup-dev:testing` で入れた設定が第 1 プロジェクトに押し込まれるだけなので、
意図が読める形に書き直す。

```ts
import { fileURLToPath } from "node:url";
import { storybookTest } from "@storybook/addon-vitest/vitest-plugin";
import { playwright } from "@vitest/browser-playwright";
import { defineConfig } from "vitest/config";

const root = fileURLToPath(new URL(".", import.meta.url));

export default defineConfig({
  resolve: { alias: { "@": root } },
  test: {
    passWithNoTests: true,
    projects: [
      {
        extends: true,
        test: {
          name: "unit",
          environment: "happy-dom",
          // setup-dev:testing で決めた include をそのまま持ってくる
          include: ["{app,components,lib}/**/*.{test,spec}.{ts,tsx}"],
          setupFiles: ["vitest.setup.ts"],
        },
      },
      {
        extends: true,
        plugins: [storybookTest({ configDir: `${root}.storybook` })],
        test: {
          name: "storybook",
          browser: {
            enabled: true,
            headless: true,
            provider: playwright({}),
            instances: [{ browser: "chromium" }],
          },
        },
      },
    ],
  },
});
```

`vitest.setup.ts`（`afterEach(cleanup)`）は **unit プロジェクトにだけ**付ける。
Storybook 側は addon がライフサイクルを持っている。

Storybook 10 では `.storybook/vitest.setup.ts` と `setProjectAnnotations` は不要。
addon が preview を自動で読み込む。

```json
{
  "test": "vitest run",
  "test:unit": "vitest run --project unit",
  "test:storybook": "vitest run --project storybook",
  "test:watch": "vitest --project unit"
}
```

**`test` を `vitest run` に変えたら、既存の呼び出し側も直す。** 元が
`"test": "vitest"`（watch）だったプロジェクトは、CI 側が
`bun run test --run` / `npm test -- --run` のように `--run` を後ろから足している。
そのままだと `vitest run --run` になるので、CI / deploy ワークフローを
`bun run test` に書き換えること。`.github/workflows/*.yml` を全部 grep する。

### `passWithNoTests` がテストの入口を 2 つとも隠す

`setup-dev:testing` に「`passWithNoTests: true` は `include` の書き間違いを隠す」と
あるが、**Storybook を入れると入口がもう 1 つ増える**。`.storybook/main.ts` の
`stories` glob を壊しても、テストが 0 件になるだけで CI は緑になる。

導入直後に `nr test` の「Test Files」と「Tests」の件数を控えておき、
以後はそれを目視の基準にすること。

---

## ステップ 5: サーバー専用モジュールをブラウザから追い出す

ここが一番ハマる。ストーリーはブラウザで動くので、コンポーネントが
サーバー専用モジュール（DB クライアント、認証、Server Actions）を
import していると、その依存ごとブラウザに引きずり込まれて落ちる。

```
ReferenceError: Buffer is not defined
 ❯ node_modules/.../postgres-bytea/index.js
 ❯ node_modules/.../pg/lib/utils.js
```

### `sb.mock()` では足りない

Storybook 公式の `sb.mock(import("../app/settings/actions.ts"))` を preview に
書いても、**コンポーネントから張られた推移的な import は捕まえられず、
実体が評価されてしまう**（`forms.tsx` だけを import する検証用ストーリーで確認済み）。

代わりに Vite の解決を横取りする。

```ts
// .storybook/mock-modules.ts（main.ts に直接書いてもよい。分けるなら下の注意を読む）
import { dirname, isAbsolute, resolve } from "node:path";
import { fileURLToPath } from "node:url";

const here = dirname(fileURLToPath(import.meta.url));
const srcDir = resolve(here, "../src");

// key は拡張子なしの絶対パス、value はモック実体
const moduleMocks: Record<string, string> = {
  [resolve(srcDir, "lib/client")]: resolve(srcDir, "lib/__mocks__/client.js"),
};

function mockModules() {
  return {
    name: "mock-modules",
    enforce: "pre" as const,
    resolveId(source: string, importer: string | undefined) {
      const path = source.split("?")[0] ?? source;

      let target: string | null = null;
      if (path.startsWith("@/")) {
        target = resolve(srcDir, path.slice(2));
      } else if (isAbsolute(path)) {
        target = path;
      } else if (importer && path.startsWith(".")) {
        target = resolve(dirname(importer), path);
      }
      if (!target) return null;

      return moduleMocks[target.replace(/\.(m?[jt]sx?)$/, "")] ?? null;
    },
  };
}

const config: StorybookConfig = {
  // ...
  viteFinal(vite) {
    vite.plugins ??= [];
    vite.plugins.push(mockModules());
    return vite;
  },
};
```

**`viteFinal` だけに登録して済ませない。** `optimizeDeps` / `alias` と同じで、
vitest から走る storybook プロジェクトは `vitest.config.ts` の vite 設定を使う。
モックが効かないまま実 API を叩いても**テストは緑になる**ので、事故に気づけない。
両方に登録する。

```ts
// vitest.config.ts — storybook プロジェクトの plugins に足す
plugins: [mockModules(), storybookTest({ configDir: path.join(root, ".storybook") })],
```

### プラグインを別ファイルに出すなら拡張子を付ける

`.storybook/main.ts` から拡張子なしの相対 import を書くと Storybook が警告を出す。

```
▲  One or more extensionless imports detected: "./mock-modules" in file
│  ".storybook/main.ts".
```

`import { mockModules } from "./mock-modules.ts";` と拡張子を付け、tsconfig に
`allowImportingTsExtensions: true` を足す（`noEmit: true` が前提。このスキルの
対象プロジェクトは満たしている）。**この警告が出るのは main.ts 自身の import だけ**で、
`vitest.config.ts` から `./.storybook/main` を拡張子なしで読む分には出ない。

**相対パスの比較だけでは足りない。** `resolve.alias` を設定していると、Vite の
alias プラグインは `enforce: "pre"` のユーザープラグインより**先**に走る
（`resolvePlugins()` の並びが `aliasPlugin` → `prePlugins`）。つまり `@/lib/client`
は**絶対パスに化けた状態で**このプラグインに届く。相対・エイリアス・絶対の
どれで書かれても同じファイルなら捕まるよう、上のように正規化してから比較する。
拡張子（`.ts` が付いて来ることがある）とクエリ（`?v=` 等）も落とす。

**モック対象はサーバー専用モジュールだけではない。** 次のどれかに当たるモジュールは
ここで差し替える。

| 対象 | 理由 |
|---|---|
| API クライアント（oRPC / tRPC / fetch ラッパ） | 放っておくと実 API を叩き、結果がネットワーク次第になる。差し替えれば Loading / Error / Empty / 正常系を作り分けられる |
| 外部 SDK を `<script>` で注入するコンポーネント（YouTube IFrame API、地図タイル、決済 SDK） | CI ではそもそも取れない。「外部リソースを取りに行かない」の実装手段がこれ |
| **ルーターに依存する Context / hook**（`useSearch` / `useParams` / `useRouter`） | ルーターごと立てないと動かない。hook を差し替えればストーリー側から状態を作れる |

### モックの中身

```js
// app/settings/__mocks__/actions.js
import { fn } from "storybook/test";

export const claimHandle = fn(async () => ({})).mockName("claimHandle");
export const issueToken = fn(async () => ({})).mockName("issueToken");
```

- **実体を `import * as actual from "./actions"` しない。** 一度でも評価すると
  結局サーバー依存を引きずり込む
- モックファイルは **JavaScript の ESM**（TypeScript 不可）
- 型 export（`export type Foo`）は実行時に消えるので並べなくてよい
- **元モジュールに export を足したらモックにも足す**必要がある。
  これは手作業なので、プロジェクトの AGENTS.md に一行書いておくこと
- JSX は使えない（`.js` は Vite の JSX 変換対象外）。
  プレースホルダを描くなら `createElement` を使う
- **プレースホルダに本番のデザイントークンを流用しない。**
  `--muted-foreground` のような控えめな色をそのまま使うと、モックのせいで
  a11y パネルに color-contrast 違反が出る（実例: 黒背景に 4.3:1）。
  **a11y 違反を見たらまず、その要素が自分のモックかどうかを確認する**

ストーリー側では `mocked()` で型を付けて振る舞いを差し込む。

```tsx
import { mocked } from "storybook/test";
import { claimHandle } from "./actions";

export const HandleTaken: Story = {
  async beforeEach() {
    mocked(claimHandle).mockResolvedValue({ error: "@foo はすでに使われています" });
  },
  play: async ({ canvas, userEvent }) => { /* ... */ },
};
```

### ルーター依存の hook を差し替える

関数 export だけでなく、**hook 自体を `fn()` にしておく**と、ストーリーから
`mockReturnValue` で状態を作れる。実モジュールに同名の export があるので
`mocked()` の型も通り、プロダクションコードには一切手を入れずに済む。

```js
// src/contexts/__mocks__/video-player-context.js
import { fn } from "storybook/test";

export const useVideoPlayer = fn(() => ({
  selectedBroadcast: null,
  selectedIndex: undefined,
  openPlayer: fn().mockName("openPlayer"),
  closePlayer: fn().mockName("closePlayer"),
  updateIndex: fn().mockName("updateIndex"),
})).mockName("useVideoPlayer");

// Provider は素通しでよい
export function VideoPlayerProvider({ children }) {
  return children;
}
```

```tsx
const closePlayer = fn().mockName("closePlayer");

export const Open: Story = {
  async beforeEach() {
    closePlayer.mockClear();           // ストーリー間で持ち越さない
    mocked(useVideoPlayer).mockReturnValue({ selectedBroadcast: BROADCAST, closePlayer, /* ... */ });
  },
};
```

**`beforeEach` で毎回 `mockClear()` する。** spy をモジュールスコープに置くと
呼び出し回数がストーリー間で積み上がり、`toHaveBeenCalledTimes(1)` が
**実行順によって落ちたり通ったりする**。

---

## ステップ 6: `optimizeDeps.include` に固定する

ストーリーからしか使わない依存（アイコンライブラリなど）は、テスト実行中に
Vite の dep optimizer が初回バンドルを走らせ、**ページがリロードされて
その時走っていたテストが落ちる**。

```
[vite] dependency optimized: lucide-react
[vite] optimized dependencies changed. reloading
[vitest] Vite unexpectedly reloaded a test.
TypeError: Failed to fetch dynamically imported module: .../@storybook_react-dom-shim.js
```

**アイコンライブラリより先に `react/jsx-dev-runtime` を疑う。** これが遅れて
最適化されると addon の setup file 自体が読めなくなり、**全ストーリーが 0 件で
落ちる**。しかもエラーが原因を指さない:

```
Error: Failed to import test file .../@storybook/addon-vitest/dist/vitest-plugin/setup-file.js
Caused by: Error: Vitest failed to find the runner. One of the following is possible:
- "vitest" is imported directly without running "vitest" command
  ...
```

「`vitest` を直接 import している」という案内は**全部ハズレ**で、実際の原因は
ページリロード。`SyntaxError: ... does not provide an export named 'elementRoles'`
（aria-query）が出ることもあるが、これも同じ根っこ。

先に固定しておく。`react/jsx-dev-runtime` は最低限入れる。

```ts
// .storybook/main.ts — vitest.config.ts からも参照するのでここを唯一の定義にする
export const optimizeDepsInclude = [
  "react",
  "react-dom/client",
  "react/jsx-runtime",
  "react/jsx-dev-runtime",
  "lucide-react",
  // ストーリー / decorator からしか使わないものを足していく
];

// viteFinal 内
vite.optimizeDeps ??= {};
vite.optimizeDeps.include = [...(vite.optimizeDeps.include ?? []), ...optimizeDepsInclude];
```

**`viteFinal` に書くだけでは vitest 側に効かないことがある。** vitest から走る
storybook プロジェクトは `vitest.config.ts` の vite 設定を使うので、そちらにも
同じリストを渡す（上のように main.ts から export して import すれば二重管理にならない）。

```ts
// vitest.config.ts
import { optimizeDepsInclude } from "./.storybook/main";

export default defineConfig({
  optimizeDeps: { include: optimizeDepsInclude },
  // ...
});
```

`rm -rf node_modules/.cache/storybook node_modules/.vite` してから
`nr test:storybook` が通ることを確認すること。ウォームキャッシュでは再現しない。

---

## ステップ 7: ストーリーを書く

```tsx
import type { Meta, StoryObj } from "@storybook/nextjs-vite";
import { expect } from "storybook/test";
import { PathTrail } from "./path-trail";

const meta = {
  component: PathTrail,
  tags: ["ai-generated"],
} satisfies Meta<typeof PathTrail>;

export default meta;
type Story = StoryObj<typeof meta>;

export const Short: Story = { args: { path: ["a", "b"] } };
```

- `canvas` / `userEvent` / `canvasElement` は **play の引数から受け取る**。
  `import { userEvent } from "storybook/test"` や `within(canvasElement)` は書かない
- `expect` / `waitFor` / `mocked` / `screen` は `storybook/test` から import する
- **Portal（Radix の Tooltip / Dialog / Popover）は `canvas` では引けない。**
  `document.body` 直下に出るので `screen.findByRole("tooltip")` を使う
- AI が書いたストーリーには `tags: ["ai-generated"]` を付けておくと、
  後から人がレビューする対象を絞れる

### `satisfies Meta<>` が TS2742 で落ちるとき

`declaration: true`（monorepo の共有パッケージなど）を持つ tsconfig 配下では、
推論された `meta` の型が深い依存の内部パスを参照してしまうことがある。

```
error TS2742: The inferred type of 'meta' cannot be named without a reference to
'../../../node_modules/.pnpm/embla-carousel@8.6.0/.../components/Options'.
```

そのパッケージを直接の依存に足しても**サブパスが exports に無ければ直らない**。
このストーリーだけ `satisfies` をやめて型注釈にする。

```ts
const meta: Meta<typeof Carousel> = { component: Carousel, /* ... */ };
```

args の推論はコンポーネントの props 由来になるので、実用上の損失はほぼ無い。

### `play` はどこに書くか

**引数違いのバリエーションには書かない。** 描画に失敗すればテストは落ちるので、
`getByRole(...).toBeVisible()` だけの play は何も足していない。

書く価値があるのは、描画結果だけでは分からないこと:

- インタラクション（入力 → 送信 → エラーが出る）
- 状態を反映した aria 属性（`aria-invalid`, `aria-expanded`）
- CSS 由来の意味のある状態（アクセントの罫線、無効時の見た目）
- 非同期に出てくるもの（`findBy*`）

### CSS が読み込まれているかの番人を 1 つ置く

`toBeVisible` は素の HTML でも通る。**グローバル CSS が preview に
読み込まれているかを保証できるのは `getComputedStyle` の実測値だけ。**
プロジェクトに 1 つでいいので、代表的なコンポーネントに置く。

```tsx
export const CssCheck: Story = {
  args: { path: ["水出し麦茶"] },
  play: async ({ canvas }) => {
    const node = canvas.getByText("水出し麦茶");
    // border-trail (#A8C4BE)。Tailwind が読み込まれていなければ落ちる
    await expect(getComputedStyle(node).borderColor).toBe("rgb(168, 196, 190)");
  },
};
```

これが落ちたら、他のストーリーの見た目は一切信用できない。

番人にしたい要素にテキストも role も無いとき（スピナーの円弧など）は
`canvasElement.querySelector()` で引く。ここだけは実装詳細に寄って構わない。

```tsx
export const CssCheck: Story = {
  play: async ({ canvasElement }) => {
    const pulse = canvasElement.querySelector(".loading-pulse");
    const style = getComputedStyle(pulse as Element);
    // border-top-color は var(--broadcast-red) = #e8364f
    await expect(style.borderTopColor).toBe("rgb(232, 54, 79)");
    await expect(style.width).toBe("48px");
  },
};
```

### ローカルで変えた vendored コンポーネント

shadcn の `components/ui/**` などに手を入れた箇所は、**ストーリーで固定する**。
upstream のドキュメントに載っていない差分なので、ここでしか記録されない。

```tsx
/** upstream から変えている: bg-transparent ではなく bg-card */
export const Default: Story = {
  play: async ({ canvas }) => {
    await expect(getComputedStyle(canvas.getByRole("textbox")).backgroundColor)
      .toBe("rgb(243, 245, 239)");
  },
};
```

---

## ステップ 8: CI

`setup-dev:ci` の CI Workflow に足す。

```yaml
      # storybook プロジェクトは Chromium 実機で走るのでバイナリを先に入れる
      # monorepo では playwright を持つワークスペースで実行する
      - name: Install Playwright browser
        working-directory: apps/web
        run: na exec playwright install --with-deps chromium

      - run: nr lint
      - run: nr format:check
      - run: nr typecheck
      - run: nr test
      - run: nr build
      - run: nr build-storybook
      - run: nr knip
```

`nr build-storybook` も回す。ストーリーが型は通るのにビルドで落ちる
（動的 import、静的アセット）ケースを拾える。

---

## ステップ 9: ツール設定

**`.gitignore`** — init が追記する。確認するだけでよい。

```
*storybook.log
storybook-static
```

**knip** — `.storybook/**` は knip の Storybook プラグインが自動で entry として扱う。
ストーリーからしか使わない依存（アイコンライブラリ等）が
`ignoreDependencies` に入っていたら、実際に使われるようになった時点で外す。

init はこれを**実依存に変える**ので、既に入っていたら外す:

```
Configuration hints
Unused item in ignoreDependencies: @vitest/coverage-v8
```

`@vitest/coverage-v8` は「設定に文字列 `"v8"` で書くだけ」なので、init 前は
`ignoreDependencies` に逃がしてあることが多い。init が devDependency として
明示的に入れた時点でその行は不要になる。

**oxfmt / oxlint（biome も同じ）** — `.storybook/**` と `*.stories.tsx` は対象に
含める。自分で書くコードなので除外する理由がない。`__mocks__/*.js` も同様。
monorepo では**ストーリーを置いたパッケージに lint スクリプトがあるか**を確認する。
無ければ CI の lint はそのパッケージを素通りする。

**導入直後に format:check が落ちたら、まず既存の崩れかを疑う。** lefthook の
`biome check --write`（や `oxfmt --write`）は staged files を書き換えるが
**index に add し直さない**ので、整形前の内容がそのままコミットされていることがある。
落ちたファイルが `git status` で自分の変更に入っていないなら、それは storybook とは
無関係な既存の崩れ。まとめて直してよいが、**その旨を報告に書く**（storybook 導入の
差分に見えてしまうため）。

**tsconfig** — `.storybook` と `vitest.config.ts` / `vitest.shims.d.ts` を
`include` に足す。init は足してくれないので、設定ファイルの型エラーが
`nr typecheck` をすり抜ける。

```jsonc
"include": ["src", ".storybook", "vite.config.ts", "vitest.config.ts", "vitest.shims.d.ts"]
```

**lefthook** — 追加設定は不要。pre-commit の lint/format が staged files に効く。

---

## 検証チェックリスト

導入したら、この順で確認する。上から順に「効いていない」の切り分けになる。

1. `nr test:unit` — 既存のテスト件数が**減っていない**こと（projects 分割で
   `include` を落としていないか）
2. `rm -rf node_modules/.cache/storybook node_modules/.vite && nr test:storybook` —
   コールドキャッシュで全ストーリーが通ること。**ここを省くと `optimizeDeps` の
   取りこぼしが CI まで残る**（ウォームでは絶対に再現しない）
3. `nr test` — 「Test Files」「Tests」の件数を控える。以後の基準値になる
4. `nr storybook` で起動し、**サイドバーに期待したコンポーネントが全部出ているか**を目視
   （`stories` glob の取りこぼしは件数を見ないと気づけない）
5. 代表的なストーリーを開き、配色とフォントが本番と一致しているか目視。
   フォントは `document.fonts` の `loaded` 件数で確認する（ステップ 3 参照）
6. **a11y パネルの Violations を全ストーリーぶん見る。** 0 にする必要は無いが、
   **モック由来の違反だけはその場で潰す**（ステップ 5）。残ったものは
   プロダクション側の既存違反なので、直さずに報告に回す
7. `nr build-storybook`
8. `nr lint` / `nr format:check` / `nr typecheck` / `nr knip`
9. `nr knip` の出力を**導入前と見比べる**。新規の警告が無いことまで確認する
   （「エラーが無い」ではなく「増えていない」が基準）
10. pre-commit フックを実際に走らせる（`git add -A && git hook run pre-commit`）。
    lint / format / typecheck とは別に knip などを持っていることがある。
    確認したら `git reset` で戻す

ストーリーを書いた後に `stories` glob や `titlePrefix` を変えたら、2 からやり直す。
テスト ID が変わるので、キャッシュが残っていると古い ID で走ることがある。

---

## Vite SPA の場合

- `--type react --builder vite` で init する
- framework は `@storybook/react-vite`、preview / main の型 import も同じパッケージから

### 既存 `vite.config.ts` は当てにしない

`@storybook/react-vite` は dev / build ではプロジェクトの `vite.config.ts` を
自動で読むが、**vitest から走る storybook プロジェクトでは読まれない**。
使うのは `vitest.config.ts` の vite 設定なので、`resolve.alias` をそちらに
書いていないとこうなる:

```
Error: The following dependencies are imported but could not be resolved:
  @/lib/query-key (imported by .../src/components/place-image.tsx)
  @/lib/utils (imported by .../src/components/place-image.tsx)
```

しかもこの解決失敗は dep のスキャンごと巻き添えにするので、続けて
`optimizeDeps` 側のエラー（`does not provide an export named 'elementRoles'` 等）
が出て原因が二重に見えなくなる。**alias は `vitest.config.ts` と
`.storybook/main.ts` の両方に明示する。**

```ts
// vitest.config.ts（projects は extends: true で継承する）
resolve: { alias: { "@": fileURLToPath(new URL("./src", import.meta.url)) } },
```

```ts
// .storybook/main.ts の viteFinal — 配列形式で来ることがあるので分岐する
vite.resolve ??= {};
vite.resolve.alias = Array.isArray(vite.resolve.alias)
  ? [...vite.resolve.alias, { find: "@", replacement: srcDir }]
  : { ...vite.resolve.alias, "@": srcDir };
```

`mergeConfig(viteConfig, ...)` で丸ごと合成する手もあるが、`vite.config.ts` に
副作用（`execFileSync` でポートを引く等）が入っているプロジェクトでは避ける。

### Tailwind v4 はプラグインを `viteFinal` に足さないと効かない

`vite.config.ts` が読まれないのは alias だけの問題ではない。**Tailwind v4 は
`@tailwindcss/vite` プラグインが CSS を生成する**ので、これが無いと
`@import "tailwindcss"` が素通しになる。

症状は「素の HTML だが表示はされる」で、**エラーが一切出ない**。ストーリーは全部
通るのにスタイルだけが無い状態になり、そういう見た目だと思い込んで進んでしまう。

```ts
viteFinal: (vite) => {
  // storybook dev / build では vite.config.ts が読まれるので二重登録を避ける
  const plugins = (vite.plugins ??= []);
  const hasTailwind = plugins
    .flat(Number.POSITIVE_INFINITY)
    .some((p) => p && typeof p === "object" && "name" in p && String(p.name).includes("tailwind"));
  if (!hasTailwind) plugins.push(tailwindcss());
  return vite;
}
```

`preview.tsx` で CSS を import するだけでは足りない（import しても中身が空になる）。
ステップ 7 の「CSS が読み込まれているかの番人」がここで効く。

### メタフレームワークのプラグインは先回りで剥がさない

dev / build で `vite.config.ts` が読まれるということは、**TanStack Start の
`tanstackStart()` のようなプラグインも一緒に読まれる**。壊れそうに見えるが、
`viteFinal` で `plugins` を間引く必要は無かった（TanStack Start v1.159 +
`@storybook/react-vite` 10.5 で `storybook dev` / `build-storybook` とも成功）。

**まず素で通してみて、落ちてから対処する。** 先回りで剥がすと、剥がしたせいで
別の解決が壊れて原因が二重になる。

### `viteFinal` は SPA でも要る

サーバー専用モジュールが無くても、次の 2 つで必要になる:

- **`base` の打ち消し** — アプリが `base: "/docs/"` などで配信されている場合、
  Storybook は自身のルートから配信するので `vite.base = "/"` に戻す
- **API クライアントのモック**（ステップ 5）と **`optimizeDeps`**（ステップ 6）

### `<html class="dark">` などのルート属性

Vite SPA はテーマクラスを `index.html` の `<html>` に直書きしていることが多い。
preview の iframe には伝わらないので、`preview.tsx` の先頭で付ける。

```tsx
document.documentElement.classList.add("dark");
```

`preview-head.html` は `<head>` の中身しか入れられないので、ここでは使えない。

**`<body>` 側のクラスも同じ。** ルートレイアウトが `<body className="...">` で
背景ノイズやスキャンラインのような装飾を当てている場合、それも preview には来ない。

```tsx
// アプリ本体は body に付けている（src/routes/__root.tsx）
document.body.classList.add("noise-overlay", "scan-line");
```

Storybook 自身も `sb-show-main` 等を body に足すが、`classList.add` なら共存する。

### Provider が要るコンポーネント

React Query などを使うコンポーネントは decorator で包む。**ストーリーごとに
client を作り直さないとキャッシュが隣のストーリーへ漏れる。**

```tsx
const withQueryClient: Decorator = (Story) => {
  const [queryClient] = useState(
    () => new QueryClient({ defaultOptions: { queries: { retry: false } } }),
  );
  return <QueryClientProvider client={queryClient}><Story /></QueryClientProvider>;
};
```

`retry: false` を入れないと、エラー系ストーリーが既定のリトライ待ちでタイムアウトする。

`setup-dev:vite-react` と併用するときは、そちらの `vitest.config.ts` を
projects 構成に置き換える形になる。

---

## monorepo の場合

**Storybook はアプリ側に 1 つだけ置く。** 共有 UI パッケージのストーリーは
`stories` のオブジェクト形式（ステップ 2）で `titlePrefix` を付けて拾う。
パッケージごとに `.storybook` を立てると設定と CI が丸ごと二重になる。

```ts
stories: [
  "../src/**/*.stories.tsx",
  { directory: "../../../packages/ui/src", files: "**/*.stories.tsx", titlePrefix: "ui" },
],
```

そのうえで、ストーリーを置くパッケージ側に手当てが要る。

| 対象 | やること | 理由 |
|---|---|---|
| `packages/*/package.json` | `@storybook/react-vite` と `storybook` を devDependency に足す | そのパッケージ単体の `tsc --noEmit` が `Meta` / `storybook/test` を解決できない（pnpm は非ホイスト） |
| `packages/*/package.json` | `lint` スクリプトが無ければ足す | turbo の `lint` はパッケージ単位。無いとそのパッケージのストーリーが CI の lint 対象から漏れる |
| `turbo.json` | `storybook`（`cache: false` / `persistent: true`）と `build-storybook`（`outputs: ["storybook-static/**"]`）を足す | ルートから `nr storybook` / `nr build-storybook` で回せるようにする |
| ルート `package.json` | `"storybook": "turbo run storybook"` / `"build-storybook": "turbo run build-storybook"` | 同上 |

`knip` は Storybook プラグインが `.storybook/**` を entry として扱うので、
workspace の `project` を `src/**/*` に絞っていても追加設定は要らなかった。

---

## やらないこと

- **サンプルの `stories/` を残さない。** 実プロジェクトと無関係な CSS が混ざる
- **`play` を全ストーリーに付けない。** 冗長なだけで、壊れたときの情報量は増えない
- **ストーリーのためにプロダクションコードの形を変えない。**
  Server Actions を prop で受け取るように変える、といった改変は本末転倒。
  モックで解決する（ステップ 5）
- **a11y の `test: "todo"` を安易に `"error"` にしない。** 既存プロジェクトでは
  たいてい既存のコントラスト違反を拾って CI が落ちる。まず違反を洗い出し、
  直す・許容するを決めてから上げること
- **既存のバグをストーリーで固定しない。** 書いている途中でプロダクション側の
  バグ（分岐順の誤りでローディングが出ない等）に気づくことがある。現状の挙動を
  assert すると誤りが仕様として残る。そのストーリーは書かずに報告だけして、
  直すかどうかは別で判断してもらう
- **ストーリーで外部リソースを取りに行かない。** 画像は data URI（インライン SVG）
  にする。ネットワーク次第でテストが揺れるうえ、CI では取れないこともある。
  外部 SDK を読むコンポーネント（YouTube / 地図タイル / 決済）はモックで止める。
  **止められないものはストーリーにしない**（例: 地図タイルを直に引く地図本体）
- **モック由来の a11y 違反をプロダクションの問題として報告しない。** 逆に、
  プロダクション側の違反を**黙って直さない**。どちらも、まず「どっちの由来か」を
  a11y パネルの要素で確定させてから動く
- **`storybook dev` を起動したまま設定をいじって「効いていない」と判断しない。**
  `stories` / `framework` / `viteFinal` は再起動が要る
