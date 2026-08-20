# Biome / ESLint+Prettier から oxlint + oxfmt への移行

`setup-dev:tooling` の「モード判定」で `migrate-from-biome` /
`migrate-from-eslint-prettier` になった場合に読む。`fresh` / `update` モードでは不要。

移行を終えたら SKILL.md の各セクション（lefthook, knip, similarity-ts, worktrunk,
dotenvx）に戻り、既存設定とマージしながら適用する。

---

## Biome から oxlint + oxfmt への移行

`migrate-from-biome` モードで最初に実行する一連の手順。

### 1. 旧 Biome の削除

```bash
nun @biomejs/biome
# モノレポの場合: nun @biomejs/biome -w
rm -f biome.json biome.jsonc
```

ファイル削除前にユーザーへ確認すること。

### 2. oxlint + oxfmt のインストール

SKILL.md の「oxlint」「oxfmt」セクションに従う。

### 3. 設定の引き継ぎ

旧 `biome.json` の代表的な設定 → 新 `.oxlintrc.json` への対応:

| Biome の設定 | 移行先 |
|---|---|
| `formatter.indentStyle: "space"` / `indentWidth: 2` | oxfmt のデフォルト（設定不要） |
| `linter.rules.recommended: true` | `.oxlintrc.json` の `categories.correctness: "error"` + `categories.suspicious: "warn"` |
| `assist.actions.source.organizeImports` | **代替なし**（`import/order` は oxlint 1.65 系で廃止）。IDE の import 整理か手動運用に倒す |
| `vcs.useIgnoreFile: true` | oxlint / oxfmt のデフォルト動作（設定不要） |
| `files.includes` のカスタム除外 | `.gitignore` に移すか、`.oxlintrc.json` の `ignorePatterns` を使う |

旧 `biome.json` にプロジェクト固有のカスタムルール（`linter.rules.*` の個別指定）があった場合は、ESLint ルール名にマッピングして `.oxlintrc.json` の `rules` に転記する。マッピングが不明なルールはユーザーに確認。

### 4. lefthook と package.json の更新

SKILL.md の「lefthook」セクションに従い、`biome check --write` を `oxlint --fix` + `oxfmt` の 2 段に分割する。`package.json` の旧 scripts (`"lint": "biome check ."`, `"format": "biome format --write ."`) は SKILL.md の「lint + format スクリプト」の形に書き換える。

### 5. CI の確認

`nr lint` の中身が oxlint に切り替わるため、ワークフロー側の YAML 変更は基本不要。ただし Biome は `biome check` 一発でフォーマット差分も拾っていたため、CI ステップに `nr format:check` を追加することを推奨。

### 6. 検証

```bash
nr lint
nr format:check
nr typecheck
```

エラーが出なければ移行完了。

---

## ESLint + Prettier から oxlint + oxfmt への移行

`migrate-from-eslint-prettier` モードで最初に実行する一連の手順。ESLint だけ / Prettier だけ片方しか入っていない場合も、該当する小節だけ実行すればよい。

### 1. 旧 ESLint の削除（ESLint が入っている場合のみ）

削除対象パッケージを `package.json` から確認:

```bash
nun eslint
# 関連パッケージも一緒に削除
nun @typescript-eslint/parser @typescript-eslint/eslint-plugin
nun eslint-plugin-react eslint-plugin-react-hooks eslint-plugin-import eslint-plugin-jsx-a11y
nun eslint-config-prettier eslint-config-airbnb eslint-config-standard
# その他 eslint-* / @typescript-eslint/* で始まるパッケージがあれば確認のうえ削除
```

モノレポでは `-w` を付ける。プロジェクト固有のカスタム ESLint プラグインがある場合は削除前にユーザー確認。

設定ファイルを削除（ユーザー確認のうえ）:

```bash
rm -f .eslintrc .eslintrc.{js,cjs,mjs,json,yml,yaml} eslint.config.{js,cjs,mjs,ts} .eslintignore
```

### 2. 旧 Prettier の削除（Prettier が入っている場合のみ）

```bash
nun prettier
# Prettier プラグイン
nun prettier-plugin-tailwindcss @trivago/prettier-plugin-sort-imports
# その他 prettier-plugin-* / @*/prettier-plugin-* があれば確認のうえ削除
```

設定ファイルを削除（ユーザー確認のうえ）:

```bash
rm -f .prettierrc .prettierrc.{js,cjs,mjs,json,yml,yaml,toml} prettier.config.{js,cjs,mjs,ts} .prettierignore
```

### 3. oxlint + oxfmt のインストール

SKILL.md の「oxlint」「oxfmt」セクションに従う。

### 4. ESLint 設定の引き継ぎ

旧 ESLint 設定（`.eslintrc.*` / `eslint.config.*`）の代表的な設定 → 新 `.oxlintrc.json` への対応:

| ESLint の設定 | 移行先 |
|---|---|
| `extends: ["eslint:recommended"]` | `categories.correctness: "error"` |
| `extends: ["plugin:@typescript-eslint/recommended"]` | `plugins: ["typescript"]` + `categories.correctness: "error"` |
| `extends: ["plugin:react/recommended"]` | `plugins: ["react"]` |
| `extends: ["plugin:react-hooks/recommended"]` | `plugins: ["react-hooks"]` |
| `extends: ["plugin:import/recommended"]` | `plugins: ["import"]` |
| `extends: ["plugin:jsx-a11y/recommended"]` | `plugins: ["jsx-a11y"]` |
| `rules: { "<rule-name>": "error" }` | `.oxlintrc.json` の `rules` に同名で転記 |
| `parserOptions` / `parser` | oxlint は不要（Rust 製パーサ内蔵） |
| `env: { browser: true, node: true }` | oxlint は不要（自動推論） |

oxlint がサポートしていない ESLint ルールがある場合（例: 一部の typescript-eslint の型情報を要するルール、独自プラグイン由来のルール）は、ユーザーに以下のいずれかを選んでもらう:

- そのルールを諦めて `.oxlintrc.json` から外す
- 部分的に ESLint を残す（その場合は本移行の対象外、別途運用）

旧 `.eslintignore` のカスタム除外は `.gitignore` に統合するか、`.oxlintrc.json` の `ignorePatterns` に転記する。

### 5. Prettier 設定の引き継ぎ

oxfmt は alpha のため設定項目が限定的。旧 Prettier 設定の対応:

| Prettier の設定 | oxfmt の対応 |
|---|---|
| `printWidth: 80` | デフォルト一致（変更不要） |
| `tabWidth: 2` | デフォルト一致 |
| `useTabs: false` | デフォルト一致 |
| `semi: true` | デフォルト一致 |
| `singleQuote: false`（または未設定） | デフォルト一致（double quote） |
| `singleQuote: true` | **oxfmt では対応設定が限定的** — ユーザー確認のうえ受け入れるか、Prettier を一時残置するか判断 |
| `trailingComma` / `arrowParens` 等の細かい設定 | 同上、oxfmt の現状サポート範囲を確認しつつ判断 |
| `prettier-plugin-*`（Tailwind 用、import sort 用 等） | Tailwind class sort は oxlint プラグイン側。**import sort は代替なし**（`import/order` は廃止済み）。並べ替えを維持したいなら該当の Prettier プラグインだけ残す判断もある |

設定の乖離が大きい場合（例: `singleQuote: true` を強く維持したい）、oxfmt の現状を踏まえて Prettier を一時残置する選択もユーザーに提示する。

旧 `.prettierignore` のカスタム除外は `.gitignore` に統合する。

### 6. lefthook と package.json の更新

SKILL.md の「lefthook」セクションに従い、`eslint --fix` / `prettier --write` のコマンドを `oxlint --fix` + `oxfmt` に置換する。`package.json` の旧 scripts（`"lint": "eslint ."`, `"format": "prettier --write ."` など）は SKILL.md の「lint + format スクリプト」の形に書き換える。

### 7. CI とエディタ統合の確認

- `.github/workflows/*.yml` で `eslint` / `prettier` を直接呼んでいる箇所があれば `oxlint` / `oxfmt` に置換、または `nr lint` / `nr format:check` 経由に統一
- `.vscode/settings.json` の `editor.defaultFormatter` が `esbenp.prettier-vscode` の場合は oxfmt 用エディタ拡張に変更（拡張が未対応の場合は `nr format` で代替）
- `.vscode/extensions.json` の推奨拡張も同様に更新

### 8. 検証

```bash
nr lint
nr format:check
nr typecheck
```

エラーが出なければ移行完了。
