---
name: setup-dev:tsconfig
description: TypeScript tsconfig.json configuration conventions. Defines base compiler options (target, module, moduleResolution, strict) aligned with Node.js version. Referenced by other setup-dev skills.
disable-model-invocation: true
---

# TypeScript Configuration Conventions

## Base compilerOptions

すべてのプロジェクト/パッケージで共通の基本設定:

- `target`: Node.js version に合わせる (Node 24+ → `ES2024`, Node 22 → `ES2024`, Node 20 → `ES2023`, Node 18 → `ES2022`)
- `module`: `ESNext`
- `moduleResolution`: `Bundler`
- `strict`: `true`
- `skipLibCheck`: `true`
- `esModuleInterop`: `true`

## ビルド成果物を出すパッケージのみの追加設定

**ライブラリ / 他パッケージから import される package にだけ付ける。** アプリケーション（Next.js, Vite, Expo など、バンドラが出力を持つもの）には付けない:

- `declaration`: `true`
- `outDir`: `dist`
- `rootDir`: `src`

アプリ側は `noEmit: true` にして、出力はバンドラに任せる。アプリに `outDir` / `declaration` を付けると、バンドラの出力と別に `dist/` が生えたり、`noEmit` と衝突して警告になる。

## React フロントエンド向け追加設定

- `jsx`: `react-jsx`
- `types`: `["vite/client"]`（`import.meta.env` の型サポートに必要）

**Vite 以外のフレームワークでは上記2つをそのまま持ち込まない。** 例えば Next.js は `jsx: "preserve"`（変換は SWC が行う）で、`vite/client` の型は無関係。フレームワーク公式のテンプレが生成した設定を尊重し、`target` のような素の TypeScript 設定だけを揃える。

## `include` に設定ファイルも入れる

`include` を `["src"]` だけにすると、**リポジトリ直下の設定ファイルが型検査から
漏れる**。`vite.config.ts` / `vitest.config.ts` / `.storybook/**` / `*.d.ts` は
自分で書く TypeScript なので、`nr typecheck` の対象に入れる。

```jsonc
"include": ["src", ".storybook", "vite.config.ts", "vitest.config.ts", "vitest.shims.d.ts"]
```

ツールの init（`storybook init` など）は `include` を更新してくれない。設定ファイルを
足したら `include` も見直すこと。エラーが出ないまま型検査をすり抜けるので気づけない。

## `declaration: true` は推論型の可搬性まで要求する

ビルド成果物を出すパッケージ（上記の `declaration: true`）では、**推論された型が
外から名前で参照できないと TS2742 で落ちる**。pnpm の非ホイスト構成では
依存の内部パスがそもそも参照できないため、深い型を推論させたときに起きる。

```
error TS2742: The inferred type of 'meta' cannot be named without a reference to
'../../../node_modules/.pnpm/embla-carousel@8.6.0/.../components/Options'.
```

そのパッケージを直接依存に足しても、**サブパスが `exports` に無ければ直らない**。
`satisfies` をやめて型注釈を書く（推論させない）のが確実な回避。

```ts
const meta: Meta<typeof Carousel> = { /* ... */ };  // satisfies Meta<...> だと落ちる
```

## 既存プロジェクトの tsconfig を触るときの原則

`plugins`（Next.js の `{ "name": "next" }` など）、`include` に並んだ生成物のパス（`.next/types/**/*.ts`、`routeTree.gen.ts`）、`paths` のエイリアスは**フレームワークの動作に必要**なので消さない。`exclude` に生成物ディレクトリを足すのも危険（`include` で明示的に取り込んでいる型定義まで落ちる）。

## Cloudflare Workers 向け追加設定

- `types`: `["@cloudflare/workers-types"]`
- Hono の JSX を使う場合:
  - `jsx`: `react-jsx`
  - `jsxImportSource`: `hono/jsx`

## モノレポの shared パッケージ向け追加設定

- `composite`: `true`（プロジェクト参照に必要）

## ソース消費パッケージに `composite` / `references` を付けない

`exports` が `src` を直接指す共有パッケージ（バンドラが TypeScript のまま取り込む形）に
`composite: true` を付け、消費側に `references` を書くと、**ビルド成果物を要求されて落ちる**。

```
error TS6305: Output file '.../packages/schema/dist/index.d.ts' has not been built
from source file '.../packages/schema/src/index.ts'.
```

このパッケージは `dist` を出さないので、いくら待っても解決しない。
`composite` / `declaration` / `outDir` / `rootDir` を外して `noEmit: true` にし、
消費側の `references` も消す。**参照は `paths` だけで足りる。**

```jsonc
// packages/schema/tsconfig.json
{
  "compilerOptions": {
    // exports が src を直接指す「ソース消費」パッケージなので dist は出さない
    "noEmit": true
  },
  "include": ["src"]
}
```

`references` が要るのは「ビルド成果物を出すパッケージ」だけ（上の節を参照）。

## モノレポのパス参照設定

他パッケージを参照する場合、`paths` と `references` を設定:

```json
{
  "compilerOptions": {
    "paths": {
      "@repo/shared": ["../../packages/shared/src"]
    }
  },
  "references": [{ "path": "../../packages/shared" }]
}
```
