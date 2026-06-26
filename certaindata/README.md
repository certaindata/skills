# CertainData Skill

The CertainData skill gives AI agents access to live paid reference data and autonomous x402 payments — without the agent needing to hold a wallet or private keys. CertainData signs each payment from your own CertainData account wallet and applies your spend controls — and **takes no fee on your x402 payments; you pay only the seller's price.**

---

## What It Does

**Data lookups** — the skill matches your query to the live CertainData paid service catalog and handles the rest. Supported data includes FX rates, BIN/card metadata, country data, public holidays, IP geolocation, and more — with new services added to the catalog without requiring a skill update.

**x402 payments** — when your agent needs to pay any x402 endpoint (inside or outside the CertainData catalog), CertainData signs the payment from your CertainData account wallet so the skill can complete the call. You never need a wallet, a private key, or to sign or fund the payment yourself — the skill handles the 402 exchange and payment header for you.

---

## Choosing a Mode

The skill runs in one of two modes, chosen once at first setup:

| Mode | CertainData account | What the skill does |
|---|---|---|
| `certaindata_flow` | Required | For both catalog data lookups and external x402 endpoints, CertainData signs the payment from your CertainData account wallet (applying your spend controls) and the skill completes the call and returns the data. No payment plumbing on your side, and CertainData takes no fee on the payment. Sandbox (test USDC on Base Sepolia) is available for catalog services that support it — CertainData funds your test wallet and signs the test payment for you |
| `catalog_only` | Not required | Matches your query to the catalog and returns the request blueprint and payment terms for you to pay yourself |

**Use `certaindata_flow`** if you want CertainData to handle payment for you — for both catalog data lookups and any x402 endpoint — signing from your CertainData account wallet with your spend caps and allow/deny lists enforced automatically.

**Use `catalog_only`** if you already have your own x402 wallet or tooling and just want the catalog matching and request details.

---

## How Payments Work

In `certaindata_flow` mode you never handle a wallet, a private key, or a payment header. When your agent reaches a paid endpoint:

1. It calls the endpoint and receives a `402 Payment Required`.
2. CertainData signs the payment from your CertainData account wallet — applying your spend caps and allow/deny lists — and returns a payment header.
3. Your agent retries the call with that header; the payment settles on-chain and the data comes back with a settlement transaction.

The payment goes straight from your wallet to the seller — CertainData signs it but never holds your funds.

You only ever need USDC in your wallet — never ETH. Network (gas) fees are sponsored, so there is nothing else to top up.

In `catalog_only` mode CertainData signs nothing. Your agent pays with its own wallet or tooling, using the request details and payment terms the skill hands back.

---

## Trust Tiers

Every catalog service carries a trust tier, returned with the catalog so you and your agent can see what a service is before paying:

- **CertainData Verified** — partner services CertainData has vetted for data quality, uptime, freshness, and licence validity.
- **Buyer Approved** — self-onboarded catalog services. CertainData does not vouch for their quality, uptime, or licensing.
- **Open Ecosystem** — endpoints outside the catalog altogether (the public x402 Bazaar, or a URL you supply). Not vetted; CertainData only signs the payment.

In `certaindata_flow` the tiers also act as a spending preference: CertainData signs a payment only for tiers you've enabled on your account — Verified is always on; Buyer Approved and Open Ecosystem are optional toggles in the [portal](https://portal.certaindata.ai). The tiers stack (each includes the one above it), and a payment is never signed if the endpoint is on your deny list or would breach a spend cap.

---

## Installation

**Via the skills CLI:**

```shell
npx skills add certaindata/skills --skill certaindata
```

This installs the skill into every detected agent platform on your machine simultaneously.

**Manual install:**

Download `SKILL.md` from either location and place it in your agent platform's skills directory. Refer to your agent platform's documentation for the correct path.

- [github.com/certaindata/skills](https://github.com/certaindata/skills) (in the `certaindata/` folder)
- [certaindata.ai/for-agents](https://www.certaindata.ai/for-agents)

**Verify it works:**

Ask your agent to run a quick lookup:

```
Look up BIN 424242
```

On first use the skill runs setup (see below). Once configured, it matches the lookup to a catalog service — then, in `certaindata_flow`, it completes the paid call and returns the result with the on-chain settlement details; in `catalog_only`, it hands back the request blueprint and payment terms for you to pay yourself.

**Updating:**

Run `npx skills update` to upgrade to the latest version. Your local `skill-config.json` (mode, API-key reference, environment preference) is preserved across updates.

---

## Setup

Setup runs a short interview to configure the skill and writes a local `skill-config.json`. It triggers in two ways:

- **First invocation** — if no config exists, the skill runs setup automatically before answering anything
- **Explicit setup** — tell your agent to set up or configure the skill at any time:
  ```
  Set up CertainData
  Configure the CertainData skill
  ```

**Reconfiguring:** To change your mode or any setting, ask your agent to reconfigure:
```
Reconfigure CertainData
```
This clears the existing config and runs the interview again from the start.

### certaindata_flow

You will need a CertainData account. If you do not have one, sign up at [portal.certaindata.ai](https://portal.certaindata.ai) before running setup.

Setup will ask for:
1. The env var name to store your API key under (default: `CERTAINDATA_API_KEY`)
2. The env file path to read it from (default: `~/.env`)
3. Your environment preference — whether to offer sandbox when a data service supports it, or always use production (default: always production)

Once setup writes its config, add your API key to the env file:

```
CERTAINDATA_API_KEY=your-api-key-here
```

Then restart your agent gateway and invoke the skill — it will proceed automatically.

Your spend controls (per-call, daily, and monthly caps), trust-tier preferences, and endpoint allow/deny lists live on your CertainData account and are managed in the [portal](https://portal.certaindata.ai) — not in the skill. They are enforced automatically each time CertainData signs a payment.

### catalog_only

No account or API key required. Setup asks for your environment preference:

- **Ask each time** — when a service has a test endpoint available, the skill will offer you the choice of sandbox or production before each call
- **Always production** — skip the prompt and always use the live endpoint (recommended for production or automated environments)

Once answered, the skill is ready immediately.

---

## Sandbox Mode

Sandbox uses test USDC on Base Sepolia — no real money is spent. It is offered **only for catalog services that support it** (the catalog flags each one); Open Ecosystem endpoints and any URL you supply always use production. When you choose sandbox, the same service URL is used — the skill just routes the call to the test environment. Selecting it requires your confirmation each time.

How it works depends on your mode:

- **`certaindata_flow`** — CertainData funds your Base Sepolia test wallet automatically and signs the test payment for you, just like production. You do nothing extra.
- **`catalog_only`** — the skill hands you the test (Base Sepolia) request and payment terms, and you fund and pay your own test wallet.

---

## Examples

### Data lookups

```
What is the EUR to USD exchange rate?
```
```
Look up BIN 424242
```
```
What are the public holidays in South Africa this year?
```
```
What is the current BTC price in USD?
```

The skill checks the live catalog first. If a matching service exists, it handles the lookup and returns the result with the on-chain settlement details. If no service matches, it searches the public x402 Bazaar before falling back to web search.

### x402 payments

```
Pay this x402 endpoint: https://api.example.com/data
```

In `certaindata_flow` mode the skill calls the endpoint, has CertainData sign the payment from your CertainData account wallet, and completes the call — returning the data and the settlement transaction. Paying endpoints outside the CertainData catalog requires the **Open Ecosystem** preference to be enabled on your account (managed in the portal). In `catalog_only` mode the skill returns the request blueprint and payment terms for you to pay yourself.

---

## What This Skill Is Not

- **Not an agent-controlled wallet** — payments are signed from your own CertainData account wallet, and CertainData applies your spend controls. The agent never holds your private keys.
- **Not a substitute for web search** — it checks paid data first and falls back to the web only when nothing matches.
- **Not authoritative or a live feed** — results are indicative reference data, not a system of record or a real-time stream. FX rates are reference rates (not trading signals); BIN data is indicative metadata (not bank-of-record verification); and every call is a discrete request/response, with no subscription or push model.
- **Not a blanket data-quality guarantee** — CertainData vets **Verified** catalog services for data quality, uptime, and freshness, but does not vouch for **Buyer Approved** or open-ecosystem endpoints. For those, you get whatever the seller returns after payment settles, which may be empty or wrong.
- **Not infallible in sandbox** — some sandbox services accept only curated test inputs, so an out-of-set input can return a useless result even though the test payment settled.

---

## Troubleshooting

**The skill doesn't run / nothing happens.** It may not be loaded. Re-install (`npx skills add certaindata/skills --skill certaindata`) or reload your agent, confirm `SKILL.md` is in your platform's skills directory, then try a quick lookup again.

**It can't find your API key, or the key was rejected (`certaindata_flow`).** Check that your env file contains the key line (e.g. `CERTAINDATA_API_KEY=your-actual-certaindata-api-key-here`) under the variable name and path you set during setup, then restart your agent gateway. Manage or rotate the key in the [portal](https://portal.certaindata.ai).

**A payment was refused.** CertainData applies your spend caps, trust-tier preferences, and allow/deny lists at sign time. When a payment is blocked the skill tells you why — adjust the relevant setting in the [portal](https://portal.certaindata.ai) and retry.

**Still stuck?** Reach support (below).

---

## Support

- **Portal & account management:** [portal.certaindata.ai](https://portal.certaindata.ai)
- **Contact:** [certaindata.ai/contact](https://www.certaindata.ai/contact)
- **Email:** support@certaindata.ai
