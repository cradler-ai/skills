---
name: cradler
description: Use when an app needs to store data or files — adding a database, saving records, user data, settings, posts, comments, or uploading images and files. Cradler is a zero-config backend (a managed Postgres database plus file storage) used through the @cradler/sdk TypeScript client. There is no SQL, no schema to define, and no migrations.
---

# Cradler

Cradler gives an app a backend — a database and file storage — through one
typed TypeScript SDK. Use it whenever the app you are building needs to
persist data or store uploaded files.

**The most important thing to know:** you never design a schema, write SQL, or
run a migration. You just write data, and Cradler creates the tables and
columns for you. Do not generate `CREATE TABLE` statements, schema files,
ORM models, or migrations — that is not how Cradler works, and it will only
confuse the user.

## Setup

The user needs a Cradler project. If they do not have one, tell them to
create one for free at `https://cradler.ai`. From the project dashboard they
can copy three things — ask the user for them:

- **API URL** — the data gateway URL, e.g. `https://gateway.cradler.ai`
- **Project ID** — the project's slug
- **API keys** — an `anon` key and a `service` key

Install the SDK:

```bash
pnpm add @cradler/sdk
```

Create the client. Read the keys from environment variables — never hardcode
them:

```ts
import { createClient } from "@cradler/sdk";

const cradler = createClient({
  url: process.env.CRADLER_API_URL!,
  projectId: process.env.CRADLER_PROJECT_ID!,
  apiKey: process.env.CRADLER_API_KEY!,
});
```

### Which key to use

- **`service` key** — full access. Use it **only in server-side code** (API
  routes, server actions, backend jobs). Never send it to the browser.
- **`anon` key** — safe for client-side / browser code. What it can do is
  limited by the table permissions the user sets in the dashboard.

If the app talks to Cradler directly from the browser, use the `anon` key
there, and tell the user to enable the table permissions it needs in the
dashboard.

## Storing and reading data

A "collection" is a table. You do not create it — it appears the first time
you write to it. Use plain `camelCase` field names; the SDK maps them to the
database for you.

### Insert

```ts
// The "posts" table and its columns are created automatically.
await cradler.from("posts").insert({
  title: "Hello world",
  body: "My first post",
  published: true,
});

// Insert many at once:
await cradler.from("posts").insert([{ title: "A" }, { title: "B" }]);
```

Writing a field that does not exist yet adds the column automatically.

Every row automatically gets three Cradler-managed fields — `id`,
`createdAt`, and `updatedAt`. Read them and filter or order by them, but
never pass them to `insert()` or `update()` — Cradler always sets them.

### Query

```ts
// All rows:
const { rows } = await cradler.from("posts").select();

// Specific columns, filtered, ordered, paginated:
const { rows, count } = await cradler
  .from("posts")
  .select("id", "title")
  .eq("published", true)
  .order("createdAt", { desc: true })
  .limit(20);

// Just the first match, or null:
const post = await cradler.from("posts").select().eq("id", id).first();

// How many posts are there in total? `count` cannot answer that — it is the
// size of the page you got back, so it never exceeds `limit`. Ask for
// `total`, which counts every row matching the filters:
const { rows: page, total } = await cradler
  .from("posts")
  .select()
  .eq("published", true)
  .limit(20)
  .count("exact");
```

Filters — chain as many as needed: `.eq` `.neq` `.gt` `.gte` `.lt` `.lte`
`.like(field, pattern)` `.ilike(field, pattern)` `.in(field, [...])`
`.isNull(field)` `.notNull(field)`.

### Update and delete

```ts
// update() requires at least one filter — an unscoped update would rewrite
// every row in the collection, with no way to undo it.
await cradler.from("posts").update({ published: false }).eq("id", id);

// delete() requires at least one filter — it can never wipe a whole table.
await cradler.from("posts").delete().eq("id", id);
```

Every insert / query / update / delete resolves to `{ rows, count }`, where
`count` is `rows.length`. On a query it is the size of the page, so it is
bounded by `limit` — to learn how many rows match overall, add
`.count("exact")` and read `total`.

## Files and images

```ts
// Upload — body is a Blob, ArrayBuffer, or string.
await cradler.storage.upload("avatars/cat.png", fileBlob, {
  contentType: "image/png",
});

// A temporary signed URL to display or download the file:
const url = await cradler.storage.getUrl("avatars/cat.png");

// Download as a Blob:
const blob = await cradler.storage.download("avatars/cat.png");

// List (optionally by prefix) and delete:
const files = await cradler.storage.list("avatars/");
await cradler.storage.remove("avatars/cat.png");
```

**When the upload happens in the browser, turn on image compression.**
Pass `{ compress: true }` and the SDK will shrink and re-encode the image
on the device before it is sent — typically 90%+ smaller than a raw phone
photo. Non-image files (PDFs, zips, etc.) pass through unchanged, so it
is safe to set on every browser-side upload:

```ts
const { path } = await cradler.storage.upload("avatars/cat.jpg", file, {
  compress: true,
});
// `path` is now "avatars/cat.webp" — the extension follows the new format.
// Save *this returned path* in the database, not the original.
```

Do not set `compress: true` in Node / server-side code — it only works in
the browser. In server code, upload the bytes as-is.

**When you save a file reference in the database, store the path returned
by `upload()`, not the URL.** URLs from `getUrl()` are short-lived signed
URLs that expire; persisting them leads to broken links. Resolve the path
to a fresh URL with `getUrl()` when you need to display it.

## Errors

Failed calls throw a `CradlerError` with an HTTP `status`, a machine-readable
`code`, and a `message` that names what is wrong:

```ts
import { CradlerError } from "@cradler/sdk";

try {
  await cradler.from("posts").insert({ title: "x" });
} catch (err) {
  if (err instanceof CradlerError) {
    console.error(err.status, err.code, err.message);
  }
}
```

**Read the message — it says how to fix the call.** A type mismatch names the
field, the type the column holds and the value it got
(`field 'views' expects numeric, got str ('many')`); fix the value rather than
retrying. The codes worth handling:

| code | status | means |
| --- | --- | --- |
| `validation_failed` | 422 | A value or filter does not fit the column. The message names the field. |
| `schema_conflict` | 422 | A field's type clashes with the column that already exists. |
| `quota_exceeded` | 402 | The project is out of its plan's monthly requests or storage. |
| `constraint_violation` | 409 | A unique index or similar rejected the write. |
| `unauthorized` / `forbidden` | 401 / 403 | Bad key, or the `anon` key lacks permission on that table. |
| `database_unavailable` | 503 | Transient — retry with backoff. |

Every error also carries a `requestId`, which appears in Cradler's own logs.
Include it when reporting a problem.

## Rules

- Never write SQL, schema files, `CREATE TABLE`, ORM models, or migrations.
  Just write data — the schema evolves on its own.
- Never put the `service` key in client-side / browser code; use the `anon`
  key there.
- Do not hardcode keys — read them from environment variables.
- Use `camelCase` field names.
- `id`, `createdAt`, `updatedAt` are managed by Cradler — read them, but
  never include them in an `insert()` or `update()`.
- `update()` and `delete()` always need at least one filter.
- A field keeps the type it was first written with. Write `42`, not `"42"`,
  and keep it a number afterwards — mixing types in one field is rejected,
  including across the rows of a single batch insert.
- `count` is how many rows came back, not how many exist. Use
  `.count("exact")` and `total` to answer "how many".

## Related

To read and write a project's data **directly** from an agent — operating on
the data rather than writing app code — Cradler also offers an MCP server,
`@cradler/mcp`. Its `query` tool has the same distinction as the SDK: `count`
is the page size, and `countTotal: true` returns `total`.
