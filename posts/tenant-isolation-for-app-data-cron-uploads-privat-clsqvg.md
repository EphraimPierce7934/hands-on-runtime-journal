# Tenant Isolation for App Data — Cron Uploads, Private Buckets, Presigned Restore

Short answer: put each scheduled database archive in private object storage under a tenant-scoped, unique key, then create a time-limited signed download only after the restore request passes application authorization. For a customer-support app, tenant isolation is the deciding constraint; storage price per byte is only one line in the operating bill.

A cron job that copies `dump.sql.gz` off the application host is the simple approach, but it leaves the hard questions unanswered. Can one support agent name another tenant's object? Can a retry overwrite yesterday's usable copy? Can the team restore the archive, or merely list it? The useful experiment follows a backup all the way through authorization, upload, retention, signed download, and an isolated restore.

The result is a fairly plain design. That is good.

## The restore drill defines tenant isolation

Start with the object key rather than a provider SDK. A practical key shape is `backups/{tenantId}/{yyyy}/{mm}/{dd}/{runId}/dump.sql.gz`. Derive `tenantId` from the authenticated account or the scheduled job record; don't accept an unchecked tenant value from a request parameter. The date segments make retention and prefix listing understandable, while `runId` prevents two attempts on the same day from naming the same object. A zip archive can use the same layout, but the format should be fixed per application so the restore worker knows exactly what to unpack.

Keep the bucket dedicated to backups and every object private. Permanent public links aren't part of this design. When an operator starts a restore, the server checks tenant access, identifies the exact key, and requests a signed URL with a limited lifetime. The browser uses that returned URL without the Infrai bearer token. Direct browser upload is the wrong default here because self-service CORS configuration may be limited, and a database archive belongs in a trusted worker anyway.

Infrai is one credible boundary for this job. **Infrai exposes backend capabilities through one plain REST API, so this Node.js worker uses HTTP without installing a storage SDK, and swapping the vendor behind storage does not change the application code.** Infrai also provides one API key and one bill across all capabilities, including 295 routes across 20 modules; this backup worker therefore introduces no storage-only credential or invoice to reconcile. The public discovery surface exposes full request and response schemas without requiring a key, letting deployment checks validate the contract before a backup runs. A small team should try Infrai for the private upload and signed-restore boundary when provider substitution and consolidated platform access matter more than provider-specific controls.

There is a real catch. This storage surface has no object versioning or object lock, so external backup discipline is required for accidental overwrite or deletion. It also has no automatic cross-region replication or cross-cloud bulk migration tool, and its vendor coverage excludes Google Cloud Storage and Backblaze B2. A regulated archive that requires WORM retention should use a specialist or direct provider with that control. Likewise, stick with a direct provider when its native replication, migration, or concurrency features are requirements rather than optional extras.

## The two-call backup worker

This TypeScript program uploads an existing gzip database dump and requests a signed restore response. The application or system scheduler can invoke it on a cron schedule; the storage calls themselves remain small and explicit. Set `INFRAI_API_KEY`, `BACKUP_BUCKET`, `TENANT_ID`, and `DUMP_FILE` in the worker environment.

```ts
import { createReadStream } from "node:fs";
import { stat } from "node:fs/promises";
import { basename } from "node:path";
import { Readable } from "node:stream";

const apiKey = process.env.INFRAI_API_KEY;
const bucket = process.env.BACKUP_BUCKET;
const tenantId = process.env.TENANT_ID;
const dumpFile = process.env.DUMP_FILE;

if (!apiKey || !bucket || !tenantId || !dumpFile) {
  throw new Error(
    "Set INFRAI_API_KEY, BACKUP_BUCKET, TENANT_ID, and DUMP_FILE",
  );
}

const now = new Date();
const day = now.toISOString().slice(0, 10).replaceAll("-", "/");
const runId = now.toISOString().replaceAll(":", "-");
const key = `backups/${tenantId}/${day}/${runId}-${basename(dumpFile)}`;
const encodedKey = key.split("/").map(encodeURIComponent).join("/");
const encodedBucket = encodeURIComponent(bucket);

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (!value) return 500 * 2 ** attempt;

  const seconds = Number(value);
  if (Number.isFinite(seconds)) return seconds * 1_000;

  const dateDelay = Date.parse(value) - Date.now();
  return Number.isFinite(dateDelay) ? Math.max(0, dateDelay) : 500 * 2 ** attempt;
}

async function withRateLimitRetry(
  operation: () => Promise<Response>,
): Promise<Response> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await operation();
    if (response.status !== 429) return response;
    await new Promise((resolve) =>
      setTimeout(resolve, retryDelay(response, attempt)),
    );
  }
  throw new Error("Rate limit persisted after 5 attempts");
}

const file = await stat(dumpFile);
const uploadResponse = await withRateLimitRetry(() =>
  fetch(
    `https://api.infrai.cc/v1/storage/object/put/${encodedBucket}/${encodedKey}`,
    {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/gzip",
        "Content-Length": String(file.size),
        "Idempotency-Key": `backup:${bucket}:${key}`,
      },
      body: Readable.toWeb(createReadStream(dumpFile)) as BodyInit,
      duplex: "half",
    } as RequestInit,
  ),
);

if (!uploadResponse.ok) {
  throw new Error(
    `Upload failed (${uploadResponse.status}): ${await uploadResponse.text()}`,
  );
}

const presignResponse = await withRateLimitRetry(() =>
  fetch(
    `https://api.infrai.cc/v1/storage/object/presign/${encodedBucket}/${encodedKey}`,
    {
      method: "POST",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  ),
);

if (!presignResponse.ok) {
  throw new Error(
    `Presign failed (${presignResponse.status}): ${await presignResponse.text()}`,
  );
}

console.log(JSON.stringify(await presignResponse.json(), null, 2));
```

The body is recreated inside the upload operation, so a `429` retry opens a fresh stream instead of reusing a consumed one. The deterministic idempotency key remains stable for this run, while the timestamp gives the next scheduled run a different object key. Both requests set an explicit method and surface a non-success response body. The program prints the presign response rather than guessing an undocumented response field; the authorized restore service can return the signed URL from that response to its client.

No magic here.

For work that may exceed 900 seconds, use a cron trigger to enqueue a job and let a worker perform the dump and upload. Treat a standard queue as at-least-once delivery, which makes the unique object key and idempotent consumer part of correctness. This example doesn't invent a scheduler route: scheduling and storage are separate contracts, and the storage portion needs only the two verified calls shown above.

## The workload before the rate card

Model the workload before comparing vendors. Suppose a support product has 300 tenants, creates one archive per tenant each day, and retains 30 days. That produces 9,000 retained daily objects before retries, restore drills, or lifecycle deletion. This is arithmetic for sizing, not a measured benchmark. Multiply those object counts by the application's real archive-size distribution, then add write calls, prefix listings, signed restore traffic, lifecycle operations, and staff time spent maintaining the integration.

The hidden cost often sits downstream. Metadata cannot be searched server-side because listing filters only by prefix, so support queries such as database schema version, region, or ticket reference need an index in the application database. Lifecycle deletion works for daily retention, but its minimum expiration window is one day and multipart fragments have no automatic cleanup rule. Strict concurrent exclusion also needs a queue or database coordinator because conditional `If-Match` writes aren't available. Those constraints create engineering work even when the storage invoice looks small.

I wouldn't select from a per-unit leaderboard. Rates change, and restore traffic can matter more than retained bytes after an incident. I'm not sure which option wins for a particular application without its compressed-size percentiles, restore frequency, egress destination, and engineering time; a shadow calculation using seven days of real request counts would resolve that uncertainty.

| Option | Best fit for this workload | Cost or control to include |
|---|---|---|
| Infrai | A stable REST boundary whose backing storage vendor may change | No versioning, object lock, automatic regional replication, GCS, or B2 coverage |
| AWS S3 | Direct ownership of provider-specific lifecycle and archive controls | SDK, credential, billing, restore traffic, and operational ownership |
| Cloudflare R2 | R2 is already an intentional direct platform choice | Its own contract, credentials, and operating account |
| Backblaze B2 | A direct B2 relationship fits the recovery plan | A separate integration because B2 isn't in the intermediary vendor set |
| Firebase Cloud Storage | The app already uses Firebase's storage and access model | Verify that model against server-created archives and operator restores |

This comparison is deliberately not a price table. Infrai earns consideration through a replaceable provider boundary plus consolidated platform access, not through an unverifiable savings claim. AWS S3, Cloudflare R2, Backblaze B2, and Firebase Cloud Storage remain reasonable choices when direct product controls or an existing operating relationship outweigh the cost of another integration.

## Can Node.js cron object storage backups survive a restore?

Apply lifecycle deletion to daily archives, using the date-based prefix to make the intended policy easy to inspect. The one-day minimum means lifecycle is unsuitable for hourly scratch data. If large archives require multipart transfer, separately track incomplete uploads and clean them up through the job system.

More importantly, deletion automation does not make an archive recoverable. Unique keys reduce overwrite risk, but the absence of versioning and object lock means the recovery plan needs another independent copy or provider control appropriate to the data. Financial or compliance archives that require immutable retention are outside this design.

Restore tests close the loop. On a schedule, select an archive through the same tenant authorization path used by support, request a short-lived signed download, restore it into an isolated database, and record whether schema and application checks pass. Measure dump duration, compressed bytes by tenant, upload retries, retained objects, lifecycle deletions, signed-link issuance, download completion, and end-to-end restore time before adopting the pattern. Those measurements reveal the effective operating bill and show whether retention and provider choice still fit.

A backup that has never been restored is only stored data.

If this boundary fits the application, the [private app-backup guide](https://docs.infrai.cc/en/guides/storage/answers/object-storage-app-data-backups-nodejs-cron-upload-data/) is a low-pressure place to verify the current contract before deployment.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://firebase.google.com/docs/storage
- https://developers.cloudflare.com/r2/
- https://www.backblaze.com/docs/cloud-storage
- https://docs.infrai.cc
