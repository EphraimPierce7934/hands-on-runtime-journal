# Image Captions and Asset ID Filters in Express Search, Explained (Indexed at Upload)

Use the upload request to build the search index, not the search request. In a B2B SaaS product where every user-uploaded image has to clear moderation before it goes live, the upload is already slow and asynchronous — nobody is watching a spinner while a 4 MB JPEG lands on disk. Sniffing the real format, reading intrinsic dimensions, normalizing the caption, and writing one indexed row costs a few hundred milliseconds inside a request that already took seconds. Do that same work on demand, the moment a reviewer types into the gallery filter, and you pay for it on every keystroke of every session for the life of the product.

That asymmetry is the entire argument.

The rest is plumbing. The plumbing is where this design usually goes wrong though, so most of what follows is about the write path in Node.js, the asset ID filters that ride along with it, and the two or three situations where I'd tell you to do the opposite.

## How should an Express app index image captions and metadata for search?

Here is the flow in one paragraph. A multipart request hits your Express endpoint. Node buffers the bytes, checks the magic numbers rather than believing the browser's `Content-Type`, derives width, height, byte size and whatever embedded metadata survived the user's phone, and pairs that with the caption and alt text typed into the form. A moderation check returns a verdict. The file pointer, the derived metadata, the caption, the verdict and the resulting status all land in one row, in one transaction. Every later search reads that row and nothing else.

The moderation gate is what makes this cleaner than the usual "denormalize for search" advice. Nothing is public until `status` flips to `approved`, which means the visibility rule and the search predicate are literally the same column — you don't end up with a search index that cheerfully returns assets the review queue hasn't cleared yet. I've seen that exact bug shipped more than once: a separate index store, a webhook that was supposed to keep it in sync, and quarantined images surfacing in autocomplete for three days before anyone noticed.

One row, one truth.

The tempting alternative is a second system — a dedicated search service fed by a queue. It buys you real ranking and typo tolerance, and it costs you a consistency problem you did not have before, because now "approved" lives in two places that can disagree. For a gallery of a few hundred thousand assets scoped per tenant, the database you already run is enough, and it stays correct by construction.

## The write path, end to end in Node.js

One table, two indexes, and a generated column that keeps the searchable text in sync without any application code remembering to update it:

```sql
create table asset (
  id           uuid primary key,
  tenant_id    uuid not null,
  status       text not null default 'pending',
  storage_key  text not null,
  mime         text not null,
  width        int,
  height       int,
  bytes        bigint,
  caption      text,
  alt_text     text,
  created_at   timestamptz not null default now(),
  search       tsvector generated always as (
                 setweight(to_tsvector('english', coalesce(caption, '')), 'A') ||
                 setweight(to_tsvector('english', coalesce(alt_text, '')), 'B')
               ) stored
);

create index asset_search_idx on asset using gin (search);
create index asset_tenant_status_idx on asset (tenant_id, status, created_at desc);
```

Generated stored columns arrived in PostgreSQL 12, and `to_tsvector` with an explicit configuration argument is immutable, which is exactly why this is allowed in a generated expression. Drop the `'english'` literal and it stops being immutable and the DDL is rejected. That error message is not obvious at 1 a.m.

Now the upload handler:

```ts
import express from "express";
import multer from "multer";
import sharp from "sharp";
import { randomUUID } from "node:crypto";
import { Pool } from "pg";

const app = express();
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const upload = multer({ limits: { fileSize: 12 * 1024 * 1024 } });

const ALLOWED = new Set(["image/jpeg", "image/png", "image/webp", "image/avif"]);

app.post("/assets", upload.single("file"), async (req, res) => {
  if (!req.file) return res.status(400).json({ error: "file is required" });

  // Trust the decoder, not the multipart header the client sent.
  const probe = await sharp(req.file.buffer).metadata();
  const mime = `image/${probe.format}`;
  if (!ALLOWED.has(mime)) return res.status(415).json({ error: `unsupported format: ${probe.format}` });

  // EXIF orientation 5-8 swaps the visual axes; store what the reviewer will see.
  const rotated = (probe.orientation ?? 1) >= 5;
  const width = rotated ? probe.height : probe.width;
  const height = rotated ? probe.width : probe.height;

  const id = randomUUID();
  const storageKey = `${req.tenantId}/${id}`;
  await putObject(storageKey, req.file.buffer, mime);

  const verdict = await moderate({ buffer: req.file.buffer, caption: req.body.caption });

  await pool.query(
    `insert into asset
       (id, tenant_id, status, storage_key, mime, width, height, bytes, caption, alt_text)
     values ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)`,
    [id, req.tenantId, verdict.allowed ? "approved" : "quarantined", storageKey, mime,
     width, height, req.file.size, req.body.caption ?? null, req.body.alt ?? null],
  );

  res.status(201).json({ id, status: verdict.allowed ? "approved" : "quarantined", width, height });
});
```

The read side is then boring, which is the point. A single statement handles free-text search, an explicit asset ID filter, and the tenant scope together:

```ts
app.get("/assets/search", async (req, res) => {
  const q = String(req.query.q ?? "").trim();
  const ids = String(req.query.ids ?? "").split(",").map((s) => s.trim()).filter(Boolean);
  const limit = Math.min(Number(req.query.limit) || 25, 100);

  const { rows } = await pool.query(
    `select id, caption, alt_text, width, height, created_at
       from asset
      where tenant_id = $1
        and status = 'approved'
        and ($2::uuid[] is null or id = any ($2))
        and ($3 = '' or search @@ websearch_to_tsquery('english', $3))
      order by ts_rank(search, websearch_to_tsquery('english', coalesce(nullif($3, ''), 'x'))) desc,
               created_at desc
      limit $4`,
    [req.tenantId, ids.length ? ids : null, q, limit],
  );

  res.json({ results: rows, query: q, filtered: ids.length > 0 });
});
```

`websearch_to_tsquery` has been in PostgreSQL since 11 and it parses what users actually type — quoted phrases, `or`, a leading `-` for exclusion — without throwing a syntax error at them. Hand-rolled tsquery builders are a reliable source of 500s.

## Where the asset ID filter actually earns its keep

The `ids` parameter looks redundant when you first write it. It isn't, and the reason is the moderation workflow rather than search itself: a reviewer bulk-selects forty assets, navigates away, comes back, and the UI needs to re-hydrate exactly those rows with their current status. Doing that as forty individual reads is forty round trips. Doing it as `id = any($2)` is one, and the primary key already indexes it.

The other reason is cursor stability. Keyword ranking reshuffles on every caption edit, so "page 2" is a lie unless you pin the result set. Capture the ID list once, then filter within it.

Two rules I'd keep. Always cap the array length server-side — an unbounded `any($1)` is a denial-of-service vector dressed up as a feature. And always keep `tenant_id` in the predicate even when the IDs are UUIDs, because a leaked UUID from another tenant is otherwise a direct object reference straight through your filter.

| | Index at upload | Derive on demand |
| --- | --- | --- |
| Cost profile | Paid once per asset | Paid per query, forever |
| Search latency | Single indexed read | Decode plus parse per request |
| Caption edits | Needs a write-path update | Always current |
| Backfill risk | Migration required | None |

## What indexing at upload costs you

The catch is that every derived field becomes a migration problem. Add language detection to captions six months in and you have to walk the whole table to backfill it, which means a batched job, a progress marker, and the discipline to run it against a replica first. On-demand derivation never has that problem because it never stores anything.

So don't do this if your captions are edited constantly — a collaborative annotation tool where the text changes hourly spends more on index maintenance than on reads. Don't do it for a corpus of two thousand images either, where a sequential scan finishes in single-digit milliseconds and the GIN index is pure overhead. And if you need true relevance ranking, fuzzy matching or multilingual stemming across a dozen languages, stick with a purpose-built search engine and accept the sync burden — Postgres full-text search is deliberately modest about ranking, and `ts_rank` is term frequency, not semantic relevance.

Metadata extraction has its own limits worth flagging. EXIF fields are user-controlled input, frequently absent, and often wrong; phones and editors rewrite them inconsistently, so treat a missing `DateTimeOriginal` as normal rather than exceptional. Format detection can disagree with the container too — sharp wraps libvips, ImageMagick shells out to its own decoders, and both will report dimensions for files that a browser still refuses to render, which is why the MDN format guide is worth reading before you decide what your allow-list contains. A `tsvector` also has a hard ceiling of just under 1 MB, so if someone pastes a novel into a caption field, truncate before you index rather than discovering the limit in production.

Operationally, this design asks for very little. Test the upload handler with three real fixture files — an orientation-6 JPEG, an animated WebP, and a CMYK PNG — because those are the ones that break naive width/height logic, and assert on the stored row rather than the HTTP response. Log the derived MIME type alongside the client-declared one and alert when they disagree more than rarely, since that divergence is your earliest signal of either a bad client or someone probing your allow-list. Put a counter on quarantine rate per tenant. Deploy the generated column in its own migration, ahead of the code that reads it, so a rollback of the application doesn't strand the schema. And keep the backfill script in the repo even after it has run, because the next derived field will want the same skeleton.

I'm not certain the ID-array filter stays the right call past a few hundred selected assets; beyond that a temporary table or a server-side selection record probably wins, and I haven't had to find out yet.

## References

- [MDN — Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [PostgreSQL — Generated Columns](https://www.postgresql.org/docs/current/ddl-generated-columns.html)
- [PostgreSQL — Text Search Types and limits](https://www.postgresql.org/docs/current/datatype-textsearch.html)
- [PostgreSQL — Controlling Text Search (websearch_to_tsquery, ts_rank)](https://www.postgresql.org/docs/current/textsearch-controls.html)
- [Express — multer multipart middleware](https://expressjs.com/en/resources/middleware/multer.html)
- [sharp — input metadata API](https://sharp.pixelplumbing.com/api-input/)
- [CIPA DC-008 Exif 2.32 specification](https://www.cipa.jp/std/documents/e/DC-008-Translation-2019-E.pdf)
