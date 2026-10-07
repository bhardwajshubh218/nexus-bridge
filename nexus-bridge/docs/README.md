# Nexus Bridge: One Merchant, One Identity

The same merchant exists as four unrelated records in Passport, MXM, CPX and ACH. There's no common key between them, so Support and Finance match records by hand whenever someone asks about a payment or a settlement.

Nexus Bridge is a small identity layer between the Shared Entity System (SES) and the Unified Transaction API. On every transaction it does four things:

1. **Identity.** Resolves any legacy ID to one canonical SES entity.
2. **Authorization.** Checks that the caller's tenant is allowed to act on that entity.
3. **Entitlements.** Confirms the entity is ACTIVE for the payment rail being used (ACH, Card or Payout).
4. **Routing.** Translates the canonical ID into the right provider ID underneath: an MXM merchant ID or a Passport / Banking PaaS account.

If any step fails, the transaction stops before it reaches the Unified Transaction API. Nothing gets guessed, and nothing gets merged automatically.

- Prototype: `prototype/nexus-bridge-prototype.html`. There's also a dark version in the same folder.
- Hosted copy: https://claude.ai/artifact/PyX2WiGsCPmZ3ey4HnYnYB (only works for people it has been shared with)
- Slides: `presentation/nexus-bridge-deck.html`

All data in the prototype is made up. It doesn't connect to the real SES, MXM, CPX, Passport, ACH, Kafka, Redis or the Unified Transaction API.

---

## What's in the folder

```
nexus-bridge-submission/
├── README.md                              this file
├── prototype/
│   ├── nexus-bridge-prototype.html        the prototype, light theme (all code is in this file)
│   └── nexus-bridge-prototype-dark.html   same prototype, dark theme
├── presentation/
│   └── nexus-bridge-deck.html             6-slide deck
├── docs/
│   ├── impact.md                          short write-up of the business impact
│   ├── demo-script.md                     what to click and say during the demo
│   ├── prototype-specification.md         original screen spec
│   └── upgrade-brief.md                   brief for the second version
└── demo/
    └── (screen recording goes here)
```

---

## Prerequisites

You only need a web browser: a recent Chrome, Edge, Firefox or Safari.

There's nothing to install. No Node, no packages, no database and no API keys.

An internet connection is optional. It's only used to load fonts from Google Fonts. Offline, the page falls back to system fonts and everything still works.

Python 3 is also optional. You only need it if you want to serve the files on localhost.

---

## Installation

1. Unzip the package.

   ```bash
   unzip nexus-bridge-submission.zip
   ```

2. Check the prototype is there.

   ```bash
   cd nexus-bridge-submission
   ls prototype/
   ```

There's no build step.

---

## Running it

**Open the file directly.** Double-click `prototype/nexus-bridge-prototype.html`, or run:

```bash
open prototype/nexus-bridge-prototype.html        # macOS
start prototype\nexus-bridge-prototype.html       # Windows
xdg-open prototype/nexus-bridge-prototype.html    # Linux
```

**Serve it on localhost**, if your browser is strict about local files:

```bash
cd prototype
python3 -m http.server 8765
```

Then open http://localhost:8765/nexus-bridge-prototype.html.

**Slides.** Double-click `presentation/nexus-bridge-deck.html`. Move with the arrow keys or the space bar, and press F for full screen.

**Resetting.** Everything is kept in memory. Refreshing the page puts it back to the starting state, and so does the **Reset demo** button in the sidebar or the demo panel.

---

## Demo walkthrough

A **Demo script** panel in the bottom-right corner lists these steps, and each has a **Go** button. There's a longer version, with talking points, in `docs/demo-script.md`.

1. **Resolve a merchant.** Search `MXM-77329`. It resolves to Acme Retail LLC (`SES-8842-ACME`) through exactly one registry mapping. Open the identity graph to see the parent company, linked legacy IDs, sub-accounts and payment instruments. Balances are shown per currency, and the ledger combines MXM, CPX and ACH activity. The new **Products & provider routing** card shows which rails are active and which provider IDs sit under the canonical ID.
2. **Stamp a transaction.** In the Stamping Simulator, run *Clean resolution*. Both IDs resolve and the tenant is authorized. ACH is active, so it routes to Passport Banking PaaS. The output adds canonical IDs, tenant metadata and provider routing, and leaves every original field alone.
3. **Route a card payment.** Run *Card settlement · Acme Coffee*. This is the main example:

   ```
   MXM-84721
     → SES-10294
     → Acme Coffee LLC
     → Tenant TNT-NA-03 authorized
     → Status ACTIVE
     → Products: ACH ✓  Card ✓  Payout ✓
     → Provider routing: MXM (MXM-84721)
     → Unified Transaction API
     → Transaction proceeds
   ```

4. **See an entitlement block.** Run *Entitlement pending*. Brightline Bakery hasn't finished onboarding, so the payout is stopped at the pre-flight check. The Unified Transaction API is never called.
5. **Review a Tax ID conflict.** In the Exception Queue, open `EX-3021`. The names match but the Tax IDs don't, so approval isn't allowed. Create a new independent entity instead.
6. **Approve a mapping that passes.** Open `EX-3016`. The Tax ID and bank match exactly, so the mapping can be approved.

A few more things to try:
- Run *Cross-tenant access*. A caller from another tenant is refused at authorization.
- Open **SES lifecycle events** under the simulator and publish an event. "Complete Brightline Bakery onboarding" lets the pending payout through. "Place risk hold on Acme Coffee card" blocks its card payments until you release it.
- Open **Technical details** under any decision to see the API calls, the token claims, the transaction legs and the cache lookup.

---

## System architecture

### The flow

```
Legacy merchant ID  (MXM-84721)
      │
      ▼
┌─────────────────────────── Nexus Bridge ───────────────────────────┐
│  1. Resolve identity     registry lookup → SES canonical entity    │
│  2. Authorize tenant     caller token → tenantId / ownerId         │
│                          → validate against SES → allow or deny    │
│  3. Entitlement gate     GET /v1/businesses/:id/locations/:id/     │
│                          services → rail must be ACTIVE            │
│  4. Provider routing     SES ID → MXM merchant ID (card, ACH       │
│                          acquiring) or Passport / Banking PaaS     │
│                          account (payouts, wires, ACH origination) │
└────────────────────────────────────────────────────────────────────┘
      │  stamped request                 │  any step fails
      ▼                                  ▼
Unified Transaction API            blocked, not persisted,
      │                            reason shown, logged
      ▼
Downstream provider (MXM / Passport Banking PaaS)
```

Nexus Bridge sits in front of the existing provider systems. They aren't rebuilt or changed. Stamping only adds fields, so existing consumers keep working.

### Unified Transaction API model

Money movement is modelled as two symmetric legs, `source` and `destination`. Each leg carries a `party` (the caller's own merchant) or a `counterparty`. A party can be given as an SES ID (`tenant_entity_id`), or as an inline entity payload that Nexus Bridge resolves at runtime using the same rules.

```json
{
  "source":      { "counterparty": { "tenant_entity_id": "SES-0100-CNET" } },
  "destination": { "party":        { "tenant_entity_id": "SES-10294" } },
  "rail": "card",
  "provider_ref": { "provider": "MXM", "merchant_id": "MXM-84721" }
}
```

Because every leg ends up with a canonical SES ID, KYC/KYB data, entity deduplication, risk profiles and payment lifecycle tracking can all key on the same identity.

### Events and caching

SES publishes lifecycle events to Kafka:

- `ses.entity.created` and `ses.entity.updated`
- `ses.onboarding.status_changed`
- `ses.entitlement.changed`
- `ses.risk.status_changed`

Nexus Bridge consumes these events and refreshes a Redis cache. The cache holds `nb:map:{legacy_id}` for canonical ID lookups, and `nb:ent:{ses_id}` for tenant, status, entitlements and provider IDs. Transaction-time checks read from the cache so they stay fast, while lifecycle changes keep it current.

### Production view

```
Passport   MXM   CPX   ACH                     (unchanged)
    └───────┴─────┴─────┘
            │ change-data-capture
            ▼
     Registry builder ──► Nexus Registry ◄──── SES ── Kafka events ──┐
            │                   ▲                                    │
            │ can't match       │ lookups                            ▼
            ▼                   │                             Redis cache
     Exception queue ◄── Nexus Bridge (identity · auth · entitlements · routing)
            ▲                   │
            │                   ▼
     Review service       Unified Transaction API ──► MXM / Passport Banking PaaS
            ▲
     Ops console (this prototype)

     Every step writes to an append-only audit log.
```

### Resolution and gate rules

The same rules are applied on every screen.

- One registry mapping resolves the ID. No mapping, or more than one, leaves it unresolved. A canonical ID is never invented or picked arbitrarily.
- A conflicting attribute, such as a different Tax ID, sends the record to the exception queue.
- A similar name or a similarity score never merges or maps anything.
- The caller's tenant must own the party entity. Counterparties can belong to anyone.
- The entity must be ACTIVE, and so must the requested rail. PENDING, UNDER_REVIEW and NOT_ENROLLED are all blocked before orchestration.
- The entity must have a provider ID for the rail. If it doesn't, the transaction is blocked.
- Stamping only adds fields. Existing IDs and fields are never renamed or removed.

### How the prototype is built

It's a single HTML file with inline CSS and JavaScript. There's no framework, no build step and no external scripts, so it runs offline anywhere. The main functions are:

| Function | What it does |
|---|---|
| `seed()` / `extendSeed()` | Sample data: entities, registry, transactions, exceptions, entitlements, provider IDs, SES events |
| `resolveId()` | Identity resolution rules |
| `gateCheck()` | Tenant authorization, entitlement gate and provider resolution |
| `providerFor()` | Picks MXM or Passport Banking PaaS for the rail |
| `processPayload()` / `runSim()` | Runs the full pipeline, then stamps and saves the transaction or blocks it |
| `traceHTML()` / `checksHTML()` | Decision chain, technical details, validation results |
| `routingHTML()` | Products & provider routing card on Entity 360 |
| `eventsHTML()` / `publishEvent()` | SES lifecycle events and cache version |
| `renderQueue()`, `renderSheet()`, `actCreateEntity()`, `actApprove()`, `actReject()`, `actEscalate()` | Exception review |
| `bridgeHTML()` | "Where Nexus Bridge sits" overview |

---

## Sample data

| Record | Why it's there |
|---|---|
| Acme Retail Group Inc (`SES-8800-ACMG`) and its two child entities | Hierarchy, and more than one currency |
| Acme Coffee LLC (`SES-10294`, `MXM-84721`, tenant TNT-NA-03) | Main example: everything ACTIVE, routes to MXM |
| Brightline Bakery Co (`SES-10311`, `MXM-84990`) | Onboarding pending, so it's blocked at the entitlement gate |
| Beta Logistics Inc (`SES-9001-BETA`) | Counterparty for the ACH scenarios |
| Northwind Café LLC (`SES-7310-NWND`, tenant TNT-NA-02) | Payout under review; different tenant |
| `EX-3021` `MXM-99110` | Tax ID conflict |
| `EX-3019` `ACH-61877` | Ambiguous: maps to two entities |
| `EX-3016` `CPX-5521` | Missing mapping that passes every rule and can be approved |
| `EX-3014` `Passport-Z9Q1` | Missing mapping with nothing to match, so it needs a new entity |

Tax IDs and bank account numbers are masked throughout.

---

## What we tested

We ran every scenario on the finished prototype. There were no console errors.

- Legacy IDs from all four systems, and SES IDs, resolve to the right entity. Unknown, conflicting and ambiguous IDs don't resolve.
- *Clean resolution* goes through and routes to Passport Banking PaaS. *Card settlement · Acme Coffee* goes through and routes to MXM `MXM-84721`.
- *Entitlement pending* is blocked at the entitlement check.
- *Cross-tenant access* is blocked at tenant authorization.
- Missing, conflicting and ambiguous IDs are blocked at identity resolution.
- In every run, all original payload fields come through unchanged.
- Blocked transactions are never saved and never reach the Unified Transaction API.
- Completing Brightline's onboarding through an SES event lets its payout through. A risk hold on Acme Coffee blocks its card payments, and releasing the hold allows them again.
- Exception actions (create entity, approve, reject, escalate) update the registry, search and simulator, and publish an SES event.

---

## Limits

- Everything is simulated in the browser: tenant checks, entitlements, Kafka events and the Redis cache. State resets on reload.
- The inline entity payload is shown as an example request in the technical details. None of the scenarios actually resolve one.
- The benchmark tiles on the search page (50k transactions, 12 ms, 94%) are labelled *Simulated* and aren't measurements.
- We used plain CSS instead of the Tailwind CDN mentioned in the original spec, so the file works offline.
- An Activity log screen was added on top of the three required screens, because identity decisions need a record.

---

## Next steps

1. Run a shadow mode first: build the registry from change-data-capture and log what Nexus Bridge would decide, without changing any payloads.
2. Connect the tenant check and entitlement gate to live SES and the services endpoint, with Kafka events refreshing the cache.
3. Turn on stamping and provider routing for ACH, then for card and payouts.
4. Add two-person approval for registry changes, and export the audit log to compliance.

Shadow mode is also where we'd measure real time savings.

---

## Team

| Name | Role |
|---|---|
| | |
| | |

Event and track:
Idea ID:
