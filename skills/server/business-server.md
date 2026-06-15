# Business Server Callbacks

The Anchor Platform handles SEP plumbing. Everything domain-specific — KYC validation, off-chain payouts, exchange rates, custody — is your **business server's** job. The platform calls you; you respond; the SEP flow continues.

Authoritative sources, always check current versions:

- Admin guide: https://developers.stellar.org/docs/platforms/anchor-platform/admin-guide
- Reference business server (Kotlin): https://github.com/stellar/java-stellar-anchor-sdk/tree/main/kotlin-reference-server

This file is a map of *what the platform asks for and when*. The exact endpoint shapes and parameter names follow the current API; cross-check the admin guide before wiring.

---

## The three integration surfaces

```
  +--------------------+      callbacks (rates/fees/KYC/custody)       +---------------------+
  |  Anchor Platform   |  ----------------------------------------->   |  Your business      |
  | (SEP server +      |                                                |  server             |
  |  Platform API +    |  <-----------------------------------------    |                     |
  |  Stellar observer) |        Platform API calls (state ops)         |                     |
  +--------------------+                                                +---------------------+
            ^                                                                    |
            |  Kafka events:                                                     |
            |   transaction_created                                              |
            |   transaction_status_changed                                       |
            |   transaction_error                                                |
            +--------------------------------------------------------------------+
```

Three streams, three things to build:

1. **Callbacks (platform → you)** — the platform asks for a rate, a fee, KYC fields, an off-chain payout confirmation. Synchronous HTTP, signed.
2. **Platform API calls (you → platform)** — you read the transaction list, update transaction status, manually move a stuck transaction. Synchronous HTTP, authenticated.
3. **Events (Kafka)** — the platform publishes state changes; your consumer reacts. Asynchronous, at-least-once delivery.

You need all three. Build them as three thin layers around a shared transaction store.

---

## Surface 1 — Callbacks

The platform configures `CALLBACK_API_BASE_URL` to point at your business server. It calls these endpoints during SEP flows:

### Rate and fee callbacks (drives SEP-38, SEP-6, SEP-24, SEP-31)

| Call | When | What you return |
|---|---|---|
| `GET /rate?type=indicative&sell_asset=…&buy_asset=…&sell_amount=…&context=sep24` | A wallet hit `GET /price` or `GET /prices`. | The indicative `price`, `total_price`, fee breakdown. |
| `GET /rate?type=firm&...` | A wallet posted `POST /quote`. | A firm rate with `expires_at`. **Reserve the rate** internally — you must honor it. |
| `GET /unique-address` | A deposit needs a per-transaction deposit destination. | `stellar_address` + optional `memo` your wallet/exchange will route to. |
| `GET /fee` (legacy) | A wallet hit the deprecated `GET /fee`. | Single `fee` number. New integrations should not rely on this. |

Two things that go wrong here:

1. **Firm quotes ARE a commitment.** When you return one, you've taken on FX or liquidity risk until `expires_at`. Source rates from a system that can hedge or hold inventory; don't return a firm rate computed from a spot you can't actually trade at.
2. **`context=sep31` ≠ `context=sep24`.** Same pair, same amount, different fee tier and corridor. Switch on context.

### KYC callbacks (drives SEP-12)

You typically host SEP-12 directly on your business server (the platform proxies through, or `KYC_SERVER` in `stellar.toml` points at you directly). What the platform expects from your `/customer` endpoints is exactly the SEP-12 contract from [sep12-kyc.md](../client/sep12-kyc.md):

- `GET /customer` → return `{ id, status, fields, provided_fields, message }`.
- `PUT /customer` → store fields, run your KYC vendor, return `{ id }`.
- `DELETE /customer/{account}` → erase. Compliance teams care about this.
- `PUT /customer/callback` → store the callback URL keyed by `customer_id`.

The platform also fires you a `pending_customer_info_update` event when the SEP server detects a transaction needs more KYC — that's a hook to nudge the customer (push notification, email) without polling.

### Custody callbacks (drives deposits and withdrawals)

The platform owns the transaction state machine; you own the actual movement of money.

| Call | When | What you do |
|---|---|---|
| `POST /transactions/{id}/onchain-funds-sent` event-trigger or notify | On a deposit: the platform observed the user's on-chain payment land; or on a withdrawal: the user's Stellar payment hit your `withdraw_anchor_account`. | Match to the transaction by memo. Begin off-chain payout (withdrawal) or move the user's deposited Stellar asset to your treasury (deposit). |
| `POST /transactions/{id}/offchain-funds-sent` | You're telling the platform: I just executed the off-chain payout, here are the rails details. | Mark the platform's record so the state machine advances to `completed`. |
| `POST /transactions/{id}/refund` | A flow failed and you refunded. | Submit the on-chain refund payment and notify the platform. |

In practice you don't implement these as inbound endpoints; you make these calls *outbound* on the Platform API (see Surface 2). The exact mechanism — webhook vs Kafka event vs Platform API call — depends on the version and your `events.queue` config. The reference business server has working examples for the current release.

### Callback authentication

The platform signs callback requests it sends to you with `SECRET_CALLBACK_API_AUTH_SECRET`. Verify before acting. Both JWT and HMAC modes are configurable; the admin guide lists the current default.

---

## Surface 2 — The Platform API (you → platform)

Internal HTTP API the platform exposes on port 8085 (default). You authenticate with `SECRET_PLATFORM_API_AUTH_SECRET` (typically as a `Bearer` JWT). Conceptual operations:

| Verb | Path | Use |
|---|---|---|
| `GET` | `/transactions` | List transactions matching filters. Backfill, reconciliation, admin tools. |
| `GET` | `/transactions/{id}` | Inspect one transaction. |
| `PATCH` | `/transactions/{id}` | **Move a transaction forward.** Set status, add `external_transaction_id`, populate `withdraw_memo`/`stellar_account_id`, attach fee details. This is how you progress the state machine. |
| `POST` | `/transactions/{id}/actions/...` | Verb-named actions (refund, dispatch, mark notified). Names vary by version. |

The most-used call is `PATCH /transactions/{id}`. Two examples:

**Withdrawal: tell the platform where the user must send funds.**

When the platform fires `transaction_created` for a SEP-24 withdrawal, you assign an address + memo from your custody pool and PATCH it back:

```json
PATCH /transactions/abc123
{
  "status": "pending_user_transfer_start",
  "withdraw_anchor_account": "GCIBUCGPOHWMMMFPFKDKE2...",
  "memo":                    "1234567890",
  "memo_type":               "id",
  "amount_in":               "50.0000000",
  "amount_out":              "49.5000000",
  "fee_details":             { "total": "0.50", "asset": "stellar:USDC:GA5Z..." }
}
```

Until you PATCH, the wallet sees `status: incomplete` and can't move forward.

**Deposit: tell the platform the off-chain funds arrived and you're paying out on-chain.**

```json
PATCH /transactions/abc123
{
  "status":                 "pending_stellar",
  "amount_in":              "100.00",
  "external_transaction_id":"BANK-REF-998877"
}
```

After your custody layer pays out and Horizon confirms, the platform's Stellar Observer detects it and moves the transaction to `completed`.

---

## Surface 3 — Kafka events

The platform publishes a stream of state-change events. Subscribe and react. The event types in the current release:

| Event | Triggered by | Typical handler |
|---|---|---|
| `transaction_created` | A wallet POSTed `/transactions/deposit/interactive` or `/withdraw/interactive` (SEP-24), `/deposit` or `/withdraw` (SEP-6), or `/transactions` (SEP-31). | Allocate the deposit/withdraw rail (custody address, memo), compute fees, PATCH back. |
| `transaction_status_changed` | Any status transition. | Update your internal mirror; for `pending_user_transfer_start` (withdrawal), the user has now sent Stellar → start the off-chain payout pipeline. |
| `transaction_error` | Catch-all failure. | Alert on-call; the transaction is stuck and needs human eyes. |
| `customer_status_changed` (SEP-12) | KYC status flipped. | Notify the customer; reattempt their pending transactions. |

Events are at-least-once. **Make every handler idempotent** — same event id should not duplicate a payout.

Lag matters. The platform doesn't wait for your handler before responding to wallets; if your consumer lags 60 seconds behind, the wallet sees a SEP-24 transaction stuck at `incomplete` for 60 seconds. Alert on consumer lag.

---

## A typical SEP-24 deposit, end to end across the three surfaces

```
1.  Wallet POSTs /transactions/deposit/interactive  ───────────────►  SEP Server (platform)
2.                                                                     emits event: transaction_created
3.  Your event consumer wakes up                                       ──►  allocate deposit address+memo
                                                                            compute fee
                                                                            PATCH /transactions/{id} with
                                                                              status = pending_user_transfer_start
                                                                              memo, memo_type
                                                                              amount_in, fee_details
4.  Wallet sees pending_user_transfer_start in /transaction polling
5.  User sends fiat to the bank rail you assigned
6.  Your banking integration confirms receipt                          ──►  PATCH /transactions/{id} with
                                                                              status = pending_anchor
                                                                              external_transaction_id
7.  Your custody layer pays out the Stellar asset to the user          ──►  Custody signs+submits payment
8.  Stellar Observer sees the on-chain payment
                                                                       emits event: transaction_status_changed
                                                                       → status = completed
9.  Wallet sees `completed` and shows success
```

Every arrow either runs through the Platform API or the event bus. Don't side-channel anything — every state change goes through the platform so reconciliation works.

---

## Custody options

The platform supports several custody patterns; the admin guide documents the current set.

| Mode | What it means |
|---|---|
| **None (self-custody)** | The platform / your business server signs payouts directly with a Stellar key in env. Fine for testnet, dangerous for mainnet. |
| **Custody server** | The platform ships a separate custody-server component that holds the signing key and exposes a sign API. Better separation of concerns. |
| **Fireblocks** | Fireblocks integration via their API. Their HSM holds the key; the platform requests signatures. The standard production choice. |

**Hot/warm/cold split.** Even with Fireblocks, hold only a working-capital amount in the hot account. Refill from a warm-key signer with a small daily limit; refill that from a cold multisig with human approval. The platform's distribution account is your *hot* account; the rest is upstream of it.

See [production-checklist.md](production-checklist.md) for the full custody section.

---

## Business-server footgun checklist

- **Idempotency keys everywhere.** Every callback and every event handler can fire twice. Key by `transaction_id` + step, not by event id alone — the platform may emit two `transaction_status_changed` for the same status during retries.
- **PATCH status transitions must be valid.** The platform's state machine rejects backward transitions (you can't go `pending_stellar` → `pending_anchor`). Read the admin guide's status diagram; design your business logic to walk it forward only.
- **A firm rate is a commitment.** Don't return a firm quote you can't hedge or hold inventory for. Returning `expires_at: 30 minutes` and then quoting at spot 25 minutes later means you absorb the slippage.
- **Off-chain payouts must be exactly-once.** Two payouts on the same SEP-31 transaction = lawsuit. Use a database-backed payout ledger your banking integration checks before submitting.
- **`context` discrimination on rate callbacks.** SEP-6 / SEP-24 / SEP-31 deserve different fees. The `context` field on the rate callback exists for this.
- **Verify the platform's callback signature.** Anyone who can reach your business server port can forge callbacks without it.
- **The Stellar Observer is the source of truth for on-chain events.** Don't run a parallel observer that might disagree with the platform's view of "did the payment land." If you must, reconcile on a schedule rather than acting on both signals live.
- **Refunds need their own state machine.** They're not just "transaction reversed" — they're a separate Stellar payment back to the original sender or `refund_memo`. Track them as first-class objects.
