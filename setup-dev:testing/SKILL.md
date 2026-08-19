---
name: setup-dev:testing
description: Vitest and Testing Library setup for React projects. Configures vitest with happy-dom, @testing-library/react, and @testing-library/jest-dom. Referenced by other setup-dev skills.
disable-model-invocation: true
---

# Testing Setup (Vitest + Testing Library)

## React フロントエンド向け（full setup）

### インストール

```bash
ni -D vitest happy-dom @testing-library/react @testing-library/jest-dom
```

### vitest.config.ts

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "happy-dom",
    // ソースディレクトリは必ず実プロジェクトに合わせる（下の注意を読むこと）
    include: ["src/**/*.{test,spec}.{ts,tsx}"],
    setupFiles: ["vitest.setup.ts"],
    passWithNoTests: true,
  },
});
```

**`include` は必ず実際のソース配置に合わせて書き換える。** `src/` を持たない構成（Next.js App Router の `app/`、`components/`、`lib/` 直下など）でこの雛形をそのまま使うと1件もマッチしない。`passWithNoTests: true` と組み合わさると、**テストが一度も走っていないのに `nr test` も CI も緑になる**という最悪の失敗をする。

```ts
// 例: src/ を持たない構成
include: ["{app,components,lib}/**/*.{test,spec}.{ts,tsx}"],
```

導入直後に**必ずテストを1つ書いて実際に走ることを確認する**こと。「Test Files 0 passed」で通っていたら include が効いていない。

**そのときの「Test Files」「Tests」の件数を控えておく。** 以後、テストを増減させて
いないのに件数が減っていたら `include` が壊れている。エラーは出ないので、
件数を覚えていない限り気づけない。

### `vitest.config.ts` は `vite.config.ts` を継承しない

Vite プロジェクトでも、`vitest.config.ts` を別ファイルで持つと **`vite.config.ts` の
`resolve.alias` / `plugins` は一切引き継がれない**。`@/...` を使っているとこうなる:

```
Error: The following dependencies are imported but could not be resolved:
  @/lib/utils (imported by .../src/components/foo.tsx)
```

alias を両方に書くか、`mergeConfig` で合成する。

```ts
// vitest.config.ts
import { fileURLToPath } from "node:url";

export default defineConfig({
  resolve: { alias: { "@": fileURLToPath(new URL("./src", import.meta.url)) } },
  test: { /* ... */ },
});
```

`vite.config.ts` に副作用（起動時の `execFileSync` など）があるプロジェクトでは
`mergeConfig` を避けて、alias だけ二重に書くほうが安全。

### 入口が増えることを忘れない

`setup-dev:storybook` を入れると `vitest.config.ts` は projects 構成に書き換わり、
**テストの入口が `include` glob と `.storybook/main.ts` の `stories` glob の2つになる**。
どちらを壊しても「0件で緑」になる。上で控えた件数はそのときの基準値としても使う。

### E2E はこのスキルの対象外

サーバーを起動して確かめるものは `setup-dev:e2e`（`@playwright/test`）に置く。
**Vitest browser mode で E2E を書かないこと。** ブラウザは動くがサーバーは動かない。

なお `include` を `**/*.{test,spec}.ts` のように広く書いていると、**Playwright の既定の
テストパターンと `.spec.ts` を取り合う**。Vitest が Playwright のテストを拾って
`Playwright Test needs to be invoked via 'npx playwright test'` で落ちる。
E2E を入れる予定があるなら、`include` は最初からディレクトリを明示しておく。

### tsconfig の `jsx` が `"preserve"` の場合

Vitest 4 は rolldown / oxc ベースになっており、**`esbuild:` オプションは効かない**。tsconfig の `jsx` が `"preserve"`（Next.js など、フレームワーク側が JSX を変換する構成）だと JSX がそのまま残り、パースエラーで落ちる:

```
Expected ">" but found "path"
```

トランスフォーマ側で明示する:

```ts
export default defineConfig({
  oxc: { jsx: { runtime: "automatic", importSource: "react" } },
  test: { /* ... */ },
});
```

tsconfig が `jsx: "react-jsx"`（Vite の標準構成）ならこの指定は不要。

### vitest.setup.ts

```ts
import "@testing-library/jest-dom/vitest";
import { cleanup } from "@testing-library/react";
import { afterEach } from "vitest";

// Testing Library の自動 cleanup は globals: true のときしか登録されない。
// globals を使わず vitest の API を明示 import する方針なので、ここで手動登録する。
afterEach(cleanup);
```

**`afterEach(cleanup)` を省略しない。** 上記の `vitest.config.ts` は `globals: true` を設定していないため、Testing Library の自動 cleanup が登録されない。省略すると前のテストの DOM が残り、**1ファイル内で複数回 `render()` した瞬間に落ちる**:

```
TestingLibraryElementError: Found multiple elements with the text: ...
```

`globals: true` を使う方針なら自動 cleanup が働くのでこの3行は不要になるが、その場合は `describe` / `it` / `expect` の import を全テストから外す運用に統一すること。

---

## API / バックエンド向け（minimal setup）

### インストール

```bash
ni -D vitest
```

### vitest.config.ts

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    include: ["src/**/*.{test,spec}.ts"],
    passWithNoTests: true,
  },
});
```

Testing Library や happy-dom は不要（DOM テストがないため）。

`include` は実際のソース配置に合わせること（full setup 側の注意と同じ）。

`passWithNoTests: true` は、テストファイルがまだ無い段階で CI の `nr test` を落とさないために必要（vitest はデフォルトで「No test files found」で exit 1 する）。実テストが揃った後も残しておいて問題ない。

**ただしこのフラグは `include` の書き間違いを隠す。** 導入時とテスト追加時に、`nr test` の出力の「Test Files」が期待どおりの件数になっているかを目視すること。
