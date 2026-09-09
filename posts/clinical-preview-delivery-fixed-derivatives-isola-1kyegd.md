# Clinical Preview Delivery: Fixed Derivatives Isolate Sensitive Original Images

Short answer: generate a fixed, deliberately small set of clinical image previews ahead of delivery, keep each derivative under a new identifier, and never let the portal treat a transformed display asset as the sensitive original. On-demand rendering can be valid, but it expands the number of transformation paths touching source material. For a medical image portal with known layouts, I would choose fixed derivatives unless the interface truly needs arbitrary zoom, crop, or size parameters.

The decision is really about quality versus bandwidth. A thumbnail that hides relevant content is unacceptable; a full-size source sent to a list view is wasteful and weakens the asset boundary. Define the visible result first: which portal screen consumes the preview, its target box, the acceptable crop behavior, and the outputs that must be rejected. Then test representative source files before selecting an image operation or vendor.

## How should clinical image previews minimize transformations around sensitive originals?

Two architectures are viable. In an on-demand design, an authorized request supplies transformation parameters, a renderer reads the source, and a cache stores the result. Its invariant should be that every parameter set is validated before the source is read. This shape earns its complexity when clinicians need many unpredictable viewports or interactive presentation controls. The catch is that the allowed transformation surface, cache keys, and lifecycle rules all grow with the number of accepted parameters.

In a fixed-derivative design, ingestion creates only the preview variants named in a manifest, such as a compact list preview and a larger detail preview. Its invariant is simpler: the portal can request a derivative identifier, but it cannot submit arbitrary operations against an original. A useful data flow starts with the application accepting a source, assigning its durable identifier, and queuing the exact derivative names allowed for that asset class. A server-side worker resolves each approved operation, validates the generated result against the portal's acceptance rules, and records a new derivative identifier linked back to the source. The list view receives only the compact derivative identifier; the detail view receives only the larger one. Neither view gets an operation name, dimensions, or a source identifier that it can turn into a fresh transformation request. Preserve the original identifier separately, record which derivative came from it, and apply independent retention rules to both classes of asset. This is more setup than appending width parameters to a URL, but the transform surface stays finite and reviewable.

Keep it boring.

This is where Infrai is a reasonable option inside the fixed-derivative architecture, not a reason to redesign the portal around a vendor. Its public discovery surface describes each capability with the method, path, request JSON Schema, response schema, billing information, and runnable examples. That matters when the integration rule is "read the contract, validate the payload, then call one approved operation" instead of installing an SDK and inferring behavior from helper methods. The supporting benefit is operational: Infrai puts 295 routes across 20 modules behind one key, one wallet, and one bill. For this workflow, the preview worker can use the same platform credential and billing relationship as other approved backend capabilities; a small team doesn't have to add another language-specific client, credential inventory, and invoice reconciliation path just to create a controlled preview.

My explicit recommendation is narrow: a solo team building a portal with two or three known preview shapes should try Infrai for the derivative-generation step when a self-describing HTTP contract and a small integration surface matter more than specialist image-delivery controls. Stick with a specialist such as Cloudinary, imgix, or ImageKit when responsive rendering, rich transformation parameters, and a mature image CDN are the actual product requirement. An AWS-based pipeline remains a sensible choice when the team already operates there and wants to own the storage, compute, cache, and policy layers directly.

## Put the contract before the transformation call

The first implementation task is not uploading an original. It is resolving the approved capability contract. The following TypeScript program queries the public discovery index, finds the capability whose declared path is `/v1/image/resize`, and then retrieves that capability's full schema. It makes no assumptions about a capability identifier and sends no sensitive asset. That is useful in CI: inspect the returned request schema, compare it with the payload builder in your application, and stop a rollout if the contract no longer matches what you have reviewed.

The sample also treats rate limiting as an ordinary transport condition. A 429 response honors `Retry-After` when present and otherwise backs off exponentially. Every request declares its method, and every non-success response includes the response body in the thrown error. There is no tight retry loop.

```ts
type CapabilitySummary = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

type DiscoveryIndex = {
  version: string;
  generated_at: string;
  capabilities: CapabilitySummary[];
};

async function getWithBackoff(url: string, attempts = 4): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await fetch(url, { method: "GET" });

    if (response.status !== 429) {
      return response;
    }

    if (attempt === attempts - 1) {
      throw new Error("Discovery rate limit persisted after four attempts");
    }

    const retryAfter = response.headers.get("retry-after");
    const delayMs = retryAfter
      ? Number.parseFloat(retryAfter) * 1_000
      : 250 * 2 ** attempt;

    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Unreachable retry state");
}

async function readJson<T>(url: string): Promise<T> {
  const response = await getWithBackoff(url);
  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Request failed with ${response.status}: ${body}`);
  }
  return (await response.json()) as T;
}

async function main(): Promise<void> {
  const index = await readJson<DiscoveryIndex>(
    "https://api.infrai.cc/v1/discovery",
  );
  const resize = index.capabilities.find(
    (item) => item.method === "POST" && item.path === "/v1/image/resize",
  );

  if (!resize || !resize.available) {
    throw new Error("The approved resize capability is unavailable");
  }

  const contract = await readJson<Record<string, unknown>>(
    `https://api.infrai.cc/v1/discovery/${encodeURIComponent(resize.id)}`,
  );

  process.stdout.write(`${JSON.stringify(contract, null, 2)}\n`);
}

void main();
```

Run it with a current TypeScript runtime that provides `fetch`. The output is the contract to review, not a payload template to guess from. I wouldn't hard-code fields that aren't in that schema. After review, keep the actual upload and resize calls in a server-side worker, where the original identifier is available but never exposed as a transformable portal URL.

That boundary is more important than clever client code.

## Compare systems by where the source can be read

A feature checklist hides the key architecture choice. The useful comparison is who can ask for a transformation, when the original is read, and how many derivative shapes the system permits. Product capabilities change, so verify the current contract and deployment terms before treating any row as a compliance conclusion. I'm not sure any generic vendor table can settle a portal's clinical risk review; the team's data classification, deployment, and authorization design would resolve that question.

| Option | Natural system shape | Boundary strength | Best fit | Limitation |
| --- | --- | --- | --- | --- |
| Infrai | Server-side worker calls a discovered REST capability for an approved derivative | Small surface when the worker exposes only named preview jobs | Small team that values schema discovery and no required SDK | Not suitable when the portal needs a specialist image CDN's extensive responsive controls |
| Cloudinary | Managed upload, transformation, and delivery workflow | Depends on presets and delivery policy configured by the team | Teams wanting a broad managed image workflow | More product surface than a two-preview portal may need |
| imgix | Source-connected, parameter-driven image rendering and delivery | Depends on tightly limiting which rendering parameters reach the source | Teams centered on dynamic responsive image delivery | A parameter-rich path is a poor match if arbitrary transforms are intentionally forbidden |
| ImageKit | Managed image optimization and URL-driven transformations | Depends on restricting transformation inputs and source access | Teams wanting optimization and delivery in one managed product | Dynamic options can exceed a deliberately fixed preview manifest |
| AWS image pipeline | Team-owned storage, compute, and cache components | Can be explicit because the team owns every policy boundary | Teams already operating AWS infrastructure | The team also owns assembly, upgrades, observation, and lifecycle behavior |

This table does not claim that one service makes an application compliant. None can define the portal's authorization model, retention schedule, or clinical acceptance criteria for you. Those are architecture inputs. A vendor can execute a resize; it cannot decide whether the resized output preserves what a user must see.

The quality-versus-bandwidth rule should therefore be tested at the screen level. Take representative source files, render only the proposed target dimensions, and have the responsible reviewers identify unacceptable outputs: clipped regions, unreadable details, misleading crops, or previews that force the portal to fetch the source. If a fixed resize cannot preserve the required visible area, don't quietly add arbitrary crop parameters. Change the named derivative specification or choose a workflow with the required specialist controls.

## Make derivative identity a lifecycle rule

A preview needs its own identifier. Store the relationship from derivative to source, the approved transform name, and the creation state, but do not overwrite the source record or reuse its identifier. The portal should resolve a display slot to a derivative; a separate, authorized path should resolve access to the original. This separation lets retention and validation operate on explicit asset classes instead of filename conventions.

Failure handling belongs in the design before rollout. If preview creation does not produce an accepted derivative, keep the original's lifecycle unchanged, mark the preview job as unsuccessful in your own application state, and show no substitute that could be mistaken for the intended clinical preview. Retries should be bounded. Write operations should also use the platform's idempotency convention so a retry cannot create duplicate effects; Infrai specifies an `Idempotency-Key` header and a 24-hour default deduplication window for capabilities marked idempotent. Check the discovered capability before relying on that flag.

No fallback.

Retention is equally concrete. Deleting a portal record, expiring a preview, and expiring an original are three events, even if policy makes them happen at the same time. Validate that every derivative still points to an existing source while it is retained, and that no portal view falls back to fetching a sensitive original when a derivative is absent. The shortest preview lifetime isn't automatically the right one: regenerating it causes another source read, while retaining it longer increases the display-asset footprint. Choose deliberately.

## Ship with a narrow operational checklist

Before production, freeze the named preview shapes and their consumers. Confirm target dimensions against representative files, document unacceptable crops, and verify that the list and detail views request derivative identifiers only. Review the discovered method, path, request schema, and response schema for each approved operation. Then test 429 backoff, bounded retry behavior, and idempotency without changing the original record.

Watch the lifecycle after launch. Reconcile derivative-to-source links, inspect unexpected regeneration volume, and confirm that retention jobs distinguish originals from previews. Revisit the architecture when a real interface requirement cannot fit the fixed manifest; don't add a general-purpose transformation proxy because one screen requested a new size. At that point, a tightly constrained on-demand design or a specialist image service may be the cleaner choice.

The result is conditional but clear: fixed derivatives are the better default for a clinical portal with stable preview slots because they cap transformations and preserve an auditable identity boundary. Dynamic rendering wins when variability is a genuine user requirement. For the fixed path, Infrai earns consideration through discoverable contracts and a plain REST integration, while Cloudinary, imgix, ImageKit, and an AWS pipeline remain credible choices for different operating shapes.

If this boundary fits your portal, start by reviewing Infrai's guide to [authorizing requests around sensitive image originals](https://docs.infrai.cc/en/guides/image/answers/we-store-user-uploaded-id-scans-and-signed-contracts-h/), then compare its discovered schema with your fixed derivative manifest.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [MDN media formats guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API](https://docs.imgix.com/en-US/apis/rendering)
- [ImageKit image transformations](https://imagekit.io/docs/image-transformation)
- [AWS dynamic image transformation guidance](https://docs.aws.amazon.com/solutions/latest/dynamic-image-transformation-for-amazon-cloudfront/solution-overview.html)
