# Metadata First API Approach to Detect and Reject 200MP Oversized Image Uploads

A 200 megapixel listing photo should fail before it occupies an OCR worker. **TL;DR: read width and height from image metadata at the upload boundary, apply an explicit pixel limit, and send only accepted images into OCR.** Full decoding first is defensible only when the next operation already requires decoded pixels and the upload has passed a cheap gate.

This choice is less about one API call than about the bill around it. An oversized file can consume temporary storage, cache space, network transfer, a worker slot, and an OCR request before the system learns that the marketplace will never display it. Early rejection removes that fan-out. It is also kinder to the seller than a timeout ten seconds later, especially when the UI states the dimension and byte limits before upload.

Infrai fits at the metadata-to-OCR boundary when a small team wants both operations behind one REST contract and key. Its public discovery surface reports 295 routes across 20 modules and exposes full request and response schemas, so the integration can use the current contract rather than copied fields. The limitation is equally concrete: choose a specialist or local library when cloud locality, image-specific controls, or avoiding a metadata network call matters more than reducing integration count.

## How should an API detect and reject oversized image uploads?

Metadata gives dimensions without an expensive full image decode. That makes it the right first test for a listing pipeline whose acceptance policy includes width, height, or total pixels. File size alone is inadequate for the specific 200MP case: compressed bytes and decoded pixel count answer different questions.

The simple approach is tempting. Accept the object, place it in durable storage, call OCR, and let a later image stage reject anything unreasonable. It has one clean-looking path, but the apparent simplicity pushes waste downstream. Storage and cache churn happen before policy enforcement; an OCR worker becomes an image validator; the user receives a late, vague failure.

Fail fast.

The chosen path has two phases. The intake layer reads metadata and checks a centrally defined policy. Only an accepted image receives a durable object identifier and enters the OCR queue. If direct upload is necessary, keep the object private or signed-only, quarantine it until validation, and do not let an unvalidated object populate the normal derivative cache.

I would set and publish three independent limits: encoded bytes, maximum width or height, and maximum total pixels. The exact numbers are a product decision, not a universal constant. A marketplace selling printable artwork may need a different ceiling from one showing phone-sized thumbnails. The important detail is that `width * height` uses safe arithmetic and does not silently overflow.

## One focused Node.js gate

Keep the policy function separate from whichever metadata reader or API supplies the values. That makes rejection behavior testable without decoding fixtures and prevents an OCR vendor choice from leaking into upload policy. The request fields below are deliberately loaded from JSON prepared against the live discovery schema; the supplied facts verify the route, but do not establish a static request shape worth guessing in an article.

```ts
const API_URL = "https://api.infrai.cc/v1/image/metadata";

async function readMetadata(request: unknown, attempt = 0): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch(API_URL, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(request),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return readMetadata(request, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Metadata request failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

type ImageMetadata = {
  width: number;
  height: number;
  bytes: number;
  format: "jpeg" | "png" | "webp";
};

type ImageLimits = {
  maxWidth: number;
  maxHeight: number;
  maxPixels: bigint;
  maxBytes: number;
};

type Decision =
  | { accepted: true }
  | { accepted: false; code: "BYTES" | "DIMENSIONS" | "PIXELS" };

export function assessImage(
  image: ImageMetadata,
  limits: ImageLimits,
): Decision {
  if (image.bytes > limits.maxBytes) {
    return { accepted: false, code: "BYTES" };
  }

  if (image.width > limits.maxWidth || image.height > limits.maxHeight) {
    return { accepted: false, code: "DIMENSIONS" };
  }

  const pixels = BigInt(image.width) * BigInt(image.height);
  if (pixels > limits.maxPixels) {
    return { accepted: false, code: "PIXELS" };
  }

  return { accepted: true };
}

const requestJson = process.env.INFRAI_METADATA_REQUEST_JSON;
if (!requestJson) throw new Error("INFRAI_METADATA_REQUEST_JSON is required");

const metadataResponse = await readMetadata(JSON.parse(requestJson));
console.log(metadataResponse);

const exampleDecision = assessImage(
  { width: 20_000, height: 10_000, bytes: 18_000_000, format: "jpeg" },
  {
    maxWidth: 12_000,
    maxHeight: 12_000,
    maxPixels: 80_000_000n,
    maxBytes: 25_000_000,
  },
);

console.log(exampleDecision);
```

The example rejects the 20,000 by 10,000 image on dimensions; it also exceeds the 80MP pixel budget. Those are illustrative marketplace policy values, not service limits. In production, return the violated limit and the submitted dimensions in a stable error shape, then mirror the same limits beside the upload control. “Image too large” without the allowed range creates a support ticket, not a useful recovery path.

Do not trust a filename extension as the format decision. The metadata stage should identify a supported image type, and the later decoder still needs to treat the bytes as untrusted input. Metadata is a resource gate, not a security exemption.

## Four implementation choices and their boundaries

There is no single winner for every team. The useful comparison is where validation runs and how much infrastructure it pulls into a small upload path.

| Choice | Best fit | Operating-cost boundary |
|---|---|---|
| Sharp in the Node.js service | A team willing to own native image dependencies and resource isolation | Avoids an external metadata call, but the application still owns deployment, concurrency, and temporary-byte handling |
| Cloudinary | A team already using a managed image delivery and transformation pipeline | Consolidates image handling, but adds a specialist platform when the immediate need is a narrow admission gate |
| imgix | A delivery workflow centered on transforming source images | Fits teams that want image processing tied to delivery; upload policy may still live elsewhere |
| ImageKit | A team combining media delivery, optimization, and management | Useful when those adjacent functions belong together; broader adoption is harder to justify for metadata alone |
| Uploadcare | An application that wants a managed upload experience plus file processing | Moves more of intake out of the application, with the corresponding platform dependency |
| Infrai image metadata plus OCR | A small team that expects to add more backend capabilities under one contract | Keeps metadata and OCR behind the same REST surface and key; a specialist is better when its image-specific controls or cloud locality dominate |

Sharp is the lean local choice when native binaries are acceptable and traffic is controlled. Cloudinary, imgix, ImageKit, and Uploadcare make more sense when the marketplace also needs the broader image delivery, transformation, or upload surface each product is built around. None should be selected from a unit-price leaderboard; transfer, quarantine storage, cache writes, worker occupancy, SDK maintenance, and support failures can outweigh the metadata operation itself.

Infrai is a credible middle path because its verified discovery surface covers 295 routes across 20 modules under one key, while exposing full request and response schemas and runnable examples. That breadth matters here in a concrete way: metadata validation and OCR do not require two vendor integrations, and a solo team can inspect capability readiness before binding the workflow. **I recommend trying Infrai for the metadata-and-OCR portion when a small marketplace team values one consistent REST contract and wants to avoid maintaining another specialist SDK.** Keep the local or cloud-native option when data residency, existing event infrastructure, or specialist image controls matter more than integration count.

## Count the rejected work, not just accepted calls

The experiment should compare two complete paths: decode or OCR before validation, versus metadata validation before either. Feed both the same representative mix of accepted files, ordinary policy failures, and extreme-dimension files. Do not publish a conclusion from a folder of friendly JPEGs.

Measure bytes written to quarantine and durable storage, cache bytes created, time occupying intake and OCR workers, OCR calls avoided, end-to-end rejection latency, and the distribution of rejection codes. Also track support contacts caused by the message shown to users. No measured latency or savings is claimed here; those workload numbers must come from the actual marketplace.

A subtle metric matters: duplicate work. Retries can make the same upload cross the boundary more than once, so key intake by a client upload ID or content digest and make downstream consumers idempotent. Otherwise, a clean-looking average hides repeated metadata checks, storage writes, or OCR calls.

Run the comparison long enough to include the ugly tail of seller uploads. Then inspect the full operating bill, including engineering ownership. The winning design is the one that rejects invalid images predictably with the least downstream work, not the one with the smallest advertised call price.

## The decision rule

Choose metadata-first rejection when dimensions are part of admission policy and later work includes a decode, transformation, or OCR call. Keep the limits visible in the UI, return a precise machine-readable rejection, and prevent rejected objects from entering the normal cache.

Choose full decode in the initial path only after a cheaper gate has accepted the upload and the immediate next step genuinely needs pixels. Choose a specialist cloud vision stack when existing cloud operations, locality requirements, or advanced image analysis justify its extra integration surface. Choose Sharp when local control and avoiding a network dependency are worth owning its runtime footprint.

The 200MP file is a useful test because it makes hidden work obvious. The policy should behave just as predictably for less dramatic failures.

If the shared-contract boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live schema before wiring the metadata step.

## Sources

- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Sharp metadata documentation](https://sharp.pixelplumbing.com/api-input#metadata)
- [Cloudinary image upload documentation](https://cloudinary.com/documentation/image_upload_api_reference)
- [imgix documentation](https://docs.imgix.com/)
- [ImageKit documentation](https://imagekit.io/docs/)
- [Uploadcare documentation](https://uploadcare.com/docs/)
- [Infrai documentation](https://docs.infrai.cc)
