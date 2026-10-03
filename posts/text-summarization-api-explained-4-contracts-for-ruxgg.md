# Text Summarization API Explained: 4 Contracts for Chat JSON Output

TL;DR: Treat sales-call summarization as extraction into a small, validated contract, not as free-form prose generation. The least complex useful design accepts a transcript, asks a chat-completions-compatible service for one JSON object, validates that object locally, and writes CRM actions only after every cited item can be traced to the source. For a solo builder, this keeps the first release small while protecting the field that matters most: structured output correctness.

A call digest is operational data. If the model turns a vague discussion into an accepted commitment, swaps an owner, or invents a due date, fluent wording does not rescue the result. The boundary should distinguish `null` from an inferred value, preserve short evidence quotes, and reject unknown fields.

No write on ambiguity.

## What should the JSON promise?

Start with four contracts: shape, evidence, uncertainty, and write behavior. Shape defines exactly which keys may cross the boundary. Evidence connects every CRM action to words in the transcript. Uncertainty prevents silence from becoming a fabricated deadline. Write behavior says that an invalid digest is review material, never an automatic CRM mutation.

Keep the schema narrow. A useful first pass needs a summary and proposed actions; sentiment, deal scoring, coaching notes, and account research can wait. Each added field creates another way to be confidently wrong and another test matrix to maintain.

The data flow is plain: an HTTP handler receives a transcript and call identifier, sends a constrained instruction plus the transcript to a configured chat endpoint, parses the assistant's JSON, applies local validation, and returns either a typed digest or a controlled error. A separate worker can later persist approved actions. That separation matters because generation and mutation have different failure costs.

## How should a Node.js chat completions API produce text summarization output?

This example uses the standard `fetch` interface and a generic chat-completions-shaped endpoint. It does not depend on a vendor SDK. The request asks for JSON, but the application still treats the response as untrusted input.

```ts
type Action = {
  text: string;
  owner: string | null;
  dueDate: string | null;
  evidence: string;
};

type CallDigest = {
  callId: string;
  summary: string;
  actions: Action[];
};

type ChatResponse = {
  choices: Array<{ message: { content: string | null } }>;
};

function isNullableString(value: unknown): value is string | null {
  return value === null || typeof value === "string";
}

function parseDigest(value: unknown, expectedCallId: string): CallDigest {
  if (typeof value !== "object" || value === null || Array.isArray(value)) {
    throw new Error("Digest must be an object");
  }

  const record = value as Record<string, unknown>;
  const allowed = new Set(["callId", "summary", "actions"]);
  if (Object.keys(record).some((key) => !allowed.has(key))) {
    throw new Error("Digest contains an unknown field");
  }
  if (record.callId !== expectedCallId || typeof record.summary !== "string") {
    throw new Error("Digest identity or summary is invalid");
  }
  if (!Array.isArray(record.actions)) throw new Error("Actions must be an array");

  const actions = record.actions.map((item): Action => {
    if (typeof item !== "object" || item === null || Array.isArray(item)) {
      throw new Error("Action must be an object");
    }
    const action = item as Record<string, unknown>;
    const keys = new Set(["text", "owner", "dueDate", "evidence"]);
    if (Object.keys(action).some((key) => !keys.has(key))) {
      throw new Error("Action contains an unknown field");
    }
    if (
      typeof action.text !== "string" ||
      !isNullableString(action.owner) ||
      !isNullableString(action.dueDate) ||
      typeof action.evidence !== "string"
    ) {
      throw new Error("Action fields are invalid");
    }
    return {
      text: action.text,
      owner: action.owner,
      dueDate: action.dueDate,
      evidence: action.evidence
    };
  });

  return { callId: record.callId, summary: record.summary, actions };
}

export async function summarizeCall(
  callId: string,
  transcript: string
): Promise<CallDigest> {
  const endpoint = process.env.SUMMARY_API_URL;
  const token = process.env.SUMMARY_API_TOKEN;
  const model = process.env.SUMMARY_MODEL;
  if (!endpoint || !token || !model) throw new Error("Missing runtime configuration");

  const response = await fetch(endpoint, {
    method: "POST",
    headers: {
      authorization: `Bearer ${token}`,
      "content-type": "application/json"
    },
    body: JSON.stringify({
      model,
      messages: [
        {
          role: "system",
          content: [
            "Return one JSON object with callId, summary, and actions.",
            "Each action has text, owner, dueDate, and an exact evidence quote.",
            "Use null when owner or due date is not explicit. Do not infer commitments.",
            "Return no keys beyond those requested and no Markdown."
          ].join(" ")
        },
        { role: "user", content: JSON.stringify({ callId, transcript }) }
      ],
      response_format: { type: "json_object" }
    })
  });

  if (!response.ok) throw new Error(`Summary request failed with ${response.status}`);
  const payload = (await response.json()) as ChatResponse;
  const content = payload.choices[0]?.message.content;
  if (!content) throw new Error("Summary response was empty");

  let decoded: unknown;
  try {
    decoded = JSON.parse(content);
  } catch {
    throw new Error("Summary response was not valid JSON");
  }
  return parseDigest(decoded, callId);
}
```

The example is deliberately strict and small. `response_format` expresses the desired transport shape where that convention is supported; it does not replace application validation. The local parser rejects extra top-level and action fields, checks the echoed identifier, and retains `null` as a meaningful answer.

There is one subtle gap to close before CRM writes: `evidence` must occur in the submitted transcript. Add a deterministic substring check after normalizing only line endings, not punctuation or wording. Exact quotes make review cheap and expose unsupported actions without asking another model to judge the first one. If the transcript has speaker timestamps, retain them in the input and add a source locator to the contract rather than asking the model to recreate timing.

## Why can valid JSON still be wrong?

Syntax is the shallowest layer of correctness. A response can parse cleanly while assigning “send the security packet” to the buyer instead of the account executive. It can convert “sometime next quarter” into a precise date. It can also merge two tentative ideas into one committed action.

This is why a single `JSON.parse` check is inadequate. Validate in layers: transport, schema, source support, and business policy. Transport asks whether a response arrived and can be decoded. Schema checks types, required keys, nullability, and unknown keys. Source support confirms that evidence is present. Business policy decides whether an action is eligible for automatic creation; for example, an explicit owner may be mandatory even though the extraction schema permits `null`.

**Valid structure is admission to review, not proof of truth.**

Long calls add a different pressure. Do not split a transcript at arbitrary character counts, because a commitment can straddle the boundary. Prefer speaker-turn boundaries and carry a small overlap. Extract candidate actions per segment, then run a deterministic merge keyed from normalized action text, owner, and evidence location. A final model pass may improve prose, but it should not silently create new actions that were absent from segment outputs.

Use one short retry only for failures that may change on repetition, such as malformed JSON or a transient transport response. For a `429` response, honor a server-provided retry delay when available or apply bounded exponential backoff with jitter. Validation failures caused by unsupported evidence should not be “fixed” by repeatedly prompting until something passes. Route them to review with the original response and a reason code.

Less magic. Easier operations.

## Test the boundary, then ship it

A happy-path transcript proves very little. Build fixtures around the mistakes that would create bad CRM work: no action at all, an unnamed owner, a relative date, a negated commitment, two people volunteering for the same task, and transcript text that contains instructions aimed at the summarizer. The transcript is data, even when it says “ignore the previous instruction.”

For each fixture, assert the entire object rather than checking that a summary contains a keyword. Also assert rejection behavior. An extra key, a numeric owner, a changed `callId`, or evidence that cannot be found in the input should fail before persistence. Keep those tests independent of live model calls so they run quickly on every change; store a smaller set of model-backed cases for scheduled checks against the configured runtime.

The useful metrics follow the contract. Track parse rejection rate, schema rejection rate, unsupported-evidence rate, actions sent to review, and actions accepted after review. Record latency and input/output token counts beside those outcomes, because a faster or shorter response has no value if it creates cleanup work. Avoid logging raw transcripts by default; sales calls can contain personal and commercially sensitive material. Use call identifiers, reason codes, counts, and carefully redacted samples for diagnosis.

Prompt text, schema, model configuration, and validator code form one release unit. Version them together. When any one changes, replay the fixed fixture set and compare typed outputs. That is a practical deployment gate, not a research project.

The production checklist can remain prose. Set hard request timeouts and transcript-size limits. Keep credentials server-side. Separate extraction from CRM mutation, make the eventual write idempotent under a call identifier plus action key, and send ambiguous records to a visible review queue. Log contract failures without retaining the conversation itself, then watch correctness and latency by release version.

This approach has real limitations. Exact substring evidence is cheap and auditable, but it will reject valid paraphrases and can miss matches after transcription normalization. Automatic writes are not suitable when the business needs human interpretation of implied commitments, when speaker attribution is unreliable, or when a mistaken task has a high downstream cost. In those cases, use the same typed output as a review draft instead. The trade-off is slower CRM entry in exchange for an explicit approval boundary.

Start with actions that include exact evidence and explicit nulls. Leave deal scores and elaborate narrative summaries out until the action workflow is stable. This choice reduces token use, validator surface, and reviewer effort at the same time; cost improves as a consequence rather than becoming the design's headline.

The final decision rule is blunt: automate a CRM action only when its object passes local validation, its evidence exists in the submitted transcript, and its required business fields are explicit. Everything else is a draft.

## Further reading

- JSON Schema specification: https://json-schema.org/specification
- MDN `fetch()` reference: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
- OWASP guidance for large language model applications: https://owasp.org/www-project-top-10-for-large-language-model-applications/
- OpenAI Batch API guide: https://platform.openai.com/docs/guides/batch
- Prompt Engineering Guide: https://www.promptingguide.ai
