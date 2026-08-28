---
name: setup-dev:tooling
description: 共通の開発ツールをセットアップする。mise、pnpm、TypeScript、oxlint、oxfmt、lefthook、knip、similarity-ts、worktrunk、dotenvx。新規 / update / Biome・ESLint+Prettier からの移行のモード判定を含む。他の setup-dev スキルから参照される。
disable-model-invocation: true
---

# Development Tooling Setup

このスキルは新規セットアップに加え、既存プロジェクトのアップデートと Biome / ESLint+Prettier からの移行にも対応する。冒頭の「モード判定」を必ず最初に実行し、適切なモードで各セクションを適用すること。

---

## モード判定（最初に実行する）

### 検出ロジック

カレントディレクトリのファイルとパッケージ依存を確認し、4 つのモードを判定する:

```bash
# Biome の検出
ls biome.json biome.jsonc 2>/dev/null
cat package.json 2>/dev/null | jq -r '.devDependencies + .dependencies | keys[]?' 2>/dev/null | grep -E '^@biomejs/biome$'

# ESLint の検出
ls .eslintrc .eslintrc.{js,cjs,mjs,json,yml,yaml} eslint.config.{js,cjs,mjs,ts} 2>/dev/null
cat package.json 2>/dev/null | jq -r '.devDependencies + .dependencies | keys[]?' 2>/dev/null | grep -E '^(eslint|@typescript-eslint/.*|eslint-.*)$'

# Prettier の検出
ls .prettierrc .prettierrc.{js,cjs,mjs,json,yml,yaml,toml} prettier.config.{js,cjs,mjs,ts} 2>/dev/null
cat package.json 2>/dev/null | jq -r '.devDependencies + .dependencies | keys[]?' 2>/dev/null | grep -E '^(prettier|@trivago/prettier-plugin-.*|prettier-plugin-.*)$'

# oxlint / oxfmt の検出
ls .oxlintrc.json .oxlintrc 2>/dev/null
cat package.json 2>/dev/null | jq -r '.devDependencies + .dependencies | keys[]?' 2>/dev/null | grep -E '^(oxlint|oxfmt)$'
```

| 検出結果 | モード |
|---|---|
| `package.json` がない、または devDependencies が空に近い | **fresh** |
| `@biomejs/biome` または `biome.json(c)` が存在 | **migrate-from-biome** |
| `eslint` / `prettier` 系パッケージまたは設定ファイルが存在（Biome は無し） | **migrate-from-eslint-prettier** |
| `package.json` はあるが上記いずれも入っていない、または oxlint/oxfmt の設定や scripts に不足がある | **update** |

### 複数の既存ツールが同時検出された場合

例: `Biome + Prettier 残骸`、`ESLint + Biome 共存`、`Biome 移行途中で ESLint も残っている` — どれも実プロジェクトで起こりうる。

このような変則ケースは検出された全ツールをユーザーに提示し、**どれを削除してどれを保持するか** を確認する。基本方針:

- oxlint/oxfmt 化が目的なので、Biome / ESLint / Prettier はすべて削除候補
- ただし、削除すると CI やエディタ統合が壊れる可能性があるため、**ユーザー確認なしには削除しない**
- 移行手順は `references/migration.md` の該当セクションを順に実行（例: Biome を消したあと残った Prettier 設定を別途処理）

### モード別の振る舞い

- **fresh** — 全セクションを順に実行
- **migrate-from-biome** — まず `references/migration.md` の「Biome から oxlint + oxfmt への移行」を実行し、その後の各セクション（lefthook, knip, worktrunk, dotenvx）も既存設定とマージしながら適用
- **migrate-from-eslint-prettier** — まず `references/migration.md` の「ESLint + Prettier から oxlint + oxfmt への移行」を実行し、その後の各セクションも既存設定とマージしながら適用。ESLint だけ / Prettier だけ片方しか入っていない場合も同セクションの該当手順だけ実行
- **update** — 各セクションの先頭で「既に設定済みか」を確認し、不足分のみ追加。差分が出たら下記「上書きルール」を適用

---

## ni / nr / na / nlx の使い分け（全スキル共通）

@antfu/ni はパッケージマネージャを検出して委譲するラッパー群。**どれを使うかで「ローカルのピンしたバージョン」か「registry から取り直したもの」かが変わる**ので、以下の基準で選ぶ。

| コマンド | 実体 | 使う場面 |
|---|---|---|
| `ni` | install | 依存の追加 |
| `nci` | clean install | CI |
| `nr <script>` | run script | **package.json の scripts を呼ぶとき（第一候補）** |
| `na <args>` | agent alias（`pnpm <args>` 等にそのまま委譲） | `na exec <bin>` で**ローカルの `node_modules/.bin` を叩く** |
| `nlx <pkg>` | **download & execute**（`pnpm dlx` / `yarn dlx` / `npx` / `bunx`） | **プロジェクトの依存ではないものを一度だけ実行するとき** |

### 判断基準

**そのコマンドは `package.json` の依存に入っているか？**

- **入っている** → `nr <script>`（scripts に書けるなら）、書けないなら `na exec <bin>`
- **入っていない**（scaffolding、codemod、単発ツール） → `nlx <pkg>@<version>`

```bash
# 依存に入っている -> na exec / nr
na exec oxlint --fix {staged_files}   # lefthook から staged files を渡す
na exec lefthook install
nr typecheck                          # scripts に書けるものは nr

# 依存ではない -> nlx（バージョンを明示する）
nlx create-next-app@latest .
nlx @next/codemod@canary upgrade latest
nlx eas-cli@latest init
```

### `nlx` を依存に使ってはいけない理由

`nlx` は README で **"download & execute"** と定義されており、pnpm では `pnpm dlx` に対応する。**pnpm では毎回 npm registry から取り直す**ため、`devDependencies` にピンしたバージョンと食い違う。oxlint や lefthook は registry にも存在するので「動いてしまう」ぶん、バージョンドリフトに気づきにくい。

**実測した挙動**（oxlint 1.77.0 をローカルに置いた状態）:

| 書き方 | pnpm | npm | bun |
|---|---|---|---|
| `nlx oxlint` | `pnpm dlx` → **registry から DL** | `npx` → ローカル優先 | `bunx` → ローカル優先 |
| `na exec oxlint` | ✅ ローカル | ✅ ローカル | ❌ `command not found` |
| `nr <script>` | ✅ ローカル | ✅ ローカル | ✅ ローカル |

**bun を使うプロジェクトでは `na exec` が効かない**（`bun exec` は `node_modules/.bin` を PATH に載せない）。bun 前提なら scripts に逃がして `nr` で呼ぶ（lefthook の節に具体例あり）。

**@antfu/ni に「ローカルバイナリ実行」専用のコマンドは無い。** `na exec` が実質それに当たる。

---

## mise（ランタイムのバージョン管理）

各フレームワークスキルの「Configure mise」ステップはこの節を参照する。

### 原則: バージョンは固定する（`mise.toml` / `package.json` 共通）

**`lts` / `latest` / `*` を残さない。** どちらのファイルでも、目的は同じで
「気づいたら上がっていた」を無くすこと。更新は明示的なコマンドで行い、差分をコミットする。

| ファイル | 書かない | 書く | 更新方法 |
|---|---|---|---|
| `mise.toml` | `lts` / `latest` | `24.15.0` | `mise up <tool>` |
| `package.json` | `"latest"` / `"*"` | `"^2.10.8"` | Dependabot / `nu` |
| `pnpm-workspace.yaml` の `catalog` | `latest` | `^4.12.25` | 同上 |

**`mise.toml` に `lts` / `latest` を残すと**、CI の `jdx/mise-action@v2` や別マシンの
`mise install`、worktree ごとの解決が**別のバージョンを引く**。ローカルで通ったものが
CI で落ちる、worktree ごとに Node が違う、といった再現性の問題になる。

**`package.json` に `"latest"` を残すと**、さらに厄介な形で出る:

- **Dependabot が更新 PR を作らない。** 常に最新にマッチするので「更新すべき差分」が
  存在しない扱いになり、更新の可視化から永久に外れる
- **無関係な `pnpm add` / `pnpm remove` のたびに解決が動く。** ロックファイルが
  再解決されるとき、その依存とサブツリーだけ勝手に新しいバージョンを引く
- 結果、**機能追加の PR に身に覚えのない推移依存が現れる。** Socket Security などの
  サプライチェーンスキャナはこれを新規追加として拾い、無関係なアラートを出す

実例（`setup-dev:dependabot` に詳細）: Storybook を入れただけの PR で
`react-scan@0.5.7 → @sentry/node-core@10.69.0` の「難読化コード」アラートが出た。
`react-scan` が `"latest"` 指定の既存 devDependency だったのが原因で、
PR の変更内容とは何の関係も無かった。

導入時・既存プロジェクトの点検時に洗い出す:

```bash
grep -rn '": *"\(latest\|\*\)"' package.json apps/*/package.json packages/*/package.json
grep -n ': *latest' pnpm-workspace.yaml
grep -nE '= *"(lts|latest)"' mise.toml
```

### 手順

```bash
# 1. まず lts/latest で解決させる
mise use node@lts
mise use pnpm@latest              # or: mise use bun@latest
mise use npm:@antfu/ni@latest

# 2. 解決された具体的なバージョンを確認
mise ls --current

# 3. その値で固定し直す（下は例。手順2 で出た値に置き換える）
mise use node@24.15.0 pnpm@11.19.0 npm:@antfu/ni@30.3.0
```

結果の `mise.toml`:

```toml
[tools]
node = "24.15.0"
pnpm = "11.19.0"
"npm:@antfu/ni" = "30.3.0"
```

`mise install` で全部入る。CI でも `jdx/mise-action@v2` が同じものを入れる。

### 各ツールの役割

| エントリ | なぜ mise で管理するか |
|---|---|
| `node` | プロジェクトの Node バージョン。`.nvmrc` や `engines` より優先される |
| `pnpm` / `bun` | パッケージマネージャ。**`package.json` の `packageManager` と同じバージョンを書く**（片方だけ上げると pnpm が自バージョン切り替えを試みて失敗する。pnpm の節を参照） |
| `npm:@antfu/ni` | `ni` / `nr` / `nci` / `nlx` / `na` を提供する。CI でも `jdx/mise-action@v2` が自動で入れるので、ワークフロー側に別途インストール手順が要らない |
| `cargo:*` | Rust 製 CLI（similarity-ts 等）。`cargo install` でグローバルに入れず、`rust` ツールチェーンごと mise に載せる。similarity-ts の節を参照 |

### mise の pnpm が実際に使われているか確認する

`~/Library/pnpm/pnpm`（Corepack や公式インストーラが置くもの）が PATH で mise の
shim より先にいると、**非対話シェルでは別バージョンの pnpm が黙って使われる**。
エージェント経由の実行や CI 以外のスクリプトで踏みやすい。

```bash
pnpm --version     # mise.toml の値と一致するか
mise which pnpm    # 一致しなければ PATH の順序を疑う
```

**バージョンがずれると設定の置き場所ごと変わる**（pnpm 10 は `package.json` の
`pnpm` フィールド、11 は `pnpm-workspace.yaml`）。`allowBuilds` を書いたのに
`Ignored build scripts:` が出続けるときは、設定の書き方ではなくこれを先に疑う。

### worktree との関係

worktrunk の `.config/wt.toml` に `trust = "mise trust"` を入れておくこと（worktrunk の節を参照）。これが無いと新しい worktree で `mise.toml` が信頼されず、ピンしたバージョンが使われない。

---

### pnpm の設定（モノレポ / 単一プロジェクト共通）

**まず `pnpm --version` を確認する。設定の置き場所がメジャーバージョンで違う。**

`package.json` にはどちらのバージョンでも `packageManager` を書く:

```json
{
  "packageManager": "pnpm@<実際にインストールされているバージョン>"
}
```

**`packageManager` フィールドが必要な理由**: Turborepo は `packageManager` フィールドを起動時に検証し、無いと `Could not resolve workspaces. Missing packageManager field in package.json` で起動拒否する。

#### pnpm 11 以降（現行）

**`package.json` の `pnpm` フィールドは読まれない。** 書いても以下の警告が出て無視される:

```
[WARN] The "pnpm" field in package.json is no longer read by pnpm.
       The following keys were ignored: "pnpm.onlyBuiltDependencies".
       See https://pnpm.io/settings for the new home of each setting.
```

設定はリポジトリルートの `pnpm-workspace.yaml` に camelCase で書く。単一プロジェクト（ワークスペースでない）でもこのファイルが設定の置き場所になる:

```yaml
# packages: を持たない単一プロジェクトでも、設定ファイルとしてこれを置く
managePackageManagerVersions: false

# 配列ではなく「名前 -> boolean」のマップである点に注意（pnpm 10 の onlyBuiltDependencies から形が変わっている）
allowBuilds:
  esbuild: true
  sharp: true
  lefthook: true
  msgpackr-extract: true   # Expo / Metro を使う場合
  workerd: true            # Cloudflare Workers を使う場合
```

許可すべき依存が分からない場合は、`ni` を一度実行すると `[ERR_PNPM_IGNORED_BUILDS] Ignored build scripts: <名前>` として列挙される。`pnpm config list` の `allowBuilds` にも「set this to true or false」として保留中のものが出る。

#### pnpm 10 以前

`package.json` の `pnpm` フィールドを使う（`onlyBuiltDependencies` は**配列**）:

```json
{
  "pnpm": {
    "manage-package-manager-versions": false,
    "onlyBuiltDependencies": ["msgpackr-extract", "esbuild", "sharp", "workerd", "lefthook"]
  }
}
```

#### 各設定の理由（バージョン共通）

**`managePackageManagerVersions: false`（v10 では `manage-package-manager-versions`）**: `packageManager` フィールドがあると pnpm はそのバージョンに自動切り替えを試みるが、`pnpm@<X.Y.Z>` の native バイナリが `~/Library/pnpm/.tools/...` に無いと「Failed to switch pnpm to vX.Y.Z」で失敗する。`false` にすると自動切り替えを抑止し、システム / mise が提供する pnpm をそのまま使う（mise が pnpm をピンしているので、これで十分）。

**`allowBuilds`（v10 では `onlyBuiltDependencies`）**: pnpm 10+ はサプライチェーン攻撃対策のため、デフォルトで `postinstall` スクリプトをスキップする。Metro が依存する `msgpackr-extract`、esbuild、workerd、lefthook のフック登録などはネイティブ build を必要とするため、明示的に許可する。許可しないと Mobile build 時に `Unexpected end of MessagePack data` などのエラーになる。

#### pnpm 11 のサプライチェーン検査（`minimumReleaseAge` / `trustPolicy`）

**pnpm 11 は `minimumReleaseAge` の既定が 24 時間ある。** 設定していなくても効くので、
「常に最新版を固定する」方針と正面から衝突する。新規セットアップでほぼ必ず踏む。

```
[ERR_PNPM_MINIMUM_RELEASE_AGE_VIOLATION] 1 lockfile entries failed verification:
  ai@7.0.64 was published at 2026-08-12T17:40:07Z, within the minimumReleaseAge cutoff
```

`ni <pkg>` は**その時点の最新**を `^<latest>` として書く。公開 24 時間以内の版を掴むと
**下限を満たすバージョンが存在せず install が落ちる**。`ni` した直後は通り、
あとから `pnpm install` し直したときに落ちるので原因が分かりにくい。

対処は範囲の下限を緩めて解決を pnpm に任せること。**pnpm はその範囲内で
「24 時間以上経った最新」を選ぶ**ので、結果として意図どおり固定される。

```yaml
catalog:
  # pnpm 11 は既定で公開 24 時間以内の版を弾く。^7.0.0 にして解決を pnpm に任せる
  ai: ^7.0.0
```

**範囲を変えたらロックファイルを作り直す。** 違反エントリが残ったままだと
`pnpm install` は延々と同じエラーを出す。

```bash
pnpm clean --lockfile && pnpm install
```

**`trustPolicy: no-downgrade` は旧メジャーで誤検知する。** provenance が存在しなかった
時代の版（`semver@6.3.1`、`ua-parser-js@0.7.41`）は、後発の版に provenance が付いた
せいで「トラスト降格」と判定される。実体は据え置きの旧メジャーで攻撃ではない。

```
[ERR_PNPM_TRUST_DOWNGRADE] High-risk trust downgrade for "semver@6.3.1"
This error happened while installing the dependencies of @tanstack/router-plugin
 at @babel/core@7.29.7
```

**エラー本文に出るパッケージ（`@babel/core`）は経路であって犯人ではない。**
弾かれているのは 1 行目の `semver@6.3.1`。ここを読み違えて `@babel/core` の
provenance を調べに行くと時間を溶かす。理由付きで個別に除外する。

```yaml
trustPolicy: no-downgrade
trustPolicyExclude:
  # provenance が無かった時代の 6.x 系。後発の 7.x に付いたため降格判定になる
  - "semver@6.3.1"
  # 同上。2021 年の改竄版は 0.7.29 / 0.8.0 / 1.0.0 なので、これらは別物
  - "ua-parser-js@0.7.41 || 1.0.41"
```

`||` で複数版をまとめられる。全体を緩める `trustPolicyIgnoreAfter`（分）もあるが、
個別除外のほうが「なぜ通したか」が残るので監査できる。

#### `ni` は catalog に自動登録する（pnpm 11）

`catalog:` があるワークスペースでは、`ni -D <pkg>` が **catalog にエントリを足し、
package.json 側を `"catalog:"` にする**。バージョンの単一情報源が保たれるので
そのままでよいが、「package.json を見てもバージョンが分からない」ことに驚かないこと。

### TypeScript 7（ネイティブ版）のインストール

**ネイティブ移植版は `typescript@7` として本体にマージ済み。コマンドは `tsc` で、`@typescript/native-preview` / `tsgo` は不要。**

```bash
ni -D typescript      # 単一プロジェクト
ni -D typescript -w   # モノレポ
```

`typescript@7` の `latest` は Go 実装で、`@typescript/typescript-darwin-arm64` などのプラットフォーム別ネイティブバイナリを optionalDependencies として持つ。bin は従来どおり `tsc` 1つ。

```bash
npm view typescript dist-tags   # latest が 7.x であることを確認
npm view typescript@7 bin       # -> { tsc: 'bin/tsc' }
```

`typecheck` スクリプトは `"typecheck": "tsc --noEmit"` とする。ルート `node_modules/.bin/tsc` がワークスペースの各 package からも解決されるので、モノレポでも各 package に個別インストールは不要。

**TypeScript 7 のネイティブコンパイラは JavaScript の compiler API を提供しない。** ビルド時に TS の API を叩くツール（フレームワークの型生成、一部の bundler プラグイン）が未対応だと弾かれる。既存プロジェクトを上げるときは、まず `nr typecheck` と `nr build` を通してから移行を確定させること。

**`@typescript/native-preview` / `tsgo` について**: プレビュー期の配布形態。npm 上でまだ deprecated にはなっていないが、最新が `7.0.0-dev.<日付>` の dev ビルドで正式リリースより古い。**新規セットアップでは使わない**。既存プロジェクトに入っている場合は差し替える:

```bash
nun @typescript/native-preview
ni -D typescript
# package.json の scripts / lefthook.yml / CI / AGENTS.md の "tsgo" を "tsc" に置換
```

**TypeScript 7 は 5.x より厳しい**。既存プロジェクトを 5.x から上げると、通っていたコードが落ちることがある。代表例:

| エラー | 原因 | 対処 |
|---|---|---|
| `TS2882: Cannot find module or type declarations for side-effect import of './x.css'` | `import "./globals.css"` のような**副作用 import に型宣言が無い**。5.x は黙認していたが 7 は許さない | `declare module "*.css";` を含む `.d.ts` を1枚置く（`*.svg` `*.png` 等も同様） |

移行時は、CI に入れる前にまずローカルで `nr typecheck` を通しきること。

**生成された型定義に依存するフレームワークでの注意**: Next.js の `next-env.d.ts` / `.next/types/`、TanStack Router の `routeTree.gen.ts` のように、**生成物を tsconfig が include している**場合、それを gitignore しているとクリーンな CI チェックアウトで型検査が落ちる。取れる手は2つ:

- **生成物をコミットする** — `routeTree.gen.ts` はこちら（`setup-dev:vite-react`）
- **生成コマンドを `typecheck` の前段に入れる** — `.next/types/` のようにコミットが現実的でないものはこちら（例: `"typecheck": "next typegen && tsc --noEmit"`）

**どちらかに決めて、`.gitignore` と `typecheck` スクリプトを揃えること。** 片方だけ変えると CI でだけ落ちる。

### 上書きルール（共通）

- 既存の設定ファイル（`.oxlintrc.json`, `lefthook.yml`, `knip.json`, `.config/wt.toml` 等）は **無断で上書きしない**。差分を提示してユーザーに「上書き / マージ / スキップ」を選ばせる
- `package.json` の `scripts` と `devDependencies` はマージする。既存項目は残し、新規項目のみ追加。同名で内容が異なる場合は確認
- 機能を移行する目的で**削除する**ファイル（`biome.json`, `.eslintrc.*`, `eslint.config.*`, `.prettierrc*`, `prettier.config.*`, `.eslintignore`, `.prettierignore` など）は、削除前にユーザーに確認

---

## oxlint（linter）

### インストール

```bash
ni -D oxlint
```

モノレポの場合は `-w` を付けてルートにインストール:

```bash
ni -D oxlint -w
```

**update モードの場合**: 既に `oxlint` が入っていれば再インストール不要。`oxlint --version` で最新かを確認し、必要に応じて `nu oxlint` でアップグレード。

### .oxlintrc.json の設定

`na exec oxlint --init` で雛形を生成、または以下を手動で作成:

```json
{
  "$schema": "https://raw.githubusercontent.com/oxc-project/oxc/main/npm/oxlint/configuration_schema.json",
  "plugins": ["typescript", "react", "react-hooks", "import"],
  "categories": {
    "correctness": "error",
    "suspicious": "warn",
    "perf": "warn",
    "style": "off"
  },
  "rules": {
    "react/react-in-jsx-scope": "off"
  }
}
```

**`react/react-in-jsx-scope` を off にする理由**: React 17 以降の new JSX transform では `import React from "react"` が不要だが、oxlint のデフォルトはこのルールを有効にしている。React 17+ プロジェクトでは off が前提。

**`import/order` ルールについて**: oxlint 1.65 系で `import/order` ルールは廃止された（`Rule 'order' not found in plugin 'import'` というエラーで弾かれる）。import の並べ替えは現状、IDE のフォーマッタ / oxfmt の将来サポート / 手動運用でカバーする。将来 oxlint が別ルール名で再導入したら追加する。

**設定の要点:**
- `categories` で recommended 相当のセットを有効化（Biome の `linter.rules.recommended` の代替）
- `.gitignore` に書かれているパスは自動で除外される（Biome の `vcs.useIgnoreFile` 相当はデフォルト動作）
- `react/react-in-jsx-scope` と `react/jsx-uses-react` は React 17+ の new JSX transform では不要。oxlint のデフォルトは前者が ON なので **明示的に off** にする

### React プロジェクト固有の設定

`react`/`react-hooks` プラグインは雛形に含めてある。React Compiler を使う場合、関連ルールは Compiler 側でカバーされるため、競合する `react-hooks` ルールを必要に応じて緩める。

---

## oxfmt（formatter）

### インストール

```bash
ni -D oxfmt
```

モノレポの場合は `-w` を付ける。

### 設定

oxfmt は **alpha 段階**だが、デフォルト挙動はおおむね Prettier 互換:

- インデント: space, indentWidth: 2
- セミコロン: あり
- ダブルクオート
- `.gitignore` に書かれているパスは自動除外

**`.oxfmtrc.json` を必ず作成する**:

```json
{
  "$schema": "./node_modules/oxfmt/configuration_schema.json",
  "ignorePatterns": ["**/*.md", "**/*.css"]
}
```

モノレポの場合もルート 1 か所に置けば十分（`$schema` の相対パスはルートからの参照になる）。

### フォーマット対象

**oxfmt の対象範囲はバージョンによって広がっている。導入時に `nr format:check` を一度流し、どのファイルが対象になったかを必ず確認すること。** 0.62 時点では JS/TS に加えて **Markdown・CSS・JSON も対象**になっている。

| 種別 | 状況 | 推奨 |
|---|---|---|
| JS / TS（`.js`, `.jsx`, `.ts`, `.tsx`） | 本来の対象 | そのまま使う |
| Markdown | **壊れる**。emphasis のアンダースコア処理にバグがあり `created_at` → `created*at`、`_Avoid_` → `\_Avoid*` になる | `ignorePatterns` で除外（必須） |
| CSS | 壊れはしないが、**1行にまとめたルールを全部展開し、hex を小文字化する**。手で整えたスタイルシートだと数百行の差分になる | 既存の手書き CSS があるなら除外を推奨。新規プロジェクトで最初から任せるなら除外不要 |
| JSON / JSONC（`package.json`, `tsconfig.json` 等） | 整形される。キー順は保たれるが折り返しが変わる | 通常はそのままで問題ない |
| YAML | 対象外 | 手動運用 |

**既存プロジェクトへの導入時（update / migrate モード）は、`nr format` を流す前に `git add -A` で現状を index に入れておく。** そうすれば `git diff` でフォーマッタの差分だけを切り出してレビューでき、意図しない大量書き換えを戻せる（コミットは不要）。

Markdown / CSS の整形が必要になった場合は Prettier 等の併用を検討する。oxfmt 側が安定したら `ignorePatterns` から外す。

---

## lint + format スクリプト

`package.json` の `scripts`:

```json
{
  "scripts": {
    "lint": "oxlint .",
    "lint:fix": "oxlint --fix .",
    "format": "oxfmt .",
    "format:check": "oxfmt --check ."
  }
}
```

- `nr lint` — チェックのみ（CI 向け）
- `nr lint:fix` — 自動修正
- `nr format` — フォーマット適用（書き換え）
- `nr format:check` — フォーマット差分の検出のみ（CI 向け）

---

## 既存ツールからの移行

`migrate-from-biome` / `migrate-from-eslint-prettier` モードの場合、**先に
`references/migration.md` を読んで最後まで実行する。** 内容:

- Biome の削除と `biome.json` の設定引き継ぎ
- ESLint / Prettier の削除と設定引き継ぎ（片方だけ入っている場合も対応）
- lefthook / package.json / CI / エディタ統合の置換

`fresh` / `update` モードでは読まなくてよい。移行後は以下の各セクションに戻る。

---

## lefthook

### インストール

```bash
ni -D lefthook
na exec lefthook install
```

モノレポの場合は `-w` を付ける。

### lefthook.yml

```yaml
pre-commit:
  commands:
    encrypt-env:
      glob: ".env*"
      exclude: ".env.keys"
      run: na exec dotenvx encrypt
      stage_fixed: true
    lint:
      glob: "*.{ts,tsx,js,jsx}"
      run: na exec oxlint --fix {staged_files}
      stage_fixed: true
    format:
      glob: "*.{ts,tsx,js,jsx}"
      run: na exec oxfmt {staged_files}
      stage_fixed: true
    typecheck:
      run: nr typecheck

pre-push:
  commands:
    knip:
      run: nr knip
```

**`commands:` と `jobs:` について**: lefthook 1.10 以降 `jobs:` が導入され、v2 が自分で生成する雛形も `jobs:` になっている。ただし **`commands:` は lefthook 2.1 系でも有効**（`lefthook validate` で確認済み）。既存の `lefthook.yml` が `commands:` なら書き換える必要はない。

**`nlx` ではなく `na exec` を使う理由**: oxlint / oxfmt / dotenvx は `devDependencies` に入っているローカルバイナリなので、`nlx`（= `pnpm dlx`）だと registry から取り直してピンしたバージョンと食い違う。詳細は冒頭の **「ni / nr / na / nlx の使い分け」** を参照。

**bun を使うプロジェクトでは `na exec` が効かない**（`bun exec` は `node_modules/.bin` を PATH に載せない）。その場合は `package.json` に staged files を受け取る専用スクリプトを足し、`nr` 経由で呼ぶ:

```json
{ "scripts": { "lint:staged": "oxlint --fix", "format:staged": "oxfmt" } }
```

```yaml
    lint:
      run: nr lint:staged {staged_files}
```

`nr` は追加引数をスクリプトに転送する（`nr lint:staged a.ts` → `oxlint --fix a.ts`）。**既存の `lint:fix` を流用しないこと** — `"oxlint --fix ."` の `.` が残るため `oxlint --fix . a.ts` となり、staged files に絞れず全体を lint してしまう。

**`pnpm exec tsc` ではなく `nr typecheck` を使う理由**:
- モノレポではルートに `tsconfig.json` が無いと `tsc` がコマンド引数を誤解釈する
- 生成物を先に作る必要があるフレームワーク（Next.js の `next typegen` 等）では、typecheck が単一コマンドで済まない
- `nr typecheck` (= `pnpm run typecheck`) は package.json の scripts を呼ぶので、モノレポでは `turbo run typecheck` 経由で各 package の `tsc --noEmit` が回り、単一プロジェクトでは直接 `tsc --noEmit` が回る。前段の生成コマンドも scripts 側に隠せる。どの構成にも適合する

- `lint` と `format` を別コマンドに分けているのは、oxlint と oxfmt が独立したバイナリのため
- `stage_fixed: true` で自動修正されたファイルを再度ステージングする
- 順序は `lint` → `format`。oxlint の `--fix` が import 並び替え等を行ったあと、oxfmt が最終整形を行う

**`knip` を pre-commit ではなく pre-push に置く理由**:
- `lint`/`format` は `{staged_files}` だけを対象にできるが、knip の「未使用判定」は全消費側を見ないと成立しないため**プロジェクト全体の走査が必要**で、staged 単位に絞れない。毎コミットで全体走査を走らせると煩わしい（速度というより走査スコープの問題）
- push はコミットより低頻度なので全体走査を1回挟むコストが許容でき、CI に投げる前にローカルで未使用コード・依存を潰せて round-trip が減る
- `setup-dev:ci` でも `nr knip` を回すため、ローカルの pre-push をすり抜けても最終的に CI で検出される（二重の網）

### update / migrate モードでの注意

既存の `lefthook.yml` がある場合、旧 linter/formatter のコマンド行のみを置換する。他のフック（`encrypt-env`, `typecheck` 等）は維持する:

- `nlx biome check --write {staged_files}` → `na exec oxlint --fix {staged_files}` + `na exec oxfmt {staged_files}` の 2 段
- `nlx eslint --fix {staged_files}` → `na exec oxlint --fix {staged_files}`
- `nlx prettier --write {staged_files}` → `na exec oxfmt {staged_files}`

**置換先を `nlx` にしないこと。** 既存の `lefthook.yml` が `nlx eslint` のような書き方をしていても、それを踏襲しない（上の「`nlx` ではなく `na exec` を使う理由」を参照）。

`lint-staged` 経由で呼んでいるケースもあるため、`.lintstagedrc.*` や `package.json` の `lint-staged` フィールドも同様に置換対象。

---

## knip

### インストール

```bash
ni -D knip
```

モノレポの場合は `-w` を付ける。

### 単一プロジェクト用 knip.json

```json
{
  "$schema": "https://unpkg.com/knip@latest/schema.json",
  "project": ["src/**/*.{ts,tsx}"],
  "ignoreDependencies": ["babel-plugin-react-compiler"],
  "rules": {
    "devDependencies": "warn",
    "exports": "warn"
  }
}
```

### モノレポ用 knip.json

```json
{
  "$schema": "https://unpkg.com/knip@latest/schema.json",
  "workspaces": {
    "apps/web": {
      "project": ["src/**/*.{ts,tsx}"],
      "ignoreDependencies": ["babel-plugin-react-compiler"],
      "rules": {
        "devDependencies": "warn",
        "exports": "warn"
      }
    },
    "apps/api": {
      "project": ["src/**/*.{ts,tsx}"],
      "entry": ["src/index.ts"]
    },
    "packages/shared": {
      "project": ["src/**/*.{ts,tsx}"],
      "entry": ["src/index.ts"]
    }
  }
}
```

### knip 6 では `rules` はトップレベルだけ

workspace ごとに書くと起動もしない。

```
ERROR: Invalid input (location: workspaces.apps/web, unrecognized_keys: rules)
```

```jsonc
{
  "rules": { "devDependencies": "warn", "exports": "warn" },  // ここだけ
  "workspaces": { "apps/web": { "project": ["src/**/*.{ts,tsx}"] } }
}
```

**設定ファイルは `knip.jsonc` にできる。** 除外の理由をコメントで残せるので、
`ignoreDependencies` を足すときに効く（なぜ通したかが分からない除外は後で消せない）。

### knip の注意事項

- knip は Vite/TanStack Router のプラグインを自動検出するため、`entry`/`ignore` は基本不要
- `rules` で `warn` にしているのは、TanStack Router のファイルベースルーティングの `Route` エクスポートが未使用扱いになるため
- `babel-plugin-react-compiler` は vite config 内の Babel プラグイン文字列配列で参照されるため、knip が依存関係として検出できない。`ignoreDependencies` に追加が必要
- 実行タイミングは、ローカルが lefthook の **pre-push**、リモートが **CI**（`setup-dev:ci`）の二段。pre-commit には置かない（理由は「lefthook」セクション参照）

### CSS の `@import` でしか使わない依存は追えない

**knip は CSS を解析しない。** `styles.css` からしか参照されないパッケージは
全部「未使用依存」になり、`dependencies` は `warn` にしていない限り **exit 1** になる。

```css
/* apps/web/src/styles.css */
@import "tailwindcss";
@import "tw-animate-css";
@import "shadcn/tailwind.css";
@import "@fontsource-variable/geist";
```

```jsonc
"apps/web": {
  "project": ["src/**/*.{ts,tsx}"],
  // styles.css の @import は CSS 側なので knip から追えない
  "ignoreDependencies": [
    "tailwindcss",
    "tw-animate-css",
    "shadcn",
    "@fontsource-variable/geist",
  ],
},
```

**消してはいけない。** これらは実際に使われている（消すとフォントもアニメーションも
本番から消える）。knip 自身が

```
Configuration hints (1)
.css  apps/web  knip.jsonc  Compiled extension excluded by project (imports not followed)
```

と出しているので、このヒントが出ているときの「未使用依存」はまず CSS 由来を疑う。
Tailwind v4 系（`@import "tailwindcss"`）、フォント（`@fontsource-*`）、
`tw-animate-css`、`shadcn/tailwind.css` が典型。

### exit code に効くのは error レベルだけ

`rules` で `warn` にした種別は**出力には並ぶが exit code を変えない**。

```jsonc
"rules": { "devDependencies": "warn", "exports": "warn" }
```

この設定だと「Unused exports (28)」が並んでいても exit 0 になる。CI が knip で
落ちたときに 28 行の未使用 export を消しに行くのは**完全に無駄**で、見るべきは
`Unused files` と `Unused dependencies` の 2 つだけ。まず `echo $?` で切り分ける。

shadcn を入れると生成コンポーネントが未使用 export を数十件生むので、
`exports: "warn"` はここでも効く。一方で**どこからも import されない生成ファイル**
（`ui/badge.tsx` など）は `Unused files` = error になるので、そちらは実際に削除する。
ディレクトリごと `ignore` に入れると、本当に使っていないファイルまで見えなくなる。

---

## similarity-ts（重複コード検出）

knip が「未使用コード」を見つけるのに対し、similarity-ts は「似たコード（重複の可能性）」を見つける。リファクタリング候補の洗い出しに使う。

### インストール

**similarity-ts は npm パッケージではない。** Rust 製 CLI で crates.io にのみ存在する（`ni -D similarity-ts` は `ERR_PNPM_FETCH_404` になる）。導入するかどうかは Rust ツールチェーンを引き込む判断になるので、**ユーザーに確認してから入れること**。

**cargo 製のツールも mise で管理する。`cargo install` でグローバルに入れない。** ツールのバージョンも Rust ツールチェーン自体も `mise.toml` に載せて、他のツールと同じ再現性を持たせる:

```bash
# Rust ツールチェーン自体も mise 管理下に置く（cargo が未インストールでもこれで入る）
mise use rust@latest
mise use cargo:similarity-ts@latest

mise ls --current    # 解決されたバージョンを確認
# 確認した値で固定し直す（mise.toml に latest を残さない）
mise use rust@1.95.0 cargo:similarity-ts@0.5.0
```

結果の `mise.toml`:

```toml
[tools]
rust = "1.95.0"
"cargo:similarity-ts" = "0.5.0"
```

`mise install` で解決され、CI でも `jdx/mise-action@v2` が同じものを入れる。cargo-binstall があればビルド済みバイナリを取り、無ければソースからビルドするため**初回は数分かかる**（CI のキャッシュを検討する価値がある）。

**CLI は `node_modules/.bin` ではなく mise の shims 経由で PATH に来る**ので、`package.json` の `devDependencies` には現れない。knip が「使われていないバイナリ」と誤検出する場合は `ignoreBinaries` に足す。

**導入を見送る場合**: 下記の `scripts` と `setup-dev:ci` の `nr similarity` ステップも**両方**落とすこと。片方だけ残すと CI が「コマンドが無い」で落ちる。

**update モードの場合**: 既に入っていれば `similarity-ts --version` で確認し、必要に応じて `mise up cargo:similarity-ts`。

### package.json の scripts

```json
{
  "scripts": {
    "similarity": "similarity-ts src",
    "similarity:strict": "similarity-ts src --threshold 0.95"
  }
}
```

対象パスは実際のソースディレクトリに合わせる。`src/` を持たない構成（Next.js の App Router など）では `similarity-ts app components lib` のように実在するディレクトリを列挙する。

モノレポの場合は対象パスを各 app/package に合わせて指定する（例: `similarity-ts apps packages`）。

- `nr similarity` — デフォルト閾値で類似コードを検出（CI 向け／日常運用）
- `nr similarity:strict` — 高閾値で「ほぼ完全な重複」のみに絞る（リファクタリング候補洗い出し用）

### 設定の方針

similarity-ts は基本的に CLI フラグでチューニングする（設定ファイル不要）。プロジェクトに応じて以下を検討:

- `--threshold <0.0-1.0>` — 類似度の下限。デフォルト `0.85` 前後。低くすると検出範囲が広がるが誤検出が増える
- `--min-lines <N>` — 検出対象の最小行数。短いユーティリティ関数のノイズを減らしたいときに上げる
- 対象パスは src 配下に絞る。生成ファイル（`dist/`, `.next/`, `routeTree.gen.ts` 等）は `.gitignore` 経由で除外されるため通常は明示不要

### 運用ルール

- pre-commit には**入れない**。解析が比較的重いため、毎コミットで走らせるとコミット時間が伸びる
- CI で `nr similarity` を回し、PR レビュー時に「重複が増えていないか」を確認する（`setup-dev:ci` を参照）
- 既存リポジトリに導入する場合、初回は大量の検出結果が出る可能性が高い。最初は `nr similarity:strict` で完全重複だけ潰し、徐々に閾値を下げていく運用が現実的

### 移行時の扱い

`migrate-from-biome` / `migrate-from-eslint-prettier` どちらの移行でも、similarity-ts は新規導入扱いで上記の「インストール」「package.json の scripts」を実行する。Biome や ESLint には類似コード検出機能が無いため、置換対象は無い。

---

## worktrunk (Git Worktree Management)

### .config/wt.toml

```toml
[[pre-start]]
trust = "mise trust"

[[pre-start]]
install = "ni"
```

- `[[pre-start]]`: worktree に入るたびに実行されるフック（作成時も含む）
  - `trust`: worktree ディレクトリで `mise.toml` を信頼済みにする（これがないと mise が設定を読み込まず、正しい Node/pnpm バージョンが使えない）
  - `install`: 依存関係のインストール（lockfile に変更がなければ高速に完了する）
- dotenvx を使う場合、`.env` は暗号化されて git 管理されるため worktree にも自動で含まれる。`copy` は不要。
- テンプレート変数（`{{ branch }}`, `{{ branch | hash_port }}` 等）も使用可能。

### フックは pipeline 形式（`[[...]]`）で書く

worktrunk のフックには 3 つの記法があり、**実行順序が違う**:

| 記法 | 例 | 実行 |
|---|---|---|
| 文字列 | `pre-start = "ni"` | 単一コマンド |
| テーブル `[pre-start]` | `trust = ...` / `install = ...` | **並列** |
| パイプライン `[[pre-start]]` | ブロックを複数並べる | **ブロック単位で直列**（1ブロック内の複数キーは並列） |

`trust` → `install` のように**順序に依存するものは必ず `[[pre-start]]` を分けて書く**。テーブル形式は並列実行なので、`mise trust` が終わる前に `ni` が走り、意図しない Node/pnpm バージョンでインストールされる。

順序に依存しないもの（dev サーバーと watch の同時起動など）は 1 ブロックにまとめて並列で走らせてよい:

```toml
[[post-start]]
install = "ni"

[[post-start]]
server = "nr dev"
watch = "nr watch"
```

テーブル形式の挙動は worktrunk 0.58.0 で**破壊的に変更された**:

| version | `[pre-start]`（テーブル形式）の挙動 | 警告 |
|---|---|---|
| 〜0.36 | 直列 | なし |
| 0.37〜0.57 | 直列（据え置き） | 出る（移行猶予期間） |
| 0.58〜 | **並列** | 出ない（仕様変更完了） |

**0.58.0 以降は警告が出ない**ので、テーブル形式が残っている既存プロジェクトは気づかないまま並列実行になっている。`wt config show` で記法を確認し、`wt config update`（直列を維持するマイグレーションを提示する）で `[[...]]` に移行すること。

### copy-ignored を使う場合は必ず .worktreeinclude を作る

dotenvx を使わないプロジェクトで `.env.local` 等を worktree に引き継ぐために
`wt step copy-ignored` を pre-start に入れる場合、**`.worktreeinclude` をリポジトリルートに必ず作成する**。

```toml
[[pre-start]]
trust = "mise trust"

[[pre-start]]
copy = "wt step copy-ignored"

[[pre-start]]
install = "ni"
```

```text
# .worktreeinclude
.env*
```

**`node_modules/` は含めない。** pnpm ならローカルストアからのハードリンクで数秒で入り直す（実測 4〜6 秒）。数十万ファイルをコピーする方が遅い。

- `.worktreeinclude` が無いと gitignore された**全ファイル**がコピーされる。`.next` / `.turbo` / `dist` などのビルドキャッシュが古い worktree からコピーされると、ビルドツールがキャッシュ不整合を起こす（例: Next.js の Turbopack が CPU 数百%でライブロックし、dev サーバーが応答不能になる実害あり）
- コピー対象は「gitignore されている **かつ** `.worktreeinclude` にマッチする」ものだけ
- `.worktreeinclude` は git 管理に含める（コピー元＝メイン worktree に存在する必要がある）
- 導入後は `wt step copy-ignored --dry-run` でコピー対象が意図通りか確認する

worktrunk のユーザー設定は `~/.config/worktrunk/config.toml`、プロジェクト設定は `.config/wt.toml` に配置する。

---

## dotenvx（暗号化された環境変数管理）

### インストール

```bash
ni -D @dotenvx/dotenvx
```

モノレポの場合は `-w` を付ける。

### セットアップ

```bash
# .env ファイルを作成（通常通り）
# 暗号化（.env が暗号化され、.env.keys に復号キーが生成される）
na exec dotenvx encrypt
```

### 運用ルール

- `.env` → 暗号化された状態で **git にコミットする**
- `.env.keys` → 復号キー。**git にコミットしない**（.gitignore に追加）
- `.env.example` → **不要**（暗号化された .env 自体がキー名のドキュメントになる）
- 環境ごとのファイル（`.env.production`, `.env.staging` 等）も同様に暗号化して管理可能

### コマンド実行時の使用

開発サーバーや CI で環境変数を復号して使う:

```bash
na exec dotenvx run -- nr dev
```

`package.json` の scripts で使う場合:

```json
{
  "scripts": {
    "dev": "dotenvx run -- vite",
    "dev:api": "dotenvx run -- wrangler dev"
  }
}
```

### チームメンバーへの共有

`.env.keys` の内容をチームメンバーに安全に共有する（1Password, Slack DM 等）。メンバーは `.env.keys` を配置するだけで復号できる。

---

## .gitignore（共通ベース）

```
node_modules/
dist/
*.tsbuildinfo
.env.keys
.playwright-mcp/
.claude/*.log
.claude/sessions/
.claude/settings.local.json
.claude/team-state.md
```

**`.claude/` 配下を選択的に ignore する理由**: Claude Code はプロジェクト直下の `.claude/` を「共有設定（settings.json, commands/, agents/ 等）」と「端末ローカル / 一時ファイル」の混在領域として使う。後者は他メンバーや CI から参照されないため git に乗せず、前者はチームで共有する。`.claude/` 全体を ignore すると共有設定まで除外されてしまうので、ファイル単位で列挙する。

| 除外対象 | 性質 |
|---|---|
| `.claude/*.log` | Claude Code やプラグインが直下に吐くログ全般（`sessions.log` など、端末ローカル） |
| `.claude/sessions/` | セッション保存ディレクトリ（端末ローカル） |
| `.claude/settings.local.json` | per-machine ローカル設定（権限 allowlist など個人ごとに異なる）。`.claude/settings.json` は共有なのでこちらは ignore しない |
| `.claude/team-state.md` | `team` 系コマンドが作業中に生成する一時状態ファイル |

### モノレポ向け追加項目

```
.wrangler/
.turbo/
```
