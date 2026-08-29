# Node.js Private Object Storage for PDF and ZIP Intake: Plan Around Failure

Choose an upload path by the most expensive restart a user can tolerate, then make the upload coordinator own authorization and the ready state. **Short answer: use resumable, part-based transfer when a full retry of a private PDF or ZIP would be unacceptable; keep a single upload only when that retry is genuinely cheap and predictable.**

For a document product, the useful boundary is simple: the application controls identity, object names, and state; object storage receives bytes. A browser, desktop agent, or Node.js worker can send those bytes directly with narrowly scoped, short-lived instructions. The coordinator records the session, accepts part receipts, verifies completion, and makes the document downloadable only after its own checks pass.

The file type is not the rule. A 100 MB PDF on a slow or intermittent connection can need resumability more than a 1 GB ZIP moving over a controlled internal network. The decision is about failure cost, memory pressure, and the time needed to start again.

## How should Node.js send PDF and ZIP documents to private object storage?

Start with a failure budget. Measure the slow-end connection that matters for the product, estimate how long a full resend takes, and decide whether that interruption is acceptable. Then pick a part size and bounded concurrency that keep the client and service within their memory and socket budgets. Don't inherit a threshold from a blog post: connection quality, cancellation behavior, and concurrent upload count change the result.

The coordinator is a control plane, not a byte relay. It authenticates the caller, creates an opaque key under the caller's tenant, records the proposed content length, and returns only the upload targets needed for that session. It rejects a completion request whose part numbers or receipts do not match the recorded session. A private download repeats the authorization check before it issues a short-lived read instruction.

That separation makes the important state transitions explicit:

1. `created`: the service has bound a document record, tenant, intended size, and expiration to one upload session.
2. `transferring`: numbered parts can be retried independently, while the coordinator retains only the receipts it needs to validate completion.
3. `verifying`: the service confirms the submitted receipt list and runs any required content policy before publishing availability.
4. `ready` or `aborted`: only `ready` may produce a download instruction; expired sessions are cleaned up.

Small states. Fewer surprises.

## A focused TypeScript upload client

This client talks to an application-owned coordinator. The coordinator API is deliberately generic so the storage adapter can change without spreading storage-specific multipart vocabulary through product code. The example uploads sequentially because resume and completion correctness are more valuable than premature parallelism. Increase concurrency only after measuring throughput, process memory, and cancellation behavior.

```ts
import { open } from "node:fs/promises";
import { basename } from "node:path";

type UploadSession = {
  id: string;
  partSize: number;
  targets: Array<{ number: number; url: string }>;
};

type Receipt = { number: number; receipt: string };

const coordinator = requireEnv("DOCUMENT_UPLOAD_API");
const token = requireEnv("DOCUMENT_UPLOAD_TOKEN");

async function uploadPrivateDocument(filePath: string): Promise<void> {
  const file = await open(filePath, "r");
  try {
    const { size } = await file.stat();
    const session = await request<UploadSession>("/uploads", {
      method: "POST",
      body: JSON.stringify({ filename: basename(filePath), size }),
    });
    const receipts: Receipt[] = [];

    for (const target of session.targets) {
      const offset = (target.number - 1) * session.partSize;
      const length = Math.min(session.partSize, size - offset);
      const bytes = Buffer.alloc(length);
      const { bytesRead } = await file.read(bytes, 0, length, offset);
      if (bytesRead !== length) throw new Error(`short read for part ${target.number}`);

      const response = await putWithRetry(target.url, bytes);
      const receipt = response.headers.get("etag");
      if (!receipt) throw new Error(`missing receipt for part ${target.number}`);
      receipts.push({ number: target.number, receipt });
    }

    await request(`/uploads/${session.id}/complete`, {
      method: "POST",
      body: JSON.stringify({ receipts }),
    });
  } finally {
    await file.close();
  }
}

async function putWithRetry(url: string, bytes: Buffer): Promise<Response> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, { method: "PUT", body: bytes });
    if (response.ok) return response;
    if (response.status !== 429 && response.status < 500) {
      throw new Error(`part rejected with ${response.status}`);
    }
    await new Promise((resolve) => setTimeout(resolve, 250 * 2 ** attempt));
  }
  throw new Error("retry budget exhausted");
}

async function request<T = unknown>(path: string, init: RequestInit): Promise<T> {
  const response = await fetch(`${coordinator}${path}`, {
    ...init,
    headers: {
      authorization: `Bearer ${token}`,
      "content-type": "application/json",
    },
  });
  if (!response.ok) throw new Error(`coordinator rejected request: ${response.status}`);
  return response.json() as Promise<T>;
}

function requireEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
}

await uploadPrivateDocument(process.argv[2]);
```

There are two details worth preserving if the implementation grows. First, make completion idempotent: a repeated complete call must resolve to the same document, not create another published object. Second, persist the session and each accepted receipt before reporting progress to the user. Retrying a network operation is useful; retrying an unrecorded state transition is how an upload UI begins to lie.

For a Node.js server that receives the bytes itself, stream to a bounded destination and apply backpressure. Do not read a 1 GB document into a Buffer. Direct transfer removes that relay from the hot path, which usually reduces application bandwidth and isolates object-transfer retries from request handlers, but it adds a coordinator and signed-instruction lifecycle to test.

## The catch: private storage needs more than an upload URL

A private bucket is a storage setting, not an authorization model. The application still needs tenant-to-object mapping, object-key rules that callers cannot override, expiration, and an audit trail for state changes. Store the original filename separately from the opaque key when possible; display names are user input, while opaque keys form an access boundary. There is a practical ordering here: authorize session creation before minting any target, bind every target to one server-chosen key, and require the same tenant context when completion is requested. Keep an explicit expiry in the database even if the underlying target also expires. During a support investigation, the document record should answer who initiated the transfer, which expected size was approved, which receipts were accepted, and why the object was released; it should not require reconstructing a chain from temporary URLs or application access logs. If the upload is abandoned, expiration is an authorization decision as well as a housekeeping event: the session stops accepting receipts, the object-side transfer is discarded, and the document cannot advance from `transferring` to `ready`. This is more code than passing a bucket key through a form, but it makes cross-tenant review and deletion behavior testable.

The upload API should also set limits before bytes move. Check declared size, permitted media type, destination tenant, and session expiry when creating the session. Check the final part list and object metadata before setting `ready`. If scanning or content classification is required, availability must wait for that policy result. No shortcut here.

This direct, resumable design is not suitable when the actual problem is records governance rather than byte transfer. Stick with a server-side upload for small, low-volume internal documents when direct targets add more security review and state management than they remove. Legal hold, evidence retention, regulated workflow, and organization-wide access review may require a specialized content system and a procurement review. For U.S. government workloads, FedRAMP describes a government-wide approach to assessment, authorization, and continuous monitoring of cloud services; encryption on an object alone does not establish that authorization boundary. The service scope and current authorization need to be checked in the relevant program record.

## Test the unhappy path before setting a multipart policy

The clean upload proves little. Run the same 100 MB PDF and 1 GB ZIP through cancellation during a part, a repeated completion request, an expired session, a wrong-tenant completion attempt, and a cleanup sweep. Assert that the document never becomes downloadable until verification succeeds, that abandoned sessions become unavailable, and that a retry retains its previously accepted receipts.

Instrument transitions rather than only endpoint latency. Useful fields are a document ID, tenant ID, session ID, part number, byte count, result category, and a correlation ID. Avoid retaining signed instructions, bearer tokens, or more filename data than the product needs. Track time from session creation to `ready`, retries per part, abort count, cleanup count, and bytes retransmitted. Those signals show whether the failure budget was realistic.

Cost has several components: stored bytes, transfer requests, retransmitted bytes, incomplete-upload retention, download egress, and engineering time in the adapter. Part-based transfer can shrink a retry unit while increasing request count and cleanup obligations. A one-request path has less state, but the full object is the retry unit. Your mileage may vary because the observed network distribution matters more than a fashionable cutoff.

The operational checklist is prose: set the session expiration, limit parts and concurrency, make completion idempotent, clean up expired sessions, and test authorization at both upload creation and download issuance. Review these controls whenever document retention or tenant boundaries change. Keep the selection rule visible in the service configuration so the next person can understand why a PDF and a ZIP follow different paths.

## References

- https://developers.cloudflare.com/r2/
- https://www.fedramp.gov/
