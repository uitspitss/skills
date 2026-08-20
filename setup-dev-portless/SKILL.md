---
name: setup-dev:portless
description: portless を設定し、ローカル開発を安定した .localhost URL + HTTPS + 自動ポート割当にする。複数 worktree で dev サーバーを同時起動してもポートが衝突しなくなる。Vite / wrangler / Expo と worktrunk の連携を含む。
disable-model-invocation: true
---

# Portless Development Environment Setup

portless は開発サーバーのポート番号を安定した `.localhost` URL に置き換えるツール。複数の git worktree を同時に `nr dev` しても**ポートが衝突しない**。

## What portless does

- `localhost:3000` → `https://web.localhost:1355`（覚えやすい）
- ポートは自動割当（4000〜4999）→ 衝突なし
- HTTPS/HTTP2 対応（自動証明書生成、ポート 1355）
- **Git worktree 自動検出**: `portless run --name web` は worktree のブランチ名をサブドメインに自動付加

## Key Commands

| コマンド | 説明 |
|----------|------|
| `portless proxy start --https` | HTTPS プロキシをデーモン起動（初回のみ） |
| `portless run --name <name> <cmd>` | worktree プレフィックス付きで起動 |
| `portless get <name>` | サービスの URL を取得（クロスサービス参照用） |
| `portless list` | アクティブなルート一覧 |
| `PORTLESS=0 <cmd>` | portless を一時的にスキップ |

**重要**: `portless <name> <cmd>` は worktree プレフィックスが**付かない**。worktree 対応には必ず `portless run --name <name> <cmd>` を使う。

## Prerequisites

```bash
# portless がインストールされているか確認
which portless

# 未インストールの場合（グローバルインストール、プロジェクト依存には追加しない）
npm install -g portless
```

## Setup Steps

### 1. portless プロキシの起動（初回のみ）

```bash
portless proxy start --https
```

これにより:
- ポート 1355 で HTTPS/HTTP2 プロキシがデーモン起動
- 自動生成された CA 証明書がシステムに信頼登録される
- 以降、`https://*.localhost:1355` でアクセス可能
- PC 再起動後は再実行が必要

### 2. アプリの dev スクリプトを portless でラップ

各アプリの `package.json` の `dev` スクリプトで `portless run --name <name>` を使う。

**apps/web/package.json:**
```json
{
  "scripts": {
    "dev": "portless run --name web vite"
  }
}
```

**apps/api/package.json:**
```json
{
  "scripts": {
    "dev": "nr migrate:local; portless run --name api -- sh -c 'wrangler dev --local --persist-to .wrangler/state --port $PORT'"
  }
}
```

portless は Vite を自動検出し `--port` `--host` `--strictPort` フラグを自動注入する。**ただし wrangler は PORT 環境変数を無視するため**、`sh -c '... --port $PORT'` でシェル経由で明示的に渡す必要がある。

子プロセスには以下の環境変数が設定される:
- `PORT` — 自動割当されたポート番号
- `HOST` — `127.0.0.1`
- `PORTLESS_URL` — このサービスの公開 URL

**wrangler.jsonc の `dev.port` は削除する**（固定ポートが portless のポート割当と衝突するため）。

### 3. Vite のプロキシ設定を portless 対応に変更

`portless get api` を使って API の URL を動的に取得する。portless 未使用時は従来のポートにフォールバック。

```ts
import { execFileSync } from 'node:child_process';

function resolveApiTarget(): string {
  // portless 使用時は portless get で API の URL を取得
  if (process.env.PORT) {
    try {
      return execFileSync('portless', ['get', 'api'], { encoding: 'utf-8' }).trim();
    } catch {}
  }
  // フォールバック: portless 未使用時
  return 'http://localhost:8080';
}
```

vite.config.ts の `server.proxy` で使用:

```ts
const apiTarget = resolveApiTarget();

export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: apiTarget,
        changeOrigin: true,
        secure: false, // portless の自己署名証明書を許可
      },
      // 他のプロキシルールも同様に target を apiTarget に変更
    },
  },
});
```

`portless get api` は worktree プレフィックスと HTTPS を自動解決するため、手動でのブランチ名検出は不要。

### 4. wrangler の CORS 設定を動的にする

wrangler.jsonc の `vars.CORS_ORIGIN` はハードコードされている。`portless get web` で Web の URL を取得し、`.dev.vars` で上書きする。

**`scripts/setup-portless-env.sh` を作成:**

```bash
#!/bin/bash
# portless 使用時の .dev.vars を生成する

# worktrunk フックでは mise shim が PATH に無い場合がある
eval "$(mise activate bash 2>/dev/null)" || true

if ! command -v portless &>/dev/null; then
  echo "portless not available, skipping .dev.vars generation"
  exit 0
fi

WEB_URL=$(portless get web 2>/dev/null)

if [ -z "$WEB_URL" ]; then
  echo "portless running but web service not registered, skipping .dev.vars generation"
  exit 0
fi

cat > apps/api/.dev.vars <<EOF
CORS_ORIGIN=${WEB_URL}
BETTER_AUTH_URL=${WEB_URL}
EOF

echo "Generated apps/api/.dev.vars: CORS_ORIGIN=${WEB_URL}"
```

```bash
chmod +x scripts/setup-portless-env.sh
```

`.dev.vars` は wrangler がローカル開発時に自動で読み込む（デプロイには影響しない）。

### 5. .gitignore に追加

```
# portless で動的生成されるローカル設定
apps/api/.dev.vars
```

### 6. worktrunk の pre-start フックに追加

`.config/wt.toml` を更新して、worktree に入るたびに portless 環境をセットアップ:

```toml
[[pre-start]]
trust = "mise trust"

[[pre-start]]
install = "ni"

[[pre-start]]
portless = "bash scripts/setup-portless-env.sh"
```

`[[pre-start]]`（pipeline 形式）はブロック単位で直列実行される。`trust` → `install` → `portless` はこの順序に依存するため、**テーブル形式 `[pre-start]` で書いてはいけない**（テーブル形式は並列実行）。詳細は `setup-dev:tooling` スキルの worktrunk の節を参照。

### 7. ルート package.json にスクリプトを追加（オプション）

```json
{
  "scripts": {
    "setup": "ni && nr types:generate && nr db:migrate && bash scripts/setup-portless-env.sh",
    "portless:env": "bash scripts/setup-portless-env.sh"
  }
}
```

## URL スキーム

portless はブランチ名の `feat/` 等のプレフィックスを除去し、`/` を `-` に変換する。

| 場所 | メイン worktree | feat/design ブランチの worktree |
|------|----------------|-------------------------------|
| Web | `https://web.localhost:1355` | `https://design.web.localhost:1355` |
| API | `https://api.localhost:1355` | `https://design.api.localhost:1355` |
| API (Vite proxy 経由) | `https://web.localhost:1355/api/*` | `https://design.web.localhost:1355/api/*` |

## トラブルシューティング

### Safari で .localhost が解決されない
```bash
PORTLESS_SYNC_HOSTS=1 portless proxy start --https
```
または `sudo portless hosts sync` を実行。Safari は `.localhost` サブドメインの自動解決に非対応のため `/etc/hosts` への同期が必要。

### portless 未使用時のフォールバック
- vite.config.ts: `PORT` 環境変数がなければ `http://localhost:8080` にフォールバック
- wrangler: `.dev.vars` がなければ wrangler.jsonc の `vars` がそのまま使われる
- `PORTLESS=0 nr dev` で portless を一時的に無効化できる

### 証明書エラー（Vite proxy）
proxy 設定に `secure: false` を追加する。Node.js が portless の自己署名証明書を信頼しない場合がある。

### wrangler がポートを固定する
wrangler は PORT 環境変数を無視する。対策:
1. wrangler.jsonc から `dev.port` を削除する
2. dev スクリプトで `sh -c 'wrangler dev --port $PORT'` のようにシェル経由で PORT を渡す
3. この2つの対策を両方行うこと

### プロキシが起動していない
```bash
portless proxy start --https
```
PC 再起動後は再実行が必要。自動起動したい場合は launchd/systemd で設定。

## CLAUDE.md への追記

portless セットアップ後、CLAUDE.md に以下を追記:

```markdown
## 開発サーバー（portless）

- portless proxy が起動していることを確認: `portless proxy start --https`
- `nr dev` で全アプリを同時起動（portless 経由）
- アクセス URL: `portless list` で確認
- worktree ではブランチ名がサブドメインに付加される（例: `design.web.localhost:1355`）
- portless を無効化: `PORTLESS=0 nr dev`
```

## Important Notes

- portless proxy はシステム全体で1つ起動すれば OK（全 worktree で共有）
- portless はグローバルインストール。プロジェクトの devDependencies には追加しない
- portless が未インストールの環境でも従来の `localhost:PORT` で動作するようフォールバックを維持する
- ワイルドカードサブドメイン対応: `tenant.web.localhost` も `web` にルーティングされる
