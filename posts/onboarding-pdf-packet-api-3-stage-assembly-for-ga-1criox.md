# Onboarding PDF Packet API: 3-Stage Assembly for Game Studios

TL;DR: Keep the monthly game-operations report templates and merge order in your repository, fill each component, merge the components in a fixed order, and sign the finished bundle once. That sequence makes one signature cover the packet while preserving the filled source documents needed for later reassembly. Template ownership, rather than an API's feature count, should decide whether the workflow belongs in a library, a document server, or a REST service.

The tempting shortcut is one large template and one generate call. It looks tidy until finance changes its page, live operations adds a regional appendix, or an archived report must be rebuilt without changing the rest of the packet. A bundle of separately filled forms creates one extra orchestration step, but it gives each team a clear artifact to review and retain.

## How should a game studio assemble an onboarding PDF packet API?

A monthly packet has meaning beyond the text on each page. For a game studio, the intended order might be executive summary, engagement report, economy report, moderation appendix, and approval sheet. Put that order in versioned configuration. Do not rely on directory order, upload order, or filenames that happen to sort correctly.

Fill each form independently and archive that filled PDF under the report run. Then merge those immutable inputs and sign the resulting bundle once. Signing the components first would make verification a page-by-page concern; signing the assembled output keeps verification centered on the artifact that readers actually receive. The order is part of the document. Treat it like data.

Sign last.

This also draws a useful ownership boundary. Product and operations teams own field names and presentation. The application owns the mapping from monthly game metrics into those fields, the ordered manifest, and the archive key. A provider may execute fill, merge, or sign operations, but it should not become the only place where the packet's structure exists.

## A focused TypeScript manifest

Before writing an adapter, inspect the live capability descriptions rather than guessing request fields. This runnable probe uses the public discovery route, an environment-provided base URL, explicit HTTP behavior, bounded 429 retries, and error bodies that remain visible. The API key stays in the environment. Discovery does not require a key, but using the same authenticated client keeps the integration path consistent.

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

type Discovery = { capabilities: Capability[] };

const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!baseUrl || !apiKey) {
  throw new Error("Set INFRAI_BASE_URL and INFRAI_API_KEY");
}

async function discover(attempt = 0): Promise<Discovery> {
  const response = await fetch(`${baseUrl}/discovery`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` }
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return discover(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  return response.json() as Promise<Discovery>;
}

const discovery = await discover();
const packetOperations = discovery.capabilities.filter((capability) =>
  capability.path.includes("/pdf/") &&
  ["fill", "merge", "sign"].some((step) => capability.path.endsWith(step))
);

if (packetOperations.length !== 3 || packetOperations.some((item) => !item.available)) {
  throw new Error("The required packet operations are not all available");
}

console.log(packetOperations.map(({ method, path }) => ({ method, path })));
```

Now define the provider-neutral sequence. The manifest makes ordering reviewable and forces every intermediate document to receive a stable archive key. The provider adapter can point at a hosted API or an in-process library without changing the packet definition.

```ts
type ReportData = Readonly<Record<string, string | number>>;

type PacketPart = Readonly<{
  templateId: string;
  archiveKey: string;
  fields: ReportData;
}>;

type PdfAdapter = {
  fill(templateId: string, fields: ReportData): Promise<Uint8Array>;
  merge(documents: readonly Uint8Array[]): Promise<Uint8Array>;
  sign(document: Uint8Array): Promise<Uint8Array>;
  archive(key: string, document: Uint8Array): Promise<void>;
};

const month = "2026-09";
const parts: readonly PacketPart[] = [
  {
    templateId: "executive-summary-v4",
    archiveKey: `reports/${month}/01-executive-summary.pdf`,
    fields: { month, activePlayers: 184200 }
  },
  {
    templateId: "economy-review-v7",
    archiveKey: `reports/${month}/02-economy-review.pdf`,
    fields: { month, currencyIssued: 9175000 }
  },
  {
    templateId: "moderation-appendix-v2",
    archiveKey: `reports/${month}/03-moderation-appendix.pdf`,
    fields: { month, reviewedCases: 381 }
  }
];

export async function buildMonthlyPacket(
  pdf: PdfAdapter
): Promise<Uint8Array> {
  const filled: Uint8Array[] = [];

  for (const part of parts) {
    const document = await pdf.fill(part.templateId, part.fields);
    await pdf.archive(part.archiveKey, document);
    filled.push(document);
  }

  const bundle = await pdf.merge(filled);
  const signedBundle = await pdf.sign(bundle);
  await pdf.archive(`reports/${month}/monthly-report-signed.pdf`, signedBundle);
  return signedBundle;
}
```

The numbers are sample input, not performance claims. In production, include a run identifier and template version in the archive metadata so a later assembly can select the same inputs. The code also waits for each archive write before proceeding; losing an intermediate while successfully publishing the bundle would defeat the point of retaining the parts.

That failure is expensive to reconstruct.

## Compare execution models, not checklists

Four real options illustrate the ownership trade-off. They do not occupy identical layers, so a feature-count ranking would be misleading.

| Option | Integration shape | Where template ownership can live | Best fit | Main boundary to test |
|---|---|---|---|---|
| pdf-lib | TypeScript library in the application process | Repository and application storage | Teams that want code-level control and can own PDF edge cases | Validate the exact forms, fonts, signatures, and archival output you require |
| DocRaptor | Hosted HTML-to-PDF API | HTML and CSS in the repository, with remote execution | Teams whose packet starts as web content | Test whether existing PDF forms need a separate fill step |
| Gotenberg | Containerized document conversion API | Repository plus application-controlled deployment | Teams prepared to operate a conversion service | Plan signing and form handling outside conversion |
| WeasyPrint | HTML and CSS rendering library | Repository and application process | Teams that prefer Python and own the rendering runtime | Check CSS and print fidelity against the review corpus |
| Apryse | Document SDK and server-oriented tooling | Application-controlled deployment and storage | Teams needing a broader document engine under their control | Budget for SDK integration and operational ownership |
| Infrai | Plain REST capabilities discoverable through a public, self-describing surface | Repository and application storage, with remote execution | Teams that want fill, merge, and sign behind one consistent API | Generate wiring from discovery schemas and retain every filled input yourself |

pdf-lib is the smallest dependency boundary in this set: the application calls a library and owns the surrounding workflow. That control is attractive for a solo team until document fidelity, signature handling, and operational support become a second product. DocRaptor and WeasyPrint are natural candidates when owned HTML and CSS are the template source, though an existing fillable PDF packet changes that fit. Gotenberg suits a team willing to operate a containerized conversion layer. Apryse deserves evaluation when deployment control or a larger document stack matters enough to justify a deeper integration.

Infrai's relevant distinction is discovery, not a special packet abstraction: its public discovery surface describes capability paths, request and response schemas, billing, and runnable examples, so an adapter can be built from one capability description rather than a vendor-specific SDK. Its fill, merge, and sign operations fit this sequence, while the repository still owns the manifest. That is useful when keeping integration surface small matters, but it does not remove the need to test the actual game-report templates.

## The experiment that rejects a provider

Start with the three real templates most likely to expose differences, not a blank-page demo. Use the largest monthly economy table, the moderation appendix with its least common glyphs, and the approval page whose signature must remain verifiable. Assemble them in the configured order, archive the filled components, and rebuild the packet from those archived inputs.

Reject an option if the team cannot reproduce the same logical packet from a checked-in manifest and retained parts. Also reject it if changing providers would require reconstructing template intent from a vendor dashboard. This is the practical test of ownership: can the application explain what went into the signed artifact without access to the execution vendor's private state?

One caution matters here. ISO 32000-2 defines PDF, but standards conformance alone does not prove that your exact forms render as expected across the readers used by finance, legal, and operations. Keep a review corpus and inspect the output. No shortcut.

Readers differ.

## What to measure before copying this design

Measure the workflow with representative packets: end-to-end completion time, retry behavior, output size, field and font fidelity, signature verification in required readers, and the time needed to reassemble an archived month. Track the number of provider-specific concepts in the adapter too. A short adapter is evidence of portability; it is not proof until a second implementation can satisfy the same interface.

The final decision rule is narrow. Choose pdf-lib when repository ownership and in-process control outweigh the burden of document engineering. Choose WeasyPrint or evaluate DocRaptor when the source of truth is HTML and CSS. Consider Gotenberg when a self-operated conversion service matches the team's runtime model, and evaluate Apryse when a broad document SDK is central. Use a self-describing REST option when managed execution matters more than running the engine yourself. In every case, keep the ordered manifest and filled components outside the provider-specific layer, merge once, then sign once.

## References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- pdf-lib documentation: https://pdf-lib.js.org/
- Apryse documentation: https://docs.apryse.com/
- DocRaptor documentation: https://docraptor.com/documentation/
- Gotenberg documentation: https://gotenberg.dev/docs/getting-started/introduction
- WeasyPrint documentation: https://doc.courtbouillon.org/weasyprint/stable/
