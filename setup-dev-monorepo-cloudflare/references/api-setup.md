# apps/api（Hono + Drizzle）のセットアップ

## 6. Initialize apps/api

`apps/api/package.json`:
```json
{
  "name": "@repo/api",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "wrangler dev",
    "build": "wrangler deploy --dry-run --outdir dist",
    "deploy": "wrangler deploy",
    "lint": "oxlint .",
    "typecheck": "tsgo --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "db:generate": "drizzle-kit generate",
    "db:migrate:local": "wrangler d1 migrations apply DB --local",
    "db:migrate:remote": "wrangler d1 migrations apply DB --remote",
    "db:studio": "drizzle-kit studio"
  },
  "dependencies": {
    "@repo/shared": "workspace:*",
    "drizzle-orm": "catalog:",
    "hono": "catalog:",
    "zod": "catalog:",
    "@hono/zod-validator": "latest"
  },
  "devDependencies": {
    "@cloudflare/workers-types": "latest",
    "drizzle-kit": "latest",
    "wrangler": "latest"
  }
}
```

## 7. Configure Hono for Cloudflare Workers

**API のディレクトリ構成（レイヤードアーキテクチャ）:**

```
apps/api/src/
  routes/            # Hono ルート定義（リクエスト/レスポンスの変換のみ）
    users.ts
    health.ts
  services/          # ビジネスロジック（関数引数で DI、テスト容易）
    user-service.ts
  repositories/      # Drizzle での DB 操作をラップ
    user-repository.ts
  db/
    schema.ts        # Drizzle スキーマ定義
  index.ts           # エントリポイント（ルートの集約）
```

- `routes/` → リクエスト/レスポンスの変換のみ。ロジックは `services/` に委譲
- `services/` → ビジネスロジック。DB インスタンスを関数引数で受け取る（DI コンテナ不要）
- `repositories/` → Drizzle の DB 操作をラップ。services から呼び出される

`apps/api/src/index.ts`:
```ts
import { Hono } from "hono";
import { cors } from "hono/cors";
import { healthRoute } from "./routes/health";
import { usersRoute } from "./routes/users";

type Bindings = {
  DB: D1Database;
  BUCKET: R2Bucket;
};

const app = new Hono<{ Bindings: Bindings }>()
  .use("*", cors())
  .route("/health", healthRoute)
  .route("/users", usersRoute);

export type AppType = typeof app;
export default app;
```

`apps/api/src/repositories/user-repository.ts`（例）:
```ts
import { eq } from "drizzle-orm";
import type { DrizzleD1Database } from "drizzle-orm/d1";
import { users } from "../db/schema";

export function createUserRepository(db: DrizzleD1Database) {
  return {
    findById: (id: number) =>
      db.select().from(users).where(eq(users.id, id)).get(),
    findAll: () => db.select().from(users).all(),
  };
}
```

`apps/api/src/services/user-service.ts`（例）:
```ts
import type { createUserRepository } from "../repositories/user-repository";

export function createUserService(
  repo: ReturnType<typeof createUserRepository>,
) {
  return {
    getUser: (id: number) => repo.findById(id),
    listUsers: () => repo.findAll(),
  };
}
```

`apps/api/src/routes/users.ts`（例）:
```ts
import { zValidator } from "@hono/zod-validator";
import { drizzle } from "drizzle-orm/d1";
import { Hono } from "hono";
import { z } from "zod";
import { createUserRepository } from "../repositories/user-repository";
import { createUserService } from "../services/user-service";

type Bindings = { DB: D1Database };

const createUserSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
});

export const usersRoute = new Hono<{ Bindings: Bindings }>()
  .get("/", async (c) => {
    const db = drizzle(c.env.DB);
    const repo = createUserRepository(db);
    const service = createUserService(repo);
    return c.json(await service.listUsers());
  })
  .post("/", zValidator("json", createUserSchema), async (c) => {
    const data = c.req.valid("json");
    const db = drizzle(c.env.DB);
    const repo = createUserRepository(db);
    const service = createUserService(repo);
    return c.json(await service.createUser(data), 201);
  });
```

**重要**: `app` の定義でメソッドチェーンを使い、`const app = new Hono<...>().route(...).route(...)` のように書くこと。こうすることで `AppType` にルートの型情報が含まれ、RPC クライアントで型安全にアクセスできる。

## 8. Configure Drizzle ORM with D1

`apps/api/src/db/schema.ts`（サンプルスキーマ）:
```ts
import { integer, sqliteTable, text } from "drizzle-orm/sqlite-core";

export const users = sqliteTable("users", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  name: text("name").notNull(),
  email: text("email").notNull().unique(),
  createdAt: integer("created_at", { mode: "timestamp" })
    .notNull()
    .$defaultFn(() => new Date()),
});
```

`apps/api/drizzle.config.ts`:
```ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/db/schema.ts",
  out: "./drizzle/migrations",
  dialect: "sqlite",
});
```

## 9. Configure Wrangler for API

`apps/api/wrangler.jsonc`:
```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "project-name-api",
  "main": "src/index.ts",
  "compatibility_date": "2026-03-01",
  "compatibility_flags": ["nodejs_compat"],
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "project-name-db",
      "database_id": "YOUR_DATABASE_ID",
      "migrations_dir": "drizzle/migrations"
    }
  ],
  "r2_buckets": [
    {
      "binding": "BUCKET",
      "bucket_name": "project-name-bucket"
    }
  ]
}
```

**注意**: `database_id` はユーザーに `wrangler d1 create project-name-db` で D1 データベースを作成してもらい、出力された ID を設定するよう案内する。

## 10. Configure TypeScript for API

`apps/api/tsconfig.json` — **`setup-dev:tsconfig` の「Base compilerOptions」+「Cloudflare Workers 向け追加設定」+「モノレポのパス参照設定」に従う。**

## 11. Configure Vitest for API

**`setup-dev:testing` の「API / バックエンド向け（minimal setup）」に従う。**

`apps/api/package.json` の `devDependencies` に `vitest` を追加。

---

## packages/database の分離パターン（オプション）

DB スキーマ・マイグレーション・Drizzle Studio を `apps/api` から `packages/database` に分離する場合のパターン。web からも DB スキーマの型を参照したい場合や、マイグレーション管理を独立させたい場合に有用。

### ローカル DB ファイルの検索スクリプト

`packages/database` の `dev`（drizzle-kit studio）は `apps/api` の wrangler が作成するローカル D1 SQLite ファイルを必要とする。turbo の `dev` タスクは全パッケージを並列起動するため、API がまだ起動していない状態で studio が走ると `find` が失敗する。

`packages/database/scripts/find-local-db.sh` を作成し、リトライ付きで DB ファイルを検索する:

```bash
#!/bin/bash
# D1 ローカル SQLite ファイルを探す
# wrangler が起動してから DB ファイルが作られるまでラグがあるため、リトライする

DB_DIR="../../apps/api/.wrangler/state/v3/d1/miniflare-D1DatabaseObject"
MAX_RETRIES=${1:-10}
INTERVAL=3

for i in $(seq 1 "$MAX_RETRIES"); do
  DB_PATH=$(find "$DB_DIR" -type f -name '*.sqlite' -print -quit 2>/dev/null)
  if [ -n "$DB_PATH" ]; then
    echo "$DB_PATH"
    exit 0
  fi
  if [ "$i" -lt "$MAX_RETRIES" ]; then
    echo "Waiting for D1 local database... (${i}/${MAX_RETRIES})" >&2
    sleep "$INTERVAL"
  fi
done

echo "D1 local database not found after ${MAX_RETRIES} retries" >&2
exit 1
```

### packages/database の package.json スクリプト

```json
{
  "scripts": {
    "find-local-db": "bash scripts/find-local-db.sh",
    "generate": "cross-env LOCAL_DB_PATH=$(bash scripts/find-local-db.sh 1) drizzle-kit generate",
    "studio": "cross-env LOCAL_DB_PATH=$(bash scripts/find-local-db.sh 1) drizzle-kit studio",
    "dev": "cross-env LOCAL_DB_PATH=$(bash scripts/find-local-db.sh) drizzle-kit studio"
  }
}
```

- `dev` — リトライ有り（デフォルト10回 = 最大30秒待機）。turbo で api と並列起動されるため
- `generate`, `studio` — リトライ1回。手動実行で API 起動済みを前提とする
