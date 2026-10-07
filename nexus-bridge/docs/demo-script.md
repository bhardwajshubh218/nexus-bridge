# Demo script (about 6 minutes)

Use this for the live demo . Open `prototype/nexus-bridge-prototype.html` first, and press **Reset demo** if you've already clicked around.

---

**0:00 · The problem (slides 1 and 2)**

"The same merchant is four different records in Passport, MXM, CPX and ACH, and they share no common key. Today, answering one question about a payout means checking all four systems and matching records by hand."

**0:40 · Resolve a merchant**

Search `MXM-77329`.

"This is an MXM ID. The registry has exactly one mapping for it, so it resolves to Acme Retail LLC, `SES-8842-ACME`."

Expand the identity graph, then scroll to Financial overview, Products & provider routing, and the ledger.

"One ID ties together the parent company, the legacy IDs, sub-accounts and bank accounts. Currencies stay separate. This card shows which rails are active, and the provider IDs under the one canonical ID: an MXM merchant ID for card and ACH acquiring, and a Passport banking account for payouts."

**1:40 · Stamp a transaction**

Go to the Stamping Simulator. Keep *Clean resolution* selected and click **Simulate Payload Injection**.

"Both IDs resolve, the caller's tenant owns the merchant, ACH is active, and it routes to Passport Banking PaaS. The green lines are the canonical IDs we added. Every original field is unchanged."

**2:20 · Route a card payment (the main example)**

Pick *Card settlement · Acme Coffee* and run it. Point to the Nexus Bridge decision panel.

"`MXM-84721` resolves to `SES-10294`, Acme Coffee LLC. The tenant is authorized, the status is ACTIVE, and ACH, Card and Payout are all enabled. Nexus Bridge translates the canonical ID to MXM merchant ID `MXM-84721` and sends the transaction to the Unified Transaction API."

Open **Technical details** if there's time. It shows the token claims, the services endpoint, the source and destination legs, and the cache lookup.

**3:10 · Block it before it fails later**

Run *Entitlement pending*.

"Brightline Bakery hasn't finished onboarding. Nexus Bridge stops the payout at the pre-flight check, so the Unified Transaction API is never called."

Optional: run *Cross-tenant access*. "A caller from another tenant is refused at authorization."

**3:50 · Events keep the cache current**

Open **SES lifecycle events** and click **Complete Brightline Bakery onboarding**. Run *Entitlement pending* again.

"SES publishes the change to Kafka and Nexus Bridge refreshes its cache. The same payout now goes through."

**4:30 · Review an exception**

Open the Exception Queue and click **Review** on `EX-3021`.

"The names are nearly identical, but the Tax IDs end in 4421 and 4422. The rules check fails, so approval isn't allowed." Click **Create New Independent Entity**. "It gets its own canonical ID, and nothing is merged."

**5:20 · Close (slides 4 to 6)**

"Nexus Bridge isn't just a lookup table. It handles identity, authorization, entitlements and routing, all in front of the systems we already run, and none of them need rebuilding. The next step is a shadow-mode pilot on ACH traffic, where we'd measure the real time saved."
