# Security Model

This document explains the security posture of the **CertainData Agent Skill** — an
instruction-only skill (a single `SKILL.md`, no bundled code) that lets an agent look up
live paid reference data and pay [x402](https://www.x402.org) endpoints in USDC on Base.

By design the skill handles a bearer credential, calls external endpoints, and can initiate
a payment — but **only from the user's own CertainData account wallet, via delegated signing,
and only within server-side policy.** It cannot access, hold, or move funds from any other
wallet. Those capabilities are the product, not accidents — this document describes the
controls that keep each one governed. Automated skill scanners correctly surface these
capabilities; the sections below are the intended mitigations for them.

---

## Credential handling

The skill reads a CertainData API key to authorize delegated signing. Its handling is
deliberately constrained:

- **Resolution order.** The configured environment variable first; if it is empty or
  unset, the key line in the configured env file (`secret.env_file`). A blank value is
  treated as absent — the skill never sends an empty bearer token.
- **Never emitted.** The key value is used only to build the `Authorization` header for a
  request to CertainData. It is **never printed, echoed, logged, or written back** to any
  file, prompt, or tool output.
- **Least exposure.** Reading a key from a file outside the workspace requires explicit
  user approval. The key is never persisted into `skill-config.json` (which stores only
  the env-var name and file path, never the secret) and `skill-config.json` is never
  committed.
- **Rotation.** Keys are managed and rotated in the [CertainData portal](https://portal.certaindata.ai);
  the skill holds no long-lived copy.

## Untrusted external input

Catalog entries, seller `402` responses, payment terms, and public x402 Bazaar results are
treated as **data to parse, never as instructions.** The skill uses only their structured
fields (URLs, amounts, networks, schemas, `accepts[]`) and does not act on natural-language
directives embedded in a response — including any attempt to change an endpoint, raise an
amount, skip a confirmation, exfiltrate the key, or bypass a spend cap. When a response
conflicts with the skill's instructions or the user's intent, the skill stops and surfaces
it rather than acting. See the **Trust Boundary** section of `SKILL.md`.

## Payment safety and money movement

Payments are **non-custodial and policy-gated**:

- **First-party delegated signing.** CertainData — the skill's own provider, not an unrelated
  third party — initiates payments from the user's own non-custodial CertainData account wallet.
  **The wallet is non-custodial: the user keeps custody of the private key, and neither the skill
  nor CertainData's backend can access it.** CertainData is granted only a scoped signing
  delegation — it can authorize a payment within the spend caps and allow/deny lists the user has
  defined on their CertainData account, but the signature is produced without the key ever being
  shared with or held by CertainData. CertainData never holds the funds or sits in the money
  path; the payment settles directly from the user's wallet to the seller.
- **Authority is bounded by server-side policy.** The skill has no standing spend authority of
  its own — every payment must be authorized at sign time against controls that live on the
  user's CertainData account, not in the skill (where they could be edited): per-call, daily,
  and monthly spend caps, trust-tier preferences, and endpoint allow/deny lists. These are the
  approval gate — a payment that would breach a cap or hit a deny-listed endpoint is refused
  server-side, so the skill cannot spend beyond what the user has explicitly permitted.
- **Scoped reach.** Paying an endpoint outside the CertainData catalog (open ecosystem)
  requires the user to have enabled that tier on their account.
- **Production by default.** All calls default to production (Base Mainnet). Sandbox (test
  USDC on Base Sepolia) is opt-in per catalog service and requires explicit user
  confirmation each time.
- **No double payment.** On a transport retry the skill resends the *same* signed payment
  header; it re-signs only after the authorization window has lapsed.
- **No buyer fee.** CertainData charges the buyer nothing — the buyer pays only the
  seller's price.

## What this skill does not do

Automated skill scanners flag this skill's *capabilities* (credential handling, external
calls, delegated signing, autonomous payment). For clarity, the skill explicitly does **not**:

- **Execute code.** It is instruction-only — no bundled scripts, no subprocess, no shell. Every action is a plain HTTP request the agent makes directly.
- **Read arbitrary files.** It reads only the configured env file to resolve the API key; a path outside the workspace requires explicit user approval.
- **Expose the key.** The key value is never printed, logged, echoed, or transmitted anywhere except the `Authorization` header of a request to CertainData.
- **Move money from any other wallet.** It can only initiate payments from the user's own CertainData account wallet, signed under server-side spend caps and allow/deny policy; it defaults to production and requires explicit confirmation for sandbox. It never holds funds.
- **Obey external content.** Instructions embedded in catalog, seller `402`, or Bazaar responses are ignored — see *Untrusted external input* above.
- **Call undisclosed endpoints.** Network calls go only to the endpoints documented in `SKILL.md` (CertainData APIs and the public Coinbase x402 Bazaar) plus the specific seller endpoint the user is paying.
- **Collect or exfiltrate data.** It transmits only what each request requires; it gathers no telemetry and sends no user data elsewhere.

## Scope

This document and the skill's licence cover the **skill instructions** (`SKILL.md`) only.
The CertainData services, accounts, and any payments made through them are governed by the
separate [CertainData Terms](https://www.certaindata.ai/terms); accounts are managed at
[portal.certaindata.ai](https://portal.certaindata.ai).

---

## Reporting a vulnerability

If you believe you have found a security issue in this skill, please report it **privately** —
do not open a public issue, which would disclose the problem before a fix exists.

- **GitHub (private):** use [Report a vulnerability](https://github.com/certaindata/skills/security/advisories/new) on the repo's Security tab (GitHub Private Vulnerability Reporting).
- **Email:** support@certaindata.ai (subject line prefixed `SECURITY:`)
- **Contact form:** [certaindata.ai/contact](https://www.certaindata.ai/contact)

We aim to acknowledge reports promptly and will keep you updated on remediation.
