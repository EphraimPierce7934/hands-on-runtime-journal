# Node.js Presigned Links for Tenant-Isolated Private Exports

Short answer: keep the object private, authorize the tenant in your Node.js application, and create a short-lived presigned download URL only after the export record and object key have passed that check. The URL is a delivery token, not an authorization system.

That distinction is the part I would ship first. A private file export has two separate jobs: deciding who may download a snapshot, and moving the resulting bytes. Let the application do the first job and object storage do the second. Sending a large ZIP through the Node.js process wastes a useful boundary; making the bucket public destroys tenant isolation.

The example below assumes a developer tool that exports a tenant's project data. The same design applies to CSVs and PDFs, but the isolation rule stays fixed: an object key must be scoped to a tenant, and the database row must be checked before a signature is minted.

## The export record is the control plane

Because a presigned URL usually carries enough authority for its holder to fetch the object during its validity period. It does not know that the browser belongs to tenant `acme`; it knows only the object and the signature. If an endpoint accepts an arbitrary key from the client and signs it, the storage layer has been asked to enforce an application relationship it cannot see.

Store an export record like this:

```ts
type ExportRecord = {
  id: string;
  tenantId: string;
  objectKey: string;
  contentType: "text/csv" | "application/zip";
  status: "ready" | "expired" | "deleted";
};
```

The client should submit an export ID, not an object key. The server loads that row, compares `tenantId` with the authenticated tenant, checks that `status` is `ready`, and asks the storage adapter to sign the stored key. A key is data owned by the export record; it is not a permission supplied by the browser.

Here is the boundary in TypeScript. `storage.createPresignedGetUrl` is an application adapter, so the authorization decision remains independent of the storage SDK or provider.

```ts
type Storage = {
  createPresignedGetUrl(input: {
    objectKey: string;
    expiresInSeconds: number;
    responseContentType: string;
  }): Promise<string>;
};

async function getExportDownloadUrl(
  exportId: string,
  authenticatedTenantId: string,
  exports: { findById(id: string): Promise<ExportRecord | null> },
  storage: Storage,
): Promise<{ url: string }> {
  const record = await exports.findById(exportId);

  if (!record || record.tenantId !== authenticatedTenantId) {
    throw new Error("Export not found");
  }
  if (record.status !== "ready") {
    throw new Error("Export is not ready");
  }

  const url = await storage.createPresignedGetUrl({
    objectKey: record.objectKey,
    expiresInSeconds: 600,
    responseContentType: record.contentType,
  });

  return { url };
}
```

Returning “not found” for a record owned by another tenant avoids turning the endpoint into an ID oracle. The exact error envelope can follow your API convention. The ordering cannot: authorization belongs before signing.

Short URLs help, but they do not make a leaked URL harmless. A recipient can use a valid link until it expires, and a URL may appear in browser history, referrer data, support tickets, or access logs. Keep it out of persistent export rows and avoid logging query strings. For particularly sensitive data, a download endpoint can issue the URL after a fresh session check, while the browser navigates directly to storage.

## How can Node.js keep presigned object storage links private?

Use an immutable key for each completed export, such as `tenants/acme/exports/8f2c/data.zip`. The random export ID prevents a regeneration from silently replacing bytes that an older link references. It also gives cleanup code something concrete to reconcile with the database.

The write path is deliberately uninteresting: create the job, generate the file, upload it to a private bucket, verify the write, and transition the row to `ready` in a transactionally coordinated step. Only then is the download URL available. If two workers race, each should write a distinct key; the job state decides which completed record the UI presents.

The read path is shorter:

1. The browser asks the application for an export ID.
2. The application authenticates the user and checks the tenant on the export row.
3. The application creates a time-limited presigned GET URL.
4. The browser follows that URL and receives the object directly.

The browser does not need the storage service's long-lived credentials. It may navigate to the URL or fetch it from JavaScript. Those are different browser behaviors: a JavaScript request may require the storage origin's CORS policy, while a top-level download navigation does not have the same response-reading requirement. Test the actual UI flow, including the `Content-Disposition` behavior you want, rather than assuming that a URL which works in a command-line client will behave identically in a browser.

For a web app, CORS is a browser permission mechanism, not tenant authorization. Configure it only for the origins and methods the browser truly needs, and do not add a broad origin rule merely to make an export button work. MDN's CORS guide is the useful reference here because it explains the preflight and response-header rules that affect a JavaScript fetch.

## The failure matrix matters more than the happy path

The dangerous failures are usually ordinary application races, not cryptography.

If a worker uploads bytes and then loses its database transaction, the bucket contains an unowned object. Reconciliation should find keys with no live export record and remove them according to a tested retention policy. If the database says `ready` but the object has been removed, return a regeneration state; do not keep handing the UI a URL that can never succeed.

If a user requests a new export while an old one is downloading, give the two objects different keys. A stable key such as `tenant/latest.zip` makes overwrite timing part of the user-visible behavior. That can be acceptable for a cache, but it is a poor identity for an auditable export.

Expiration is another split responsibility. URL expiry controls access to a particular request. It does not necessarily delete the bytes. The cleanup policy must separately define how long the object remains, who can delete it, and what happens to the database record after deletion. Otherwise, “the link expired” can be mistaken for “the customer data was removed.”

I would put these cases into an integration test with concrete assertions:

| Case | Assertion |
| --- | --- |
| Wrong tenant requests an export ID | No URL is created and the response does not reveal ownership |
| Ready row points to a missing object | The UI gets a regeneration state |
| Worker loses its transaction after upload | Cleanup can identify the unowned key |
| Old URL is used after expiry | The storage service rejects the request |
| Two exports finish out of order | Each result retains its own bytes and metadata |

Run the test with a small CSV and a larger ZIP. The byte count changes the latency and timeout story, even though the authorization rule does not.

## Retention decides when this pattern stops fitting

The catch is that a presigned URL is bearer access. It is not suitable when the link must be permanent, indexed, or safely pasted into a public document. Use an intentionally public delivery design for public assets, and use a stronger access gateway when every byte request needs continuous identity checks or download revocation.

It is also a poor fit when the export must be immutable for a regulatory retention period unless your chosen storage system provides the required retention and legal-hold controls. “Private” and “immutable” are separate properties. A private bucket can still permit deletion by an operator or application role.

Large exports may need a queue, resumable transfer, or a dedicated delivery layer. Your mileage may vary with client geography and file size; the ten-minute example above is a policy placeholder, not a universal answer. Measure how long real downloads take, then choose an expiry that covers a legitimate start and transfer window without creating a long-lived token.

Stick with a direct application-mediated download when the files are small, the product needs per-request policy checks, or the browser must receive a consistent product error page. Use direct storage delivery when the application has already made the access decision and the main concern is moving bytes efficiently. The right boundary is the one you can observe and revoke according to your data policy.

## Ship against evidence, not a successful browser click

Measure export duration, object size, time from signing to first byte, completed-download rate, expired-link rate, and unowned-object count. Add tenant-mismatch attempts to security telemetry without recording the signed URL itself. A useful dashboard can show whether failures cluster around generation, signing, CORS, or the final object GET.

I would also record the policy inputs used to make a decision: tenant ID, export ID, object key hash, and expiry timestamp. Do not record the bearer URL. This gives an incident review enough context to answer “which export was authorized?” without creating another copy of the credential that authorized it.

The implementation is ready when the application can prove three things: a tenant cannot obtain another tenant's object key through the export endpoint, a valid link delivers exactly the selected snapshot, and expired or deleted exports produce a defined user-facing state. Those checks are more durable than any particular SDK call.

## Further reading

- MDN: Cross-Origin Resource Sharing (CORS): https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- AWS S3: Downloading an object with a presigned URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
