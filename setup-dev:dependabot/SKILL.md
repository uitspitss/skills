---
name: setup-dev:dependabot
description: `.github/dependabot.yml` を生成する。npm / github-actions / docker、モノレポの directory 分割、groups、pnpm catalog の制約、Expo SDK の ignore、auto-merge、Renovate からの移行。他の setup-dev スキルから参照される。
disable-model-invocation: true
---

# Dependabot Setup

新規セットアップに加え、既存 `.github/dependabot.yml` の更新と Renovate からの移行にも対応する。`$ARGUMENTS` が指定されていればそのディレクトリ、なければカレントディレクトリで動作する。

## なぜ Dependabot を採用するか

- **GitHub ネイティブ**: トークン不要、設定ファイルだけで動く
- **Security Advisories と直結**: 既知脆弱性は `dependabot.yml` の設定に関係なく自動で PR が来る
- **モノレポ対応**: `directory` を ecosystem 単位で複数指定できる
- **groups**: 複数の dependency をまとめて1 PR にできる（v2 で大幅強化）

Renovate と比較した場合の主な差: Renovate の方が設定の表現力は高い（カスタムスケジュール、grouping ルール、replacement、postUpgradeTasks など）が、設定難度も高い。Dependabot で済む規模なら Dependabot を選ぶのが運用コストとして軽い。

## モード判定（最初に実行する）

```bash
# 既存設定の検出
ls .github/dependabot.yml .github/dependabot.yaml 2>/dev/null
ls renovate.json .github/renovate.json .renovaterc .renovaterc.json 2>/dev/null
```

| 検出結果 | モード |
|---|---|
| `.github/dependabot.yml` が無く Renovate も無い | **fresh** |
| `renovate.json` 系が存在 | **migrate-from-renovate** |
| `.github/dependabot.yml` が既に存在 | **update** |

### 上書きルール

既存の `dependabot.yml` は無断で上書きしない。差分を提示してユーザーに「上書き / マージ / スキップ」を選ばせる。Renovate からの移行では、Renovate 設定を**削除する前に** Dependabot 設定が機能していることをユーザーに確認する（脆弱性検知が空白期間で抜け落ちないようにする）。

## .github/dependabot.yml の基本構成

すべての更新ターゲットは `updates:` 配列の要素として宣言する。最小例:

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Asia/Tokyo"
    open-pull-requests-limit: 10
    labels:
      - "dependencies"
    commit-message:
      prefix: "chore(deps)"
      prefix-development: "chore(deps-dev)"
      include: "scope"
```

**主要パラメータの選び方:**

- `schedule.interval`: `weekly` 推奨。`daily` は PR が積み上がりすぎる。`monthly` は脆弱性対応が遅れる
- `open-pull-requests-limit`: 5〜10 が現実的。0 にすると **security updates 以外は止まる**（バージョン更新 PR は出ないが、脆弱性検知 PR は引き続き来る）。これは「セキュリティだけ自動化したい」場合に便利
- `versioning-strategy`: npm では `increase` か `auto`（既存挙動を尊重）。lockfile を見て自動判定で十分なケースが多い
- `commit-message.prefix` を分ければ、conventional commits の流儀と整合する

## ecosystem 別テンプレ

### npm（Node.js / TypeScript プロジェクト）

```yaml
- package-ecosystem: "npm"
  directory: "/"
  schedule:
    interval: "weekly"
  groups:
    types:
      patterns:
        - "@types/*"
    dev-dependencies:
      dependency-type: "development"
      update-types:
        - "minor"
        - "patch"
    production-minor-patch:
      dependency-type: "production"
      update-types:
        - "minor"
        - "patch"
  open-pull-requests-limit: 5
  commit-message:
    prefix: "chore(deps)"
    prefix-development: "chore(deps-dev)"
    include: "scope"
```

**`groups` の意図**: メジャー更新だけは個別 PR にして、minor/patch はまとめる。型定義パッケージ（`@types/*`）はバージョン同期問題が起きやすいので独立 group。

#### 前提: specifier に `"latest"` / `"*"` を残さない

**Dependabot を入れる前に `setup-dev:tooling` の「原則: バージョンは固定する」を
済ませておくこと。** `"latest"` 指定のパッケージは常に最新にマッチするので
「更新すべき差分」が存在しない扱いになり、**Dependabot が更新 PR を作らない**。
更新の可視化から永久に外れるうえ、無関係な `pnpm add` のたびに解決が動く。

実害の例。Storybook を入れただけの PR に、無関係なサプライチェーンアラートが出た:

```
Warn / High  Obfuscated code: npm @sentry/node-core@10.69.0 は 60% の確率で難読化されている
             経路: pnpm-lock.yaml → react-scan@0.5.7 → @sentry/node-core@10.69.0
```

`react-scan` は `"latest"` 指定の既存 devDependency で、PR の変更内容とは無関係。
ロックファイルが再解決された拍子にサブツリーだけ新しいバージョンを引き、
Socket Security が「新規追加された依存」として拾った。**Dependabot が
面倒を見ない依存ほど、こういう形で不意に動く。**

### github-actions（CI で使う actions の更新）

```yaml
- package-ecosystem: "github-actions"
  directory: "/"
  schedule:
    interval: "weekly"
  groups:
    actions:
      patterns:
        - "*"
  commit-message:
    prefix: "ci(deps)"
```

`actions/checkout@v7` のような GitHub Actions のバージョンを更新する。これは別 ecosystem 扱いなので、npm の設定と分けて宣言する必要がある。

### docker（Dockerfile の FROM 行）

```yaml
- package-ecosystem: "docker"
  directory: "/"
  schedule:
    interval: "weekly"
  commit-message:
    prefix: "build(deps)"
```

Dockerfile を使うプロジェクトのみ。Cloudflare Workers のように Dockerfile を使わない場合は不要。

### pip / cargo / bundler 等

GitHub 公式の[サポート対象 ecosystem](https://docs.github.com/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file#package-ecosystem) を参照。基本パターンは npm と同じ。

## モノレポでの directory パターン

pnpm workspaces / Turborepo のような構成では、各 `package.json` の場所ごとに `directory` を分けて宣言する必要がある。Dependabot はサブディレクトリの lockfile を自動探索しない。

```yaml
version: 2
updates:
  # ルート
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5

  # アプリ別
  - package-ecosystem: "npm"
    directory: "/apps/web"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5

  - package-ecosystem: "npm"
    directory: "/apps/api"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5

  - package-ecosystem: "npm"
    directory: "/apps/mobile"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5

  - package-ecosystem: "npm"
    directory: "/packages/shared"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 3

  # CI actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

実在するアプリ・パッケージだけ宣言する（`apps/mobile` が無いなら entry も無し）。

### pnpm catalog の重要な制約

`pnpm-workspace.yaml` の `catalog:` でバージョンを一元管理している場合、**Dependabot は catalog 経由のバージョンを更新できない**。各 `package.json` に `"react": "catalog:"` と書かれていても、Dependabot は「`catalog:` は具体的バージョンではない」と判定して更新対象から外す。

**対応策:**
- catalog で管理しているパッケージは、`pnpm-workspace.yaml` 自体を**手動で**更新するか、`pnpm update --interactive` で定期更新する運用にする
- catalog の更新を自動化したいなら Renovate を併用する（Renovate は catalog プロトコルを認識する）
- もしくは catalog を使わず、各 `package.json` に直接バージョンを書く運用に倒す

最低限、リポジトリの README か CLAUDE.md に「catalog バージョンは Dependabot 管轄外」と明記しておく。

## Expo / React Native プロジェクト固有の注意

Expo SDK は `expo`, `react-native`, `expo-*`, `react-native-*` などのバージョン互換を SDK バージョンに紐づけて管理している。Dependabot がこれらを勝手に更新すると、SDK と齟齬が出てネイティブビルドが壊れる。

**推奨設定**: SDK 配下のパッケージは `ignore` で除外し、`expo install --check` または `nlx expo-doctor` で SDK 整合性を保ったまま手動更新する:

```yaml
- package-ecosystem: "npm"
  directory: "/apps/mobile"
  schedule:
    interval: "weekly"
  ignore:
    # Expo SDK に紐づくパッケージは expo install で更新する
    - dependency-name: "expo"
    - dependency-name: "expo-*"
    - dependency-name: "react-native"
    - dependency-name: "react-native-*"
    - dependency-name: "@expo/*"
  open-pull-requests-limit: 3
```

**ignore せずに groups で1 PR に集約する案**もある（マイナー/パッチだけまとめて、SDK との整合は CI で `expo-doctor` を走らせて検知）。プロジェクトの厳格度に応じて選ぶ。

## groups で PR を集約する

v2 (2024年〜) で導入された groups 機能を使うと、複数 dependency の更新を1 PR にまとめられる。レビュー負担と CI 実行コストを下げる効果が大きい。

```yaml
groups:
  # 開発依存はまとめて1 PR
  dev-deps:
    dependency-type: "development"
    update-types: ["minor", "patch"]

  # 型定義はまとめて1 PR
  types:
    patterns: ["@types/*"]

  # TanStack ファミリーをまとめる
  tanstack:
    patterns: ["@tanstack/*"]

  # 本番依存の minor/patch をまとめる
  production:
    dependency-type: "production"
    update-types: ["minor", "patch"]
```

メジャー更新は意図的に group の外に置く: メジャーは破壊的変更を含むため個別 PR でレビューしたい。

## Security Updates の扱い

Dependabot の **security updates は `dependabot.yml` の設定とは独立**して動く。GitHub Security Advisories に該当する脆弱性が検知されたら、`open-pull-requests-limit: 0` であっても自動 PR が来る。

- リポジトリの **Settings → Code security and analysis** で「Dependabot security updates」が ON になっていることを確認
- 設定そのものは UI でしか変更できないため、`gh` CLI で確認する場合:

```bash
gh api repos/:owner/:repo/vulnerability-alerts -H "Accept: application/vnd.github+json"
```

## auto-merge の連携（オプション）

patch / 開発依存などリスクの低い PR は自動マージしたい場合、`.github/workflows/dependabot-auto-merge.yml` を追加する:

```yaml
name: Dependabot auto-merge
on:
  pull_request:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

jobs:
  auto-merge:
    if: github.actor == 'dependabot[bot]'
    runs-on: ubuntu-latest
    steps:
      - name: Dependabot metadata
        id: meta
        uses: dependabot/fetch-metadata@v2
      - name: Enable auto-merge for patch updates
        if: steps.meta.outputs.update-type == 'version-update:semver-patch'
        run: gh pr merge --auto --squash "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**前提**:
- リポジトリの **Settings → General → Pull Requests → Allow auto-merge** が ON
- ブランチ保護ルールで required checks が設定済み（CI 通過後にのみマージされる）

**注意**: `dependabot[bot]` の PR は GitHub Actions の通常権限ではマージできない。上記のように `permissions:` を明示するか、リポジトリ設定で「Allow GitHub Actions to create and approve pull requests」を有効化する。

メジャー更新の自動マージは推奨しない。破壊的変更を含む可能性が高く、人間のレビューを通すべき。

## Renovate からの移行（migrate-from-renovate モード）

1. 既存 `renovate.json` 系設定を読み、以下を Dependabot 設定に翻訳:
   - `packageRules` → `groups` / `ignore`
   - `schedule` → `schedule.interval` + `schedule.day`
   - `dependencyDashboard` → Dependabot にはダッシュボード機能はない（PR 一覧で代替）
   - `automerge` → 上記 auto-merge workflow
2. Dependabot 設定を有効化し、最初の数 PR が期待通り出ることを確認
3. ユーザー確認のうえで Renovate 設定ファイルと GitHub App を停止
4. CLAUDE.md / README.md の「依存更新運用」記述を Dependabot 前提に書き換え

**翻訳できない機能**: `postUpgradeTasks`、複雑な `extends` チェーン、catalog の自動認識など。これらが運用上必須なら Renovate を残すべき。

## CLAUDE.md への追記

```markdown
## 依存更新（Dependabot）

- 毎週月曜に Dependabot が PR を出す
- patch / dev deps の minor は CI 通過後に auto-merge される
- メジャー更新は人間レビュー必須
- セキュリティ脆弱性は設定とは独立して自動 PR が来る
- pnpm catalog のバージョンは Dependabot 管轄外。`pnpm update --interactive` で手動更新
- Expo SDK 紐づきパッケージ（expo-*, react-native-*）は `expo install` で SDK 整合性を保って更新（apps/mobile を含む場合）
```

## Final Verification

```bash
# 設定ファイルの構文チェック（GitHub 側で行われるが、ローカルで yaml 妥当性を確認）
yq . .github/dependabot.yml

# リポジトリの dependabot 設定を確認
gh api repos/:owner/:repo/dependabot/secrets 2>&1 | head -5
```

設定変更が反映されるのは push 後。GitHub UI の **Insights → Dependency graph → Dependabot** タブで「最後の実行」を確認できる。手動再実行は **Check for updates** ボタンから。

## Important Notes

- 既存の `.github/dependabot.yml` は無断で上書きしない（モード判定の上書きルールを参照）
- security updates は設定とは独立。`open-pull-requests-limit: 0` でも脆弱性 PR は来る
- pnpm catalog 利用時は更新が Dependabot で完結しない。README/CLAUDE.md に明記し、手動運用フローを併設する
- Expo プロジェクトでは Expo SDK 紐づきパッケージを `ignore` するか、専用 group + `expo-doctor` CI チェックの組み合わせで整合性を担保する
- auto-merge は patch + dev minor までに留め、メジャーは人間レビュー必須。脆弱性対応 PR の auto-merge はリスクと利便性のトレードオフで判断する
- モノレポでは `directory` を実在するパッケージ単位で全て宣言する。Dependabot はサブディレクトリの lockfile を自動探索しない
- Renovate との併用は技術的に可能だが、PR が二重に来るので運用が煩雑になる。基本はどちらか一方に倒す
