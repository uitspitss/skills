---
name: setup-dev:e2e
description: @playwright/test で E2E をセットアップする。playwright.config.ts、Vitest とのテストファイル衝突、setup project + storageState による認証、使い捨て E2E DB、CI の別ジョブ化。Next.js / Vite SPA / SSR / モノレポに対応。他の setup-dev スキルから参照される。
disable-model-invocation: true
---

# E2E Setup (@playwright/test)

## `setup-dev:testing` / `setup-dev:storybook` との境界

3つとも「テスト」だが役割が違う。**入れる前にどれの話かを決めること。**

| | 何を起動するか | 何を確かめるか |
| --- | --- | --- |
| `setup-dev:testing` (Vitest / happy-dom) | 何も起動しない | 純粋なロジック、DOM 構造 |
| `setup-dev:storybook` (Vitest browser mode) | Chromium。**サーバーは起動しない** | 1コンポーネントの見た目とインタラクション |
| **このスキル** (`@playwright/test`) | Chromium **＋ アプリのサーバー** | ページ遷移、サーバー側の処理、認証、DB を貫く経路 |

**Vitest browser mode で E2E を書かないこと。** ブラウザは動くがサーバーは動かないので、
「dev サーバーを別途上げておく」という暗黙の前提が要る。`webServer` / `trace` / `retries` /
`shard` / `storageState` を全部自前で組み直すことになり、得るものが無い。

逆に、**サーバーが要らないものを E2E に書かないこと**。E2E は unit の100倍遅く、
100倍壊れやすい。「このテストはサーバーが落ちていても意味があるか？」が判断基準。

---

## 見た目の確認は E2E の仕事ではない

**コンポーネントの見た目は `setup-dev:storybook`。** E2E は「経路が通るか」を見る。
両者を混ぜると、スクリーンショットを撮るためだけにサーバー・DB・LLM を起動する
羽目になり、しかも「ツール実行中」「アップロード失敗」のような途中状態は作りにくい。
Storybook なら props で直接与えられる。

E2E でのスクリーンショットは**失敗したときの証跡**（`trace: "on-first-retry"` で
自動的に残る）であって、確認手段ではない。全画面の撮影スクリプトを E2E に置かないこと。

同じ理由で、**開発者のブラウザを自動操作して見た目を確かめない。** Playwright も
Storybook（`@storybook/addon-vitest`）もヘッドレスで完結する。人のブラウザの
セッションやタブに触る必要はない。

---

## ステップ 0: 対象を決める

E2E に回すのは次のどれかに当たるものだけ。

- **複数ページをまたぐ**（一覧 → 詳細、フォーム送信 → リダイレクト）
- **サーバーでしか起きない**（SSR / ISR の出力、route handler、リダイレクト、Cookie）
- **`render()` できない**（Next.js の `async` な Server Component がその代表）
- **認証状態で分岐する**

逆に E2E に書かない:

- 表示のバリエーション → Storybook
- バリデーションの分岐網羅 → unit
- エラーメッセージの文言 → unit

**最初に書くのは 3〜5 本。** 網羅しようとすると必ず腐る。

### 要らないステップは飛ばす

以降は Next.js + Postgres + better-auth のような**一番重い構成**を前提に書いてある。
アプリがそこまで持っていないなら、該当するステップごと飛ばすこと。

| アプリの性質 | 飛ばすもの |
| --- | --- |
| 認証が無い | ステップ4（`setup` project / `storageState` / auth fixture）を丸ごと |
| DB が無い（静的 JSON、外部 API のみ） | ステップ5 を丸ごと。`prepare-db` も `test:e2e` の前段も要らない |
| ページが1枚だけ | 「ページ遷移」は対象にならない。URL ↔ 状態（クエリパラメータ）とサーバー配信が対象 |

**要らない足場を先に置かないこと。** 空の `setup` project や使われない `globalSetup` は、
後から読む人に「どこかで認証しているはず」と誤読させる。

ページが1枚のアプリでも E2E に意味はある。**Vitest はサーバーを起動しないので、
「ビルドしたものがサーバーから配られ、hydration が通り、URL が状態に反映される」経路は
誰も見ていない**。そこが対象になる。

---

## ステップ 1: インストール

```bash
ni -D @playwright/test
na exec playwright install --with-deps chromium
```

### `playwright` と `@playwright/test` は別のパッケージ

`setup-dev:storybook` を入れていると、`@vitest/browser-playwright` が **無印の `playwright`** を
devDependencies に持ち込んでいる。これは**ライブラリ本体で、テストランナーではない**。
`@playwright/test` を入れないと `import { test } from "@playwright/test"` が解決できない。

```jsonc
// 両方入っているのが正常な状態
"playwright": "^1.62.1",        // Storybook（Vitest browser mode）が使う
"@playwright/test": "^1.62.1",  // E2E ランナー
```

**バージョンは揃えること。** ずれるとブラウザバイナリを2世代ダウンロードすることになり、
CI の `playwright install` がどちらの世代を入れたのか分からなくなって
`Executable doesn't exist at .../chromium-1234/` で落ちる。Dependabot が片方だけ上げてくると
これが起きるので、**両方を同じ group にまとめる**（`setup-dev:dependabot` 参照）。

#### バージョンを揃えた直後にロックファイルを壊しやすい

`ni -D @playwright/test@1.62.1` のようにピンで入れると、パッケージマネージャによっては
package.json に**完全固定**（`"1.62.1"`）で書かれる。無印 `playwright` の `^1.62.1` に
合わせようと**手で `^` を足すと、ロックファイルに記録された指定とずれる**。

ローカルでは何も起きない。落ちるのは CI の `nci`（= `npm ci` / `pnpm install --frozen-lockfile` /
`bun install --frozen-lockfile`）で、「lockfile が package.json と一致しない」という
**インストール段階のエラー**として出る。
E2E の設定を疑って時間を溶かす。

**package.json のバージョン指定を手で編集したら、必ずインストールし直してから commit する。**

#### pnpm catalog がある構成ではピン指定で入れない

`pnpm-workspace.yaml` に `catalog:` があると、pnpm はインストール時にパッケージを
自動で catalog に登録する。このとき **`ni -D @playwright/test@1.62.1` のように
`@バージョン` を付けると、それごとキー名になる**。

```yaml
# 壊れた状態。パッケージ名が "@playwright/test@1.62.1" になっている
catalog:
  "@playwright/test@1.62.1": ^1.62.1
```
```
ERR_PNPM_FETCH_404  GET .../@playwright%2Ftest%401.62.1: Not Found
```

**バージョン無しで入れて、catalog の値を後から手で書く**のが確実。
catalog があるなら、そこが「両方を同じバージョンに固定する」場所として最適
（Dependabot の group より確実に1箇所になる）。

```yaml
catalog:
  # 揃えないとブラウザバイナリが2世代入る
  playwright: ^1.62.1
  "@playwright/test": ^1.62.1
```
```jsonc
"playwright": "catalog:",
"@playwright/test": "catalog:",
```

### コマンドは `@antfu/ni` で書く

このスキルの例は `ni` / `nr` / `na` / `nci` で書いてある。これらはプロジェクトの
パッケージマネージャを自動判別するので、**pnpm / npm / yarn では読み替え不要**。
使い分けは `setup-dev:tooling` の「ni / nr / na / nlx の使い分け」を参照。

`webServer.command` の中でも `nr` は使える（Playwright はコマンドをシェル経由で
起動し、`nr` は mise の shim として PATH にある）。

**bun のプロジェクトだけ `na exec` が効かない**（`bun exec` は `node_modules/.bin` を
PATH に載せない）。`na exec playwright install` のような箇所は `bunx` に読み替えるか、
`package.json` の scripts に逃がして `nr` で呼ぶ。

### ブラウザは chromium だけでよい

`playwright install chromium` だけにする。firefox / webkit を足すと CI が3倍遅くなる。
クロスブラウザ差分を見たいのは「E2E が安定してから」で、たいていその日は来ない。

---

## ステップ 2: `playwright.config.ts`

プロジェクトルートに置く。

```ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  // Vitest の include と食い合わないよう、E2E は専用ディレクトリに隔離する（ステップ3参照）
  testDir: "./e2e",

  // CI では .only の消し忘れを落とす
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,

  // ローカルは並列、CI は1本ずつ。DB を共有するテストは並列で壊れる
  workers: process.env.CI ? 1 : undefined,

  reporter: process.env.CI ? [["github"], ["html", { open: "never" }]] : [["list"]],

  use: {
    baseURL: "http://localhost:3100",
    // 失敗して再実行したときだけ trace を残す。常時 on はディスクを食う
    trace: "on-first-retry",
  },

  projects: [
    { name: "setup", testMatch: /.*\.setup\.ts/ },
    {
      name: "chromium",
      use: { ...devices["Desktop Chrome"], storageState: "e2e/.auth/user.json" },
      dependencies: ["setup"],
    },
  ],

  webServer: {
    command: "nr build && nr start",
    // ISR / SSG のページを url に指定しない（ステップ5）
    url: "http://localhost:3100/<動的なパス>",
    // ローカルは既に上がっているサーバーを使い回す。CI では必ず新しく起動する
    reuseExistingServer: !process.env.CI,
    timeout: 240_000,
    stdout: "pipe",
  },
});
```

### `webServer` は dev サーバーではなく本番ビルドを起動する

`next dev` / `vite` を `webServer` に指定すると、初回アクセスのオンデマンドコンパイルで
数秒〜数十秒待たされ、その待ちが**タイムアウトではなく「要素が見つからない」として現れる**。
原因が分からないまま `waitForTimeout` を足す羽目になる。

```ts
// ✗ 遅く、落ち方が分かりにくい
command: "nr dev",

// ✓ ビルドしてから起動する
command: "nr build && nr start",
```

**build を別ステップに切り出さないこと。** ISR / SSG があると、ビルドを実行した環境の
DB の内容がそのまま HTML に焼き込まれる（ステップ5）。`webServer.env` は
`command` 全体に効くので、ここに含めておけばビルドとサーバーが同じ接続先を見る。

ローカルは `reuseExistingServer: true` なので、既にサーバーを上げてあればビルドは走らない。

### 「何を起動すれば本番相当か」は推測せず実測する

フレームワークごとに `start` / `preview` の意味が違う。デプロイ先が特殊（Cloudflare
Workers、Deno Deploy、Lambda）だと**「専用のランタイムで起動しないと駄目なのでは」と
身構えがちだが、たいてい要らない**。手で確かめれば1分で決まる。

```bash
nr build
nr preview --port 3100 &        # or nr start
sleep 3
curl -s http://localhost:3100/ | head -c 300   # SSR 済みの HTML が返るか
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3100/data/foo.json  # 静的アセットが配られるか
```

- **SSR 済みの HTML が返る** → それを `webServer.command` にする。それ以上は要らない
- **空の `<div id="root">` しか返らない** → その `preview` は SSR を配っていない。
  フレームワークのサーバー起動コマンドを探す
- Cloudflare Workers 向けでも、**Worker 固有の機能（Bindings、`env`、Durable Objects）を
  E2E で通したいのでなければ `wrangler dev` は要らない**。起動が遅く、ビルド成果物を
  そのまま配るだけなら `vite preview` で同じものが見られる。
  Bindings 越しの経路をテストするなら `wrangler dev` を `command` にする

TanStack Start（Vite 環境 API で `dist/client` + `dist/server/server.js` を吐く構成）は
`vite preview` がそのまま SSR を配れた。**ただし版によって変わるので、必ず上の curl で
確かめてから採用すること。**

### ポートを開発サーバーとずらす

既定の 3000 のままだと、`nr dev` を上げたまま E2E を回したときに
**開発サーバーに対してテストが走る**。開発用のデータベースを見ているので一見通ってしまい、
E2E 用のデータベースを用意した意味が消える。3100 などにずらして `PORT` を渡す（ステップ5）。

### `stdout: "pipe"` を付ける理由

既定は `"ignore"` で、サーバー側のエラーが**完全に握り潰される**。
500 が返っているのに「要素が見つからない」としか出ない状態になる。

---

## ステップ 3: Vitest と Playwright がテストを取り合う

**これが一番よく踏む事故。**

`setup-dev:testing` の雛形は `include: ["src/**/*.{test,spec}.{ts,tsx}"]`、
Playwright の既定は `**/*.@(spec|test).?(c|m)[jt]s?(x)`。**`.spec.ts` が両方にマッチする。**

起きること:

- Vitest が Playwright のテストを happy-dom で実行 →
  `Playwright Test needs to be invoked via 'npx playwright test'` で落ちる
- Playwright が Vitest のテストを拾う → `describe` などが未定義で落ちる

両方向を塞ぐ。

```ts
// playwright.config.ts — E2E は e2e/ の外を見ない
testDir: "./e2e",
```

```ts
// vitest.config.ts — include を明示的なディレクトリに限定する（e2e/ を含めない）
include: ["{app,components,lib}/**/*.{test,spec}.{ts,tsx}"],
```

`include` が `**/*.test.ts` のように広い場合は `exclude` を足す:

```ts
exclude: ["**/node_modules/**", "**/dist/**", "e2e/**"],
```

**規約として、E2E は `.spec.ts`、unit は `.test.ts` に統一するとさらに安全**。
どちらか一方の設定が壊れても、拡張子で気づける。

### 「0件で緑」を疑う癖をつける

`setup-dev:testing` に書いたのと同じ問題が E2E にもある。`testDir` を間違えても
Playwright は **`No tests found` で exit 0 にはならない**（exit 1 で落ちる）ので Vitest よりマシだが、
`testMatch` を絞りすぎた場合は「setup だけ走って本体が0件」になり得る。
**導入直後に本数を控えておく。**

---

## ステップ 4: 認証（setup project + `storageState`）

ログインを毎テストでやり直すと遅く、壊れやすい。**1回だけ認証して状態をファイルに保存し、
全テストがそれを読み込む**のが Playwright の定石。

```ts
// playwright.config.ts（再掲）
projects: [
  { name: "setup", testMatch: /.*\.setup\.ts/ },
  {
    name: "chromium",
    use: { ...devices["Desktop Chrome"], storageState: "e2e/.auth/user.json" },
    dependencies: ["setup"],
  },
],
```

`e2e/.auth/` は **必ず gitignore する**（ステップ7）。

### 各 project に `testMatch` を必ず書く

上の `projects` をそのまま使うとき、**`chromium` 側の `testMatch` を省かないこと。**
省くとトップレベルの `testMatch` を継承し、`chromium` が `auth.setup.ts` まで拾って
**setup が 2 回走る**。

症状は認証エラーではなく「2 回目のサインアップが失敗する」形で出る。

```
Error: sign-up に失敗 (422): {"code":"USER_ALREADY_EXISTS_USE_ANOTHER_EMAIL"}
```

しかも**本体のテストは storageState を読めているので全部通り、setup だけが赤くなる**
ので、認証まわりを疑って時間を溶かす。

```ts
projects: [
  { name: "setup", testMatch: /.*\.setup\.ts/ },
  {
    name: "chromium",
    testMatch: /.*\.spec\.ts/,   // ← 省かない
    use: { ...devices["Desktop Chrome"], storageState: "e2e/.auth/user.json" },
    dependencies: ["setup"],
  },
],
```

ステップ3の「E2E は `.spec.ts`、unit は `.test.ts`」という規約は、ここでも効く。

### 認証方式ごとの選び方

| アプリの認証 | 取るべき手 |
| --- | --- |
| メール + パスワード | `request.post()` でログイン API を叩き、`request.storageState()` を保存。UI を操作しない |
| OAuth（Google など）**のみ** | 画面を自動操作しない。下記のいずれか |
| OAuth のみ + 認証ライブラリにテスト用 API がある | **それを使う**（下の better-auth の例） |
| OAuth のみ + テスト用 API が無い | モック IdP を立てるか、テスト時だけ有効になるサインイン経路を用意する |

**Google のログイン画面を Playwright で操作しようとしないこと。** bot 検出、2要素、
reCAPTCHA、UI の変更で必ず壊れる。壊れたとき直せない。

### better-auth の場合

better-auth は **公式の `testUtils` プラグイン**を持っている（`better-auth/plugins`）。
`ctx.test.getCookies()` が**正しく署名済みのセッション Cookie** を返すので、
Cookie を手で組み立てる必要がない。

プラグイン自身の docstring が
「**本番の auth config には入れず、テスト専用の auth インスタンスに入れろ**」と言っている。
条件付き spread（`...(isTest ? [testUtils()] : [])`）にすると `ctx.test` の型推論が壊れるとも
明記されている。**別インスタンスを作ること。**

```ts
// e2e/auth-instance.ts
import { betterAuth } from "better-auth";
import { drizzleAdapter } from "better-auth/adapters/drizzle";
import { testUtils } from "better-auth/plugins";
import * as schema from "@/db/schema";
import { db } from "@/lib/db";

/**
 * E2E 専用の auth インスタンス。本番の lib/auth.ts とは別物。
 *
 * Cookie の名前と署名は options から決まるので、**本番側と食い違わせないこと**。
 * secret と（設定しているなら）advanced.cookiePrefix を必ず合わせる。
 */
export const testAuth = betterAuth({
  database: drizzleAdapter(db, { provider: "pg", schema: { ...schema } }),
  plugins: [testUtils()],
});
```

```ts
// e2e/auth.setup.ts
import { mkdir, writeFile } from "node:fs/promises";
import { test as setup } from "@playwright/test";
import { testAuth } from "./auth-instance";

const authFile = "e2e/.auth/user.json";

setup("authenticate", async () => {
  const ctx = await testAuth.$context;

  const user = await ctx.test.saveUser(
    ctx.test.createUser({ email: "e2e@example.test", name: "E2E", emailVerified: true }),
  );

  // baseURL のホスト名と一致させる。ずれると Cookie が送られない
  const cookies = await ctx.test.getCookies({ userId: user.id, domain: "localhost" });

  await mkdir("e2e/.auth", { recursive: true });
  await writeFile(
    authFile,
    JSON.stringify({
      // TestCookie の expires は「未設定なら undefined」。Playwright は数値必須で
      // セッション Cookie は -1 なので、ここで埋める
      cookies: cookies.map((c) => ({ ...c, expires: c.expires ?? -1 })),
      origins: [],
    }),
  );
});
```

踏む順に落とし穴を並べる。

1. **`BETTER_AUTH_SECRET` が webServer 側と一致していないと署名検証に落ちる。**
   症状は「ログインしていない扱いになる」だけでエラーは出ない。dotenvx 構成では
   Playwright 自体も `dotenvx run --` 越しに起動する必要がある（ステップ5）。
2. **`domain` は `baseURL` のホスト名と揃える。** 既定は `baseURL` から取るが、
   設定によっては本番ドメインが入る。明示するのが安全。
3. **`getCookies` は呼ぶたびに新しいセッションを作る**（実装がそうなっている）。
   setup で一度だけ呼ぶ。
4. **`user.additionalFields` に `input: false` を付けたフィールドは `createUser` の
   overrides から落ちることがある。** handle のようなフィールドを初期値付きで欲しい場合は、
   `saveUser` の後に自前で `db.update()` するか、そのフィールドを設定する画面自体を
   テスト対象にする。
5. `storageState` を使うテストと使わないテストを混ぜたい場合は、
   未ログイン側で `test.use({ storageState: { cookies: [], origins: [] } })` を書く。

#### DB が Cloudflare の binding のときは `testUtils` を使えない

`testUtils` は **Node のプロセスから DB に直接繋げる前提**。D1 は Worker の中にしか
無いので、Playwright 側から `betterAuth({ database: drizzleAdapter(...) })` を
組み立てられない（`env.DB` が存在しない）。

メール + パスワードなら上の表どおり **API を叩くのが正解**。UI は操作しない。

```ts
// e2e/auth.setup.ts
setup("authenticate", async ({ request, baseURL }) => {
  const res = await request.post("/api/auth/sign-up/email", {
    data: { email: "e2e@example.test", password: "e2e-password-1234", name: "E2E" },
    // Better Auth は Origin 無しのリクエストを弾く（下記）
    headers: { origin: baseURL ?? "" },
  });
  if (!res.ok()) throw new Error(`sign-up に失敗 (${res.status()}): ${await res.text()}`);

  await mkdir("e2e/.auth", { recursive: true });
  await request.storageState({ path: "e2e/.auth/user.json" });
});
```

**Better Auth は `Origin` ヘッダが無いリクエストを 403 で弾く。**

```
{"message":"Missing or null Origin","code":"MISSING_OR_NULL_ORIGIN"}
```

ブラウザは自動で付けるが、**`APIRequestContext` も Node の `fetch` も付けない**ので、
E2E の setup と開発用の seed スクリプトの両方で明示が要る。値はサーバー側の
`trustedOrigins` に含まれている必要がある。

### 認証ライブラリにテスト用 API が無い場合

「テストのときだけ有効になるサインイン経路」を足すのは**最後の手段**。やるなら:

- 環境変数で明示的に開き、**既定は閉じる**（`if (process.env.E2E_TEST_MODE !== "1") return new Response(null, { status: 404 })`）
- 本番のビルドに含まれないよう、フラグの立て忘れではなく**フラグの立ち忘れ**で壊れる向きにする
- README に「この経路が本番で開いていないこと」の確認手順を書く

---

## ステップ 5: テスト用の DB

**開発用のデータベースを E2E に使わないこと。** E2E は必ず書き込む。削除経路の無いデータ
（イベントログ、不変なレコード）を作るアプリだと開発用が汚れ続け、
しかも「開発中に手で作ったデータ」に依存したテストが書けてしまう。

### Cloudflare Workers（D1 / R2 / Durable Objects）の場合

コンテナは要らない。`wrangler` の**状態ディレクトリを分ける**のが「データベースを分ける」
に相当する。`--persist-to` は D1 も R2 も Durable Object もまとめて隔離する。

```ts
// playwright.config.ts
const PORT = 8788;                        // 開発の 8787 / 5173 とずらす
const STATE_DIR = ".wrangler/e2e-state";  // 開発用の .wrangler/state とは別

webServer: {
  command: [
    // assets.directory が解決できないと wrangler は起動時に落ちる
    "pnpm --filter @repo/web run build",
    // 毎回まっさらから始める。テストが自分でデータを作る前提にできる
    `rm -rf apps/server/${STATE_DIR}`,
    `pnpm --filter @repo/server exec wrangler d1 migrations apply DB --local --persist-to ${STATE_DIR}`,
    `pnpm --filter @repo/server exec wrangler dev --port ${PORT} --persist-to ${STATE_DIR} --var BETTER_AUTH_URL:http://localhost:${PORT}`,
  ].join(" && "),
  cwd: "..",
  url: `http://localhost:${PORT}`,
  reuseExistingServer: !process.env.CI,
  stdout: "pipe",
},
```

**`vite preview` ではなく `wrangler dev` を使う判断基準**は「bindings 越しの経路が
テスト対象か」。認可が D1、ファイルが R2、チャットが Durable Object のように
**bindings そのものが仕様**なら `wrangler dev` が要る。本番も同じ Worker が
SPA を assets として配るなら、それがそのまま本番相当になる。

**ポートが解放されているか先に確かめる。** `wrangler dev` が二重起動すると

```
Fatal uncaught kj::Exception: ::bind(...): Address already in use; 127.0.0.1:8787
```

で新しい方が死ぬが、**古い方が生き続けるのでテストは「通ってしまう」**。
古いコードに対して緑になるという最悪の形なので、E2E の前に `lsof -ti:<port>` を見る。

### LLM を呼ぶアプリは決定的なダミーに差し替える

実物の推論 API を E2E から呼ばない。遅いだけでなく**壊れ方が理不尽**になる。
Workers AI の binding は remote 接続なので、マルチステップのツール呼び出しの
2 ステップ目で切れることがある。

```
url: 'workers-ai:binding/run/@cf/...'
responseBody: 'Error: Network connection lost.'
isRetryable: false
```

UI 上は「1 回目のツール呼び出しで止まったまま」に見えるので、**自分の直前の変更を
疑って調べ回る**ことになる。

モデル選択を 1 箇所に閉じてあるなら（`infrastructure/ai/model.ts` 等）、そこに
環境変数で開く分岐を足す。**既定は閉じ、動的 import にして本番バンドルに入れない。**

```ts
export async function selectModel(env: Env): Promise<LanguageModel> {
  if (env.E2E_FAKE_LLM === "1") {
    const { createFakeModel } = await import("./fake-model");  // 本番には含まれない
    return createFakeModel();
  }
  // ...本物
}
```

ダミーは AI SDK の `MockLanguageModelV4` + `simulateReadableStream`（`ai/test`）で書ける。
ステップごとに違うチャンク列を返せば、ツール呼び出しのループまで決定的に再現できる。

### コンテナは分けない。データベースを分ける

**まず「別コンテナが要るか」を疑うこと。** 必要なのは「開発用のテーブルを汚さない」ことで、
それは**同じコンテナの別データベース**で足りる。E2E の前処理が毎回テーブルを空にするなら、
コンテナを使い捨てにして得られるものは無い。

コンテナを分ける価値があるのは、Postgres のバージョンや設定を変えたいとき、
あるいは開発用を止めたまま E2E を回したいときだけ。それ以外では、
起動するコンテナが増え、ポートが増え、`nr db:up` の意味が変わる分だけ損をする。

```
postgres://user:pass@localhost:5432/myapp        # 開発用
postgres://user:pass@localhost:5432/myapp_e2e    # E2E
```

#### 埋め込み DB（Cloudflare D1 / miniflare, SQLite）なら「状態ディレクトリ」を分ける

サーバープロセスが無いので、分ける単位はデータベースではなく**ファイルの置き場**になる。
D1 なら `wrangler dev` / `wrangler d1 execute` の `--persist-to` を差し替えるだけで足りる。

```
apps/api/.wrangler/state       # 開発用（wrangler dev の既定）
apps/api/.wrangler/e2e-state   # E2E
```

`create database` も拡張も要らないので、前処理は**ディレクトリを消して SQL を流し直す**だけ。
CI に Postgres の `services` を置く必要も無くなる。

```ts
// e2e/prepare-db.ts
rmSync(resolve(API_DIR, E2E_PERSIST_TO), { recursive: true, force: true });

for (const file of E2E_SQL_FILES) {
  execFileSync(
    "na",
    ["exec", "wrangler", "d1", "execute", "DB", "--local",
     `--persist-to=${E2E_PERSIST_TO}`, `--file=${file}`],
    // wrangler.jsonc の解決も --persist-to も worker のディレクトリを起点にする
    { cwd: API_DIR, stdio: "inherit" },
  );
}
```

#### ディレクトリごと作り直すなら `reuseExistingServer: false` にする

**これは Postgres では起きず、ファイルを消す構成でだけ起きる。**
`reuseExistingServer: !process.env.CI` のまま前回のサーバーが生きていると、
前処理が消した **inode を掴んだまま**動き続ける。新しく作られたファイルは見ないので、
「migration を流したのにテーブルが無い」「seed を入れたのに一覧が空」という形で出る。

DB を作り直す側の `webServer` だけ `false` にする。ポートが埋まっていれば
Playwright が起動エラーで止まるので、黙って壊れるより良い。

```ts
{
  command: "... wrangler dev --persist-to .wrangler/e2e-state ...",
  // db:e2e:prepare が --persist-to のディレクトリごと作り直すので使い回さない
  reuseExistingServer: false,
}
```

ビルドして配るだけの web 側は状態を持たないので `!process.env.CI` のままでよい。

### データベースは前処理で作る。compose の init に置かない

`docker-entrypoint-initdb.d` の SQL は**ボリュームが空のときしか流れない**。
そこに `create database ..._e2e` を書くと、**既に開発環境を持っている人の手元では
永遠に作られない**。「自分の環境では動くのに、他の人は `database does not exist` で落ちる」
という形で出る。

前処理スクリプト側で「無ければ作る」をやると、環境の状態に依らず動く。

```ts
import { Client } from "pg";

// postgres には `create database if not exists` が無いので存在を確かめてから作る
const admin = new Client({ connectionString: `${PG}/postgres` });
await admin.connect();
const found = await admin.query("select 1 from pg_database where datname = $1", [E2E_DB_NAME]);
if (found.rowCount === 0) {
  // 識別子は値としてバインドできない。定数なので埋め込んでよい
  await admin.query(`create database "${E2E_DB_NAME}"`);
}
await admin.end();
```

`postgres` データベースは必ず存在するので、接続先に使える。

**拡張はデータベース単位。** `pgcrypto` などを開発用に作ってあっても、
新しいデータベースには無い。前処理で作る。

```ts
const db = drizzle(E2E_DATABASE_URL);
await db.execute(sql`create extension if not exists pgcrypto`);
await db.execute(sql`create extension if not exists pg_trgm`);
```

ここまでやると **CI に「拡張を作る」ステップが要らなくなる**（素の Postgres サービスを
置くだけで済む）。ローカルと CI で手順が分岐しないのは、それ自体が価値がある。

### 接続先の差し替え — 平文の `.env.e2e` を作らない

dotenvx を使っている構成では `.env*` が pre-commit の `encrypt-env` に拾われる。
平文で `.env.e2e` を置くと**コミット時に暗号化され、復号キーは gitignore なので CI で
`"encrypted:..."` がそのまま値になる**。

代わりに、E2E 用の値を1ファイルに集約して `playwright.config.ts` から差し込む。
**dotenvx は既にセットされている環境変数を上書きしない**ので、これで勝てる。

```ts
// e2e/env.ts
// E2E 専用の接続先。いずれも本番と無関係なローカル値なので平文で持ってよい。
// CI から差し替えられるよう環境変数を優先する。

/** サーバまでの接続情報（データベース名は含めない）。開発用と同じコンテナを使う */
const PG = process.env.E2E_PG_URL ?? "postgres://<user>:<password>@localhost:5432";

export const E2E_DB_NAME = "<db>_e2e";
/** `create database` を実行するための接続先。postgres データベースは必ず存在する */
export const E2E_ADMIN_DATABASE_URL = `${PG}/postgres`;
export const E2E_DATABASE_URL = `${PG}/${E2E_DB_NAME}`;

export const E2E_AUTH_SECRET =
  process.env.E2E_AUTH_SECRET ?? "e2e-secret-not-used-for-anything-real-0000";
export const E2E_PORT = process.env.E2E_PORT ?? "3100";
export const E2E_BASE_URL = `http://localhost:${E2E_PORT}`;
```

**アプリの db モジュールを E2E から import しないこと。** `process.env.DATABASE_URL` を
import 時点で読む作りが多く、そうなると「環境変数を差し替えてから import する」順序に
依存する。静的 import は巻き上げられるので、この順序は簡単に壊れる。
E2E 側は接続先を直接渡した専用インスタンスを持つ。

```ts
// e2e/db.ts
export const e2eDb = drizzle(E2E_DATABASE_URL);
```

`playwright.config.ts` は**サーバーに渡す分**だけを面倒みればよい。

```ts
// playwright.config.ts
import { E2E_AUTH_SECRET, E2E_BASE_URL, E2E_DATABASE_URL, E2E_PORT } from "./e2e/env";

export default defineConfig({
  use: { baseURL: E2E_BASE_URL },
  webServer: {
    // build も含める。理由は下の「ISR / SSG のページはビルド時の DB で焼き固まる」
    command: "nr build && nr start",
    url: `${E2E_BASE_URL}/<動的なパス>`,
    // ここがずれると Cookie の署名が合わない。env は build と start の両方に効く
    env: {
      DATABASE_URL: E2E_DATABASE_URL,
      BETTER_AUTH_SECRET: E2E_AUTH_SECRET,
      BETTER_AUTH_URL: E2E_BASE_URL,
      PORT: E2E_PORT,
    },
    reuseExistingServer: !process.env.CI,
    timeout: 240_000, // build を含むので長めに
  },
});
```

これで `webServer` の中の `dotenvx run -- next start` は、
既にセットされた `DATABASE_URL` / `BETTER_AUTH_SECRET` を上書きせず、
`BETTER_AUTH_URL` などの残りだけ `.env` から補う。

### migration とデータ投入は Playwright の**外**でやる

**setup project や globalSetup に置かないこと。** どちらも `webServer` が起動した**後**に走る。
これを間違えると2つの形で刺さる。

1. **`webServer.url` のヘルスチェックが先に飛ぶ。** テーブルの無い DB を引いて 500 を返し、
   Playwright は「まだ起動していない」と判断して**タイムアウトまで待ち続ける**。
   `Timed out waiting 120000ms from config.webServer` としか出ないので、
   原因が DB だと分からない
2. **ISR / SSG のページはビルド時と初回リクエスト時の DB で焼き固まる。**
   データが揃う前にサーバーが動くと、空の一覧がキャッシュされたまま返り続ける

`npm scripts` の前段に置けば、順序が確実になる。

```jsonc
"db:e2e:prepare": "tsx e2e/prepare-db.ts",
"test:e2e": "nr db:e2e:prepare && playwright test"
```

```ts
// e2e/prepare-db.ts — データベース作成 → 拡張 → migration → 投入
const db = drizzle(E2E_DATABASE_URL);
await db.execute(sql`create extension if not exists pgcrypto`);
await migrate(db, { migrationsFolder: "./db/migrations" });
await db.delete(childTable); // 外部キーの順に消す
await db.delete(parentTable);
await db.insert(parentTable).values(/* ... */);
```

**drizzle-kit の CLI を噛ませないのが要点。** `drizzle-kit migrate` は `drizzle.config.ts` から
`DATABASE_URL` を読むので、E2E 用の値を渡す経路をもう1本作ることになる。
`migrate()` を直接呼べば、接続先を渡した専用インスタンスをそのまま使える。

### ISR / SSG があるならビルドも `webServer` に入れる

プリレンダされるページは、**ビルドを実行したときの DB の内容が HTML に焼き込まれる**。
CI で `nr build` を別ステップに置くと、そこだけ違う接続先を見て、
「E2E 用のデータを入れたのに一覧に出てこない」という状態になる。

```ts
webServer: {
  command: "nr build && nr start",
  env: { DATABASE_URL: E2E_DATABASE_URL, /* ... */ }, // build と start の両方に効く
}
```

`webServer.url` にも**キャッシュされないパス**を選ぶ。トップページが ISR なら、
ヘルスチェックの結果そのものがキャッシュされる。動的なページを指すこと。

### seed をどう扱うか

**開発用の seed をそのまま E2E に流さない。** seed が変わるたびにテストが落ちる。
E2E が必要とするデータは、そのテスト（か setup）が自分で作る。

書き込み経路が API として存在するなら**それを使う**のが最良。UI もモデルも通るので、
DB 直叩きより実態に近い。

```ts
// e2e/fixtures.ts
export async function createEntry(request: APIRequestContext, token: string, body: object) {
  const res = await request.post("/api/entries", {
    headers: { Authorization: `Bearer ${token}` },
    data: body,
  });
  if (!res.ok()) throw new Error(`createEntry failed: ${res.status()} ${await res.text()}`);
  return res.json();
}
```

### 並列実行と DB の共有

**同じ DB を見るテストを `workers > 1` で走らせない。** 一覧の件数を数えるテストが
他のテストの挿入で落ちる、という形で出る。上の config は CI で `workers: 1` にしてある。
速度が要るようになってから、テストごとにスキーマを分ける等を検討する（たいてい要らない）。

---

## ステップ 6: package.json と CI

### scripts

```jsonc
{
  "db:e2e:prepare": "tsx e2e/prepare-db.ts",
  "test:e2e": "nr db:e2e:prepare && playwright test",
  "test:e2e:ui": "nr db:e2e:prepare && playwright test --ui"
}
```

**`nr test` に E2E を含めないこと。** `nr test` は「速くて毎回回すもの」であってほしい。
E2E はサーバーの起動と DB が要る。lefthook の pre-commit / pre-push にも入れない。

**E2E 用の DB を起動するスクリプトは要らない。** 開発用のコンテナ（`nr db:up`）の
別データベースを使い、そのデータベースは `prepare-db.ts` が無ければ作る。

`dotenvx run --` で包む必要も無い。E2E 側のモジュールが接続先を直接持ち、
アプリのサーバーには `webServer.env` が渡すので、Playwright のプロセス自体は
`.env` を必要としない。

### CI は別ジョブにする

既存の `ci` ジョブに足すと、lint の失敗を待ってから E2E が動くことになり
フィードバックが遅くなる。**独立したジョブにして並列に回す。**

```yaml
  e2e:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:18-alpine
        # E2E 用のデータベースも拡張も prepare-db.ts が作るので、素のままでよい
        env:
          POSTGRES_USER: <user>
          POSTGRES_PASSWORD: <password>
          POSTGRES_DB: <db>
        ports:
          # ローカルと同じ 5432。e2e/env.ts の既定値がそのまま使える
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U <user> -d <db>"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 20

    # dotenvx は CI に復号キーが無いので .env を復号できない。
    # アプリが使う変数を全部埋める（埋め漏らすと暗号文がそのまま値になる）
    env:
      DATABASE_URL: postgres://<user>:<password>@localhost:5432/<db>_e2e
      BETTER_AUTH_SECRET: e2e-secret-not-used-for-anything-real-0000
      BETTER_AUTH_URL: http://localhost:3100

    steps:
      - uses: actions/checkout@v7
      - uses: jdx/mise-action@v2
      - run: nci

      - name: Install Playwright browser
        run: na exec playwright install --with-deps chromium

      # ビルドは webServer が行う。ここで `nr build` しないのは、ISR / SSG の
      # ページがビルド時の DB の内容で焼き固まるため
      - run: nr test:e2e

      - uses: actions/upload-artifact@v4
        if: ${{ !cancelled() }}
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7
```

- **`if: ${{ !cancelled() }}`**。`if: failure()` だと flaky（retry で通った）ケースの
  trace が取れない。
- **`services` はローカルと同じポート・同じ認証情報にする。** そうすれば `e2e/env.ts` の
  既定値がそのまま使え、CI 用の環境変数を1つ減らせる。
- **拡張を作るステップは要らない。** `prepare-db.ts` が「無ければ作る」をやっていれば、
  ローカルと CI で手順が分岐しない。
- ブラウザバイナリのキャッシュ（`actions/cache` で `~/.cache/ms-playwright`）は
  **最初は入れないこと**。キーを `playwright` のバージョンに紐づけ損ねると、
  古いバイナリを復元して `Executable doesn't exist` で落ちる。CI が遅くて困ってから足す。

---

## ステップ 7: 周辺ツールの設定

### .gitignore

```
# Playwright
/test-results/
/playwright-report/
/blob-report/
/playwright/.cache/
e2e/.auth/
```

**`e2e/.auth/` を忘れない。** 有効なセッション Cookie が入っている。

#### monorepo では上のパターンがそのまま効かない

gitignore は**先頭以外にスラッシュを含むパターンを、その .gitignore の位置を起点に
アンカーする**。ルートの .gitignore に `e2e/.auth/` と書いても、それは
`<root>/e2e/.auth/` しか指さず、`apps/web/e2e/.auth/` は**素通しになる**。
`git check-ignore -v` で確かめるまで気づけない。

スラッシュを含むものには `**/` を付け、含まないものは先頭スラッシュを外す。

```
# ルートの .gitignore（monorepo）
test-results/
playwright-report/
blob-report/
**/playwright/.cache/
**/e2e/.auth/
```

先頭スラッシュを外すのは **biome の `vcs.useIgnoreFile` 対策**でもある。
`/apps/web/test-results/` のようにルート起点で書くと、`apps/web` から
`biome check .` を走らせたときに解決されず、**生成物やセッション Cookie が
format エラーとして報告される**（`e2e/.auth/user.json format ━━━` が出たらこれ）。

### monorepo では `e2e/` を workspace パッケージにする

ルート直下の素のディレクトリのままだと、**turbo の `typecheck` / `lint` の対象から
外れる**（turbo はパッケージ単位でしか回らない）。E2E のコードだけ型検査されない
状態になり、しかもエラーが出ないので気づけない。

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
  - "e2e"
```

```jsonc
// e2e/package.json
{
  "name": "@repo/e2e",
  "private": true,
  "scripts": {
    "lint": "oxlint .",
    "typecheck": "tsc --noEmit",
    "test:e2e": "playwright test"
  },
  "devDependencies": { "@playwright/test": "catalog:" }
}
```

`playwright.config.ts` もこのディレクトリに置き、`testDir: "."` にする。
ルートの `package.json` からは `pnpm --filter @repo/e2e run test:e2e` で呼ぶ
（`nr test` には入れない。ステップ6を参照）。

### tsconfig

`include` が `**/*.ts` なら `e2e/` も型検査される（それでよい）。
`src/**` に限定している構成では `e2e` を足す。

```jsonc
"include": ["src/**/*.ts", "e2e/**/*.ts", "playwright.config.ts"]
```

**`@playwright/test` の `expect` と Vitest の `expect` は別物。** 同じファイルに混ぜない。
ステップ3のディレクトリ分離ができていれば自然に混ざらない。

### knip

`playwright.config.ts` は knip の Playwright プラグインが認識するが、
**`e2e/` を `project` に含めないと中の import が追えず、`@playwright/test` が未使用扱いになる。**

```jsonc
{
  "project": ["{app,components,lib,db,e2e}/**/*.{ts,tsx,css}"]
}
```

`e2e/fixtures.ts` のようなヘルパは、テストからしか使われないので
`exports` の未使用検出に引っかかることがある。`ignoreIssues` で `e2e/**` の `exports` を外す。

**Playwright プラグインの既定 entry は `*.spec.ts` だけで、`*.setup.ts` を含まない。**
ステップ4の setup project を入れると `auth.setup.ts` が「未使用ファイル」、そこからしか
使われない定数が「未使用 export」として並ぶ。プラグインの `entry` を明示して足す。

```jsonc
"playwright": {
  "config": ["playwright.config.ts"],
  "entry": ["e2e/**/*.{spec,setup}.ts"]
}
```

### oxlint

追加設定は基本不要。`e2e/` 内で `await expect(...)` を多用するので、
`typescript/no-floating-promises` 系を有効にしている場合は素直に従うこと（付け忘れは実バグ）。

---

## ステップ 8: 最初のテストを書く

```ts
// e2e/timeline.spec.ts
import { expect, test } from "@playwright/test";

test("タイムラインから記事へ移動できる", async ({ page }) => {
  await page.goto("/");

  const first = page.getByRole("link", { name: /麦茶/ }).first();
  await first.click();

  await expect(page.getByRole("heading", { level: 1 })).toContainText("麦茶");
});
```

### 第三者のリソースは止める（アプリのものは止めない）

E2E から本物の YouTube / 地図タイル / Google Fonts / 解析タグを取りに行かせない。
遅く、相手の都合で落ち、CI がネットワークに依存する。しかも**それらが動くかどうかは
このアプリの関心事ではない**。

```ts
// e2e/fixtures.ts
import { test as base } from "@playwright/test";

const EXTERNAL_RESOURCES =
  /(youtube\.com|ytimg\.com|basemaps\.cartocdn\.com|fonts\.googleapis\.com|fonts\.gstatic\.com)/;

export const test = base.extend<{ blockExternalResources: RegExp }>({
  blockExternalResources: [
    async ({ page }, use) => {
      await page.route(EXTERNAL_RESOURCES, (route) => route.abort());
      await use(EXTERNAL_RESOURCES);
    },
    { auto: true },   // 全テストに自動で効く
  ],
});

export { expect } from "@playwright/test";
```

各 spec は `@playwright/test` ではなく `./fixtures` から `test` / `expect` を import する。

**これは「モックに寄せる」ことではない。** 止めてよいのは**第三者**だけで、
アプリのサーバーが配るもの（`/data/*.json`、自前の API、静的アセット）は**絶対に止めない**。
そこを止めた瞬間、E2E は「遅い Storybook」になる。判断基準は
「これはこのリポジトリがデプロイするものか？」。

#### `void` の auto fixture は Biome に怒られる

Playwright の公式例は `base.extend<{ x: void }>` だが、Biome の `noConfusingVoidType` が
警告を出す。**提案される `undefined` への置換（unsafe fix）を鵜呑みにしないこと** —
`use()` を引数無しで呼べなくなって今度は型エラーになる。
上のように**意味のある値を `use()` に渡す**のが素直（パターン自体を渡しておけば、
特定のテストだけブロックを外したいときに使える）。

### ロケータの選び方

**`getByRole` を第一候補にする。** 次点で `getByLabel` / `getByText`。
CSS セレクタと `data-testid` は最後の手段。

- `page.locator(".card > div:nth-child(2)")` は Tailwind のクラスを1つ変えただけで壊れる
- `getByRole` で取れないなら、**それはアクセシビリティの不備**であることが多い。
  テストのために `data-testid` を足す前に、見出しレベルやラベルが正しいかを疑う

#### role を持たない描画物（地図・Canvas・チャート）

Leaflet の `divIcon`、Canvas に描かれたもの、チャートライブラリが吐く要素には
role が無い。ここは例外で、**アプリが既に持っている意味のある属性**で取る。

```ts
// アプリ側のコードに元からある属性。テストのために足したものではない
page.locator('[data-broadcast-name="日本テレビ放送網"]')
```

- **テストのためだけに `data-testid` を足さない。** 描画に使っている属性が既にあるなら
  それを使う。無いなら、まず「そのマーカーは本当にキーボードで操作できなくてよいのか」を疑う
- 同じ文字列がパネルのタイトルなどにも出るなら `getByText` は多重マッチする。
  属性セレクタのほうが曖昧さが無い
- **見つけたキーボード操作不能は E2E で直さない。** 範囲外として報告に残し、
  別の変更で扱う（E2E 導入の PR に a11y 修正を混ぜない）

#### 地図・絶対配置の要素はビューポートの外だとクリックできない

Playwright は要素が見えるまで自動スクロールするが、Leaflet の pane のように
**スクロールで移動しない**領域の外にあるものはクリックできない。
中心座標とズームで対象が画面内に入ることを、最初のテストで確かめておく。

### `waitForTimeout` を書かない

```ts
// ✗ 遅く、それでも不安定
await page.waitForTimeout(1000);
expect(await page.locator("h1").textContent()).toBe("...");

// ✓ web-first assertion が自動で待つ
await expect(page.getByRole("heading", { level: 1 })).toHaveText("...");
```

`expect(locator)` の形（web-first assertion）は条件が満たされるまでリトライする。
`expect(await locator.textContent())` の形は**一度しか見ない**。この2つを混同しない。

### ISR / キャッシュを持つページ

Next.js の ISR や `revalidate` があるページは、**書き込み直後に読んでも古い内容が返る**。
E2E で「投稿 → 一覧に出る」を書くなら、対象ページの revalidate 設定を確認すること。
待てば出るものに `expect` のタイムアウトを伸ばすのは筋が悪い。
`revalidatePath` を叩く経路があるならそれを通す。無いなら、**そのシナリオは E2E に向かない**。

---

## テストを通すために設定を特別扱いしたくなったら止まる

最初のテストは**必ず落ちる**。そこで `webServer.env` の値をいじって通してしまうのが
一番よくある間違いで、しかも緑になるので気づけない。

判断の分かれ目は「**その設定は開発環境と同じか**」の一点。

| 設定を変える理由 | 判定 |
| --- | --- |
| 本番の値（本番 API の URL、本番の DB）を潰す | 正しい。E2E がそこを叩いてはいけない |
| ポート・接続先・シークレットを E2E 用にずらす | 正しい。開発環境と同じ**種類**の値のまま |
| **開発環境では空/未設定の値に、E2E だけ具体的な値を入れる** | **止まる。アプリのバグを迂回している** |

3つ目に当たったら、まずブラウザのコンソールとネットワークを見ること。

```ts
page.on("console", (m) => console.log("[console]", m.type(), m.text()));
page.on("pageerror", (e) => console.log("[pageerror]", e.message));
page.on("requestfailed", (r) => console.log("[reqfail]", r.url(), r.failure()?.errorText));
```

**「リクエストが1本も出ていない」のに読み込み中のまま**なら、fetch に到達する前に
例外が出ていて、それを react-query などのリトライが握り潰している。
`isPending` はリトライ中も true なので、既定の3回が終わるまでエラー表示にすらならない
（＝ `expect` のタイムアウトが短いと「要素が無い」としか分からない）。

**直したら、迂回のために変えた設定は元に戻す。** 開発環境と同じ経路を通っていない E2E は、
その経路が壊れても教えてくれない。

---

## 検証チェックリスト

導入直後に上から順に確認する。

1. `nr db:e2e:prepare` が、**E2E 用データベースが無い状態から**通る
   （`drop database` してもう一度流して確かめる）— DB が無いなら飛ばす
2. `nr test:e2e` で **setup が走り、本体テストの件数が期待どおり**（0件で緑になっていない）
3. `nr test` に E2E が混ざっていない（Vitest の Test Files 件数が導入前と同じ）
4. テストを1つ**わざと壊して** `nr test:e2e` が赤くなる。
   **終了コードは直接見ること** — `playwright test | tail` のようにパイプすると、
   シェルが返すのはパイプ最後のコマンドの終了コードで、**赤なのに 0 に見える**。
   `nr test:e2e > /tmp/e2e.log 2>&1; echo $?` の形で確かめる
5. 認証が要るテストで、`e2e/.auth/` を消してから走らせても setup が作り直す
6. `e2e/.auth/` `test-results/` `playwright-report/` が ignore されている。
   **素の `git status` で確かめない** — 未追跡ディレクトリは `?? apps/web/e2e/` と
   1行にまとまるので、中の `.auth/user.json` が漏れていても見えない。
   `git check-ignore -v <path>` で1件ずつ、あるいは `git status -uall` で確かめる
7. `nr typecheck` / `nr lint` / `nr knip` が通る
8. **開発用のデータベースにテストのデータが入っていない**（件数を実際に数える）
9. **2回続けて**通る（1回目のキャッシュや書き込みが2回目を壊していないか）
10. **ロックファイルが package.json と整合している**（`--frozen-lockfile` で install し直す）。
    バージョン指定を手で直した直後はここが壊れている
11. **ネットワークを切っても通る**（第三者リソースを本当に止められているか）。
    切れないなら、少なくとも外部ドメインへのリクエストが 0 件であることを確かめる
12. CI で e2e ジョブが緑になり、失敗時に playwright-report が artifact に上がる。
    **別ジョブは独自に `playwright install` が要る** — 既存の CI ジョブが
    Storybook 用に chromium を入れていても、ジョブが違えば共有されない

---

## Vite SPA の場合

違いは `webServer` だけ。

```ts
webServer: {
  command: "nr preview --port 3100",
  url: "http://localhost:3100",
  reuseExistingServer: !process.env.CI,
},
```

`vite preview` は `vite build` の出力を配るので、**先に build が要る**（Next と同じ）。
`vite dev` を使わない理由も同じ。

**SSR するフレームワーク（TanStack Start など）でも `vite preview` で足りることが多い。**
ステップ2の curl で「SSR 済みの HTML が返るか」を確かめてから決める。

### `server.proxy` は `vite preview` に効かない

開発時に `/api` をバックエンドへ流している構成でも、**`preview` は別のキーを見る**。
`server.proxy` だけ書いてあると、preview では proxy が無いものとして扱われ、
API へのリクエストが**全部 vite の 404**になる。ページは出るので原因が見えにくい。

```ts
const apiProxy = {
  "/api": { target: apiTarget, changeOrigin: true, secure: false },
};

export default defineConfig({
  server: { proxy: apiProxy },
  // E2E はビルド成果物に対して走るので、ここにも同じ経路が要る
  preview: { proxy: apiProxy },
});
```

同一オリジンになるので、ブラウザから見て CORS は発生しない。ただし
**`Origin` ヘッダは preview の URL のまま転送される**（`changeOrigin` が変えるのは
`Host` だけ）。better-auth の `trustedOrigins` のようにオリジンを検証する仕組みがあるなら、
API 側の設定値を preview の URL に上書きすること（下記）。

### `.env.production` がビルドに焼き込まれる

`vite build` は既定で `mode=production` なので **`.env.production` を読む**。
そこに本番の API URL が入っていると、E2E のビルドがそれを埋め込んで
**テストが本番を叩く**。`.env.e2e` を作るのは（ステップ5のとおり）避けたい。

`loadEnv` は **`.env` ファイルより `process.env` の `VITE_*` を優先する**ので、
`webServer.env` から渡せば勝てる。ファイルを増やさずに済む。

```ts
webServer: {
  command: "nr build && na exec vite preview --port 3100 --strictPort",
  env: {
    // .env.production の値を潰す。build と preview の両方に効く
    VITE_API_URL: "",
  },
},
```

### SPA には「サーバー側の処理」が無い

バックエンドが別サービスなら、E2E の対象は
「フロントが API を正しく叩き、返ってきたものを正しく描くか」になる。ここで分岐する。

- **API も一緒に立てる**（`webServer` は配列で複数指定できる）。本物の結合が見られる
- **`page.route()` で API をモックする**。速くて安定するが、結合は見ていない。
  それなら Storybook の play で十分なことが多い

```ts
// webServer は配列を取れる
webServer: [
  { command: "na --filter api run start", url: "http://localhost:8787" },
  { command: "na --filter web run preview --port 3100", url: "http://localhost:3100" },
],
```

**モックに寄せすぎない。** 全部モックした E2E は「遅い Storybook」でしかない。

---

## monorepo の場合

E2E は**アプリごとに置く**。ルートに1つ置いて `--filter` で切り替える構成にしない
（どのアプリのテストが落ちたのか分からなくなる）。

```
apps/web/playwright.config.ts
apps/web/e2e/
apps/api/          # API だけなら E2E ではなく Vitest + fetch で足りることが多い
```

- `@playwright/test` は**それを使うワークスペースの devDependencies** に入れる。
  ルートに入れると `na exec playwright` がどこから解決されるか曖昧になる
- CI の `playwright install` は `working-directory` を指定する（`setup-dev:ci` 参照）
- `webServer.command` はワークスペース内から相対で書く。
  `playwright.config.ts` の位置が cwd になる

```yaml
      - name: Install Playwright browser
        working-directory: apps/web
        run: na exec playwright install --with-deps chromium
```

Turborepo を使っているなら、`test:e2e` を turbo のタスクにしない。
**キャッシュしてはいけないタスク**（外部サービスの状態に依存する）なので、
`"cache": false` を書くくらいなら turbo を通さないほうが素直。

---

## やらないこと

- **クロスブラウザを最初から入れない。** chromium だけ。
- **視覚回帰（`toHaveScreenshot`）を E2E に混ぜない。** フォントのレンダリング差で
  ローカルと CI が一致せず、`--update-snapshots` を打ち続ける係が生まれる。
  見た目は Storybook 側に置く。
- **`nr test` / pre-commit / pre-push に E2E を入れない。**
- **Google などの本物の IdP のログイン画面を操作しない。**
- **`page.waitForTimeout` を書かない。**
- **開発用のデータベースを使わない。** ただし**そのためにコンテナを増やさない** —
  同じコンテナの別データベースで足りる。
- **migration とデータ投入を setup project / globalSetup に置かない。** `webServer` の
  後に走るので間に合わない。
- **本物の第三者（YouTube、地図タイル、フォント、解析）を叩かせない。**
  ただし**アプリ自身が配るものはブロックしない**。
- **テストのためだけに `data-testid` を足さない。**
- **終了コードをパイプ越しに判断しない。**
- **要らないステップの足場を先に置かない。** 認証も DB も無いなら `setup` project も
  `prepare-db` も作らない。
- **網羅しようとしない。** E2E は「アプリが起動して主要導線が生きている」ことの
  番人であって、仕様の検証器ではない。仕様は unit で検証する。
