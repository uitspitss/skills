---
name: setup-dev:ci
description: GitHub Actions CI workflow, react-doctor, and Cloudflare deploy workflow templates. Referenced by other setup-dev skills.
disable-model-invocation: true
---

# GitHub Actions Workflows

このスキルは CI / Deploy / react-doctor の workflow テンプレを扱う。**依存更新自動化（Dependabot）は別スキル `setup-dev:dependabot` に分離してある** ので、依存更新を設定したい場合はそちらを参照すること。

## CI Workflow

`.github/workflows/ci.yml`:

**前提**: `mise.toml` に `"npm:@antfu/ni"` が登録されていること。`jdx/mise-action@v2` が `ni`/`nr`/`nci` を自動インストールする。

```yaml
name: CI
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: jdx/mise-action@v2
      - run: nci
      - run: nr lint
      - run: nr format:check
      - run: nr typecheck
      - run: nr test
      - run: nr build
      - run: nr knip
      # similarity-ts を導入した場合のみ（デフォルトでは入らない。下記参照）
      - run: nr similarity
```

**`nr similarity` は similarity-ts を導入したときだけ入れる**: similarity-ts は npm パッケージではなく crates.io の Rust CLI で、`setup-dev:tooling` では**ユーザー確認のうえで導入する扱い**にしている。導入していないのにこのステップを残すと CI が「コマンドが無い」で落ちる。`package.json` に `similarity` スクリプトが無いなら、このステップも消すこと。

**`nr similarity` を末尾に置く理由**: 重複コード検出は失敗しても build/test の結果には影響しないが、解析時間がやや長い。ビルドと型検査が通っていない状態で走らせても意味が薄いため、他チェックがすべて通った後に実行する。`setup-dev:tooling` の「similarity-ts」セクションも参照。

### ビルド / 型検査に前提がある場合

テンプレをそのまま貼ると落ちる典型が2つある。導入時にクリーンなチェックアウト相当（`node_modules` と生成物を消した状態）で一度確認すること。

**1. build に環境変数が要る場合**: DB クライアントをモジュールトップレベルで初期化していたり、SSG / ISR がビルド時にデータを取りに行く構成では、`nr build` に本番同等の環境変数が要る。

```yaml
      - run: nr build
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

**fork からの pull_request では secrets が渡らない**（空文字になる）。外部コントリビュータを受け入れるリポジトリでは、build ステップを `push` イベントに限定するか、ビルド専用のダミー値を使えるようコードを変える。

**2. typecheck に生成物が要る場合**: Next.js の `next-env.d.ts` / `.next/types/` のように **gitignore される生成物**を tsconfig が include していると、クリーンなチェックアウトで型検査が落ちる。生成コマンドを `typecheck` スクリプトの前段に入れる（例: `"typecheck": "next typegen && tsc --noEmit"`）。

TanStack Router の `routeTree.gen.ts` は `setup-dev:vite-react` では**コミットする**方針なのでこの問題は起きない。gitignore する運用に倒すなら、上と同じく `"typecheck": "tsr generate && tsc --noEmit"` のように前段を足すこと。`setup-dev:tooling` の TypeScript の項も参照。

### ブラウザ実機で走るテストがある場合

Vitest browser mode（`setup-dev:storybook` の addon-vitest など）を使っていると、
`nr test` は Playwright の Chromium バイナリを要求する。**`nci` は入れてくれない**ので
専用ステップが要る。

```yaml
      - run: nci
      # ブラウザ実機で走るテストがあるのでバイナリを先に入れる
      - name: Install Playwright browser
        # モノレポでは playwright を持つワークスペースで実行する
        working-directory: apps/web
        run: na exec playwright install --with-deps chromium
      - run: nr lint
```

`--with-deps` は Ubuntu ランナー上で共有ライブラリまで入れる。省くと
起動時に `error while loading shared libraries` で落ちる。

Storybook を入れているなら `nr build-storybook` も `nr build` の後に足す。
ストーリーが型は通るのにビルドで落ちる（動的 import、静的アセット）ケースを拾える。

### E2E がある場合

**この `ci` ジョブに足さず、独立したジョブにする。** E2E はサーバーの起動と DB が要るので、
同居させると lint の失敗を待ってから動くことになり、フィードバックが遅くなる。
ジョブの雛形（`services` の Postgres、`playwright install`、`nr build` → `nr test:e2e`、
レポートの artifact 化）は `setup-dev:e2e` のステップ6にある。

---

## react-doctor Workflow

`.github/workflows/react-doctor.yml`:

```yaml
name: React Doctor
on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  pull-requests: write
  # commit-status の publish に要る。無いと warning を出してスキップされる
  statuses: write

jobs:
  react-doctor:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          # 既定の fetch-depth: 1 だと base に到達できず、scope: changed が
          # 「変更ファイル内の全指摘」に退化する（下記）
          fetch-depth: 0
      - uses: millionco/react-doctor@v2
        continue-on-error: true
        with:
          # 単一プロジェクトなら省略可（既定は "."）
          directory: apps/web
          # 正しいキーは blocking。fail-on という input は存在せず、書いても
          # Unexpected input(s) の warning が出て黙って無視される
          blocking: none
```

### `fetch-depth: 0` を必ず付ける

既定の `scope: changed` は「その PR が持ち込んだ指摘だけ」を報告する。base との
比較が要るので、**浅いチェックアウトだと base に到達できず、`files` 相当
（変更ファイル内の既存の指摘も全部）に退化する。**

アクション自身が `git fetch --depth=1 origin $BASE_SHA` を best-effort で試みるが、
失敗すると GitHub API にフォールバックしてこの warning を出す:

```
React Doctor could not derive the PR's changed files from git
(the base commit isn't reachable — set `fetch-depth: 0` on actions/checkout
for deep histories). Falling back to the GitHub API.
```

`silence-missing-baseline-warning` で黙らせられるが、それは症状を隠すだけ。
アクションの input 説明自身が「**fixing the checkout with `fetch-depth: 0` is the
real remedy**」と書いている。

### `fail-on` という input は存在しない

正しいキーは **`blocking`**（`none` / `warning` / `error`）。`fail-on` を書くと

```
##[warning]Unexpected input(s) 'fail-on', valid inputs are
['directory', 'project', 'scope', 'blocking', 'comment', ...]
```

が出て**黙って無視される**。`blocking` の既定が `none` なので一見意図どおり動くが、
既定が変わった瞬間に PR をブロックし始める。**「動いている」ことを根拠にしない。**

### `directory` は指定してよい

`directory` は既定 `"."` の正式な input で、モノレポでサブディレクトリを
指定しても問題なく動く（`apps/web` で実績あり）。ルートをスキャンさせると
`.claude/skills/` や `.agents/` に置いたテンプレートまで検査対象になるので、
**モノレポではむしろ指定すべき**。

### バージョンは移動しないタグに固定する

`millionco/react-doctor@main` は移動するブランチを指している。`v2` のような
メジャータグに寄せる。`actions/checkout` は 2026-08 時点で `v7` が最新
（`v4` からの差分は Node 24 化と fork PR の checkout 制限で、通常の
`pull_request` ワークフローには影響しない）。

---

## Cloudflare Deploy Workflow（単一プロジェクト）

`.github/workflows/deploy.yml`:

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: jdx/mise-action@v2
      - run: nci
      - run: nr typecheck
      - run: nr lint
      - run: nr test
      - run: nr build
      - uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

---

## Cloudflare Deploy Workflow（モノレポ）

`.github/workflows/deploy.yml`:

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy-api:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: jdx/mise-action@v2
      - run: nci
      - run: nr typecheck
      - run: nr test
      - uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          workingDirectory: apps/api

  deploy-web:
    runs-on: ubuntu-latest
    needs: deploy-api
    steps:
      - uses: actions/checkout@v7
      - uses: jdx/mise-action@v2
      - run: nci
      - run: na exec turbo run build --filter=@repo/web
      # Cloudflare Pages or other hosting deploy step here
```
