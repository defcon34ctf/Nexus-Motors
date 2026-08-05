# ARIA — System Overview for Threat Modeling

**A working document for the audience/participant exercise.**
This describes what the system *is* and how it's built — not where the
bugs are. Use it to threat model the application yourself before (or
alongside) playing the live CTF, then compare what you predicted against
what you find.

---

## 1. System purpose & context

ARIA is the in-vehicle AI assistant for a fictional automaker, Nexus
Motors. Drivers talk to it through a touchscreen-style web interface to
ask about their vehicle (range, charging, warranty, diagnostics), read
manufacturer content (service bulletins, customer reviews), and manage a
personalization setting (a driver nickname).

This is representative of a real product category: automotive
manufacturers are increasingly embedding conversational AI assistants
directly in vehicle infotainment systems and companion apps, often wired
into vehicle telemetry, customer records, and manufacturer content feeds.

## 2. Actors

| Actor | Description |
|---|---|
| Driver / end user | Interacts with ARIA through the web UI. Fully untrusted from the system's perspective — anything they type or configure should be treated as attacker-controlled input. |
| Nexus Motors (backend/content owner) | Publishes vehicle content ARIA can reference: service bulletins, aggregated customer reviews, vehicle records. |
| Third-party content sources | Customer reviews, in this design, originate from outside Nexus Motors entirely (e.g. a review aggregator). Anything from here should be treated as untrusted, even though ARIA is set up to consume it as if it were first-party. |

## 3. Architecture (high level)

```
 ┌────────────────────────┐        HTTPS         ┌─────────────────────────┐
 │   Browser (client)      │ ───────────────────▶ │  Serverless API layer    │
 │                          │                       │                          │
 │  - Chat UI               │                       │  /api/chat               │
 │  - Renders ARIA's replies│ ◀─────────────────── │  - "system prompt" /     │
 │  - Driver nickname field │        JSON            │    persona + rules       │
 │  - Session bootstrap     │                       │  - reads user message    │
 │                          │                       │  - reads ingested docs   │
 └────────────────────────┘                       │  - applies a safety      │
                                                     │    filter before some    │
                                                     │    responses             │
                                                     │                          │
                                                     │  /api/session            │
                                                     │  - issues a session      │
                                                     │    cookie on page load   │
                                                     └─────────────────────────┘
```

Everything the browser needs to render the page is static (HTML/CSS/JS).
All "intelligence" — the persona, the content ARIA can reference, and any
rules about what she will or won't disclose — lives behind the two API
endpoints.

## 4. Components

| Component | Responsibility | Trust level |
|---|---|---|
| Chat UI (client) | Collects user input, sends it to the API, renders whatever comes back | Untrusted — runs entirely on the user's machine, fully visible/modifiable by them |
| Driver nickname field | A personalization setting, round-tripped through the API and reflected back into ARIA's replies | Untrusted input, but treated by the rest of the system as a normal display string |
| `/api/chat` | Assembles ARIA's "instructions" plus whatever context is relevant (the user's message, and sometimes a document ARIA is asked to summarize), and decides how to respond | Server-side, but ingests untrusted input as part of its own reasoning process |
| Ingested documents (reviews, bulletins) | Represent third-party or manufacturer content ARIA can be asked to read and summarize | Origin varies — should NOT be assumed trustworthy just because it's served from Nexus Motors' own systems |
| Sensitive vehicle/customer record | Contains PII and an internal credential-like value that ARIA has access to but is meant to protect | High sensitivity |
| Safety filter | A rule-based check meant to stop ARIA from disclosing the sensitive record on request | Server-side control, but only as strong as its actual rule coverage |
| `/api/session` | Establishes a session identifier for the driver's visit | Server-side |

## 5. Data assets

| Asset | Sensitivity | Where it lives |
|---|---|---|
| Driver nickname | Low sensitivity on its own, but it's a value the user fully controls that later gets displayed back to them (and potentially to others, in a real multi-user product) | Round-trips client → API → client |
| Vehicle/customer record (owner PII + an internal key) | High — this is the kind of data a real telematics/CRM integration would hold | Held server-side, disclosed only through ARIA's responses |
| Session identifier | Medium — identifies a driver's session | Issued server-side, stored client-side |
| Ingested third-party content | Untrusted by nature, but treated by the system as safe to fold into ARIA's own context | Passed into the assistant's reasoning pipeline |

## 6. Data flows worth mapping

Draw these out yourself and mark where trust changes:

1. Driver types a message → sent to `/api/chat` → becomes part of what
   ARIA "reasons" over → a reply is generated → sent back to the browser
   → rendered into the page.
2. Driver sets a nickname → sent to `/api/chat` → included in ARIA's
   reply text → rendered into the page.
3. ARIA is asked to reference a document (review/bulletin) → that
   document's content is pulled into the same reasoning process as the
   driver's own message and Nexus Motors' own instructions to ARIA →
   a reply is generated.
4. Page loads → `/api/session` issues a session identifier → stored
   client-side → available to any code running on the page.
5. Driver asks about the vehicle/customer record → a safety filter
   evaluates the request → either blocks it or lets ARIA answer.

## 7. Controls already in place (as designed)

- A persona/instruction layer that's supposed to keep ARIA "in character"
  and scoped to vehicle-support topics.
- A rule-based safety filter in front of the sensitive vehicle/customer
  record, meant to block direct requests for it.
- Client-side rendering of ARIA's replies into the chat window.
- A session cookie issued per visit.

Whether each of these controls actually holds up under adversarial input
is exactly what you're being asked to evaluate.

## 8. Exercise: threat model it yourself

Work through this before (or instead of) jumping straight to exploitation.
For each component above, ask:

- **Spoofing** — can anything pretend to be something it isn't (a
  trusted instruction pretending to come from Nexus Motors rather than
  from user-controlled input, for example)?
- **Tampering** — where does content get mixed together (e.g. a
  document's content and the system's own instructions) without a clear
  boundary between them?
- **Repudiation** — is there any record of what ARIA was told versus
  what she decided to do?
- **Information disclosure** — trace every path by which the sensitive
  record *could* reach the client, not just the "front door." Also trace
  where session or personalization data ends up and who/what can read it.
- **Denial of service** — out of scope for this exercise, but worth a
  mention: what happens with extremely long or repeated inputs?
- **Elevation of privilege** — can a driver's own input cause ARIA to
  behave with more authority or fewer restrictions than intended?

Specific prompts:

1. **Trust boundaries.** Draw a boundary around "things Nexus Motors
   controls" and another around "things the driver controls." Now trace
   where those boundaries actually get enforced in the data flows above
   — versus where content just gets concatenated together and treated
   the same. Where do document contents cross from "external content"
   into "instructions ARIA follows"?
2. **Output trust.** ARIA's replies get rendered into a web page. What
   assumptions is the client making about the safety of that text? Where
   did that text ultimately originate, and how much of it did the driver
   influence?
3. **Filter coverage.** The safety filter in front of the sensitive
   record checks for certain phrasings. Without seeing its exact rules,
   what categories of rephrasing, encoding, or social framing would you
   expect a keyword-based filter to miss?
4. **Data minimization.** Does ARIA need access to the full sensitive
   record to do her job, or could the system be redesigned so she never
   holds data she doesn't need to disclose?
5. **Redesign.** For each risk you identify, sketch (in a sentence or two)
   what a fix would look like — and note whether the fix belongs in the
   model/prompt layer, the application layer, or the infrastructure layer.
   (Real-world hint: most durable fixes to these classes of bugs live
   outside the model itself.)

Once you've done this analysis, play the live challenge and compare notes
— did the exploitable paths match what your threat model predicted?
