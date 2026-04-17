# Buyer Operations — Detailed Steps

## Operation 1: Topup — Get or Recharge an API Key

**When to use**: User wants to get a Claude API key, buy Claude credits, top up, or onboard to WeClaude.

This operation is **fully automatic** after a single confirmation. The agent handles the entire x402 payment flow.

### Topup tiers

| Endpoint | Amount |
|---|---|
| `POST /v1/buyer/topup` | **$0.10** (default) |
| `POST /v1/buyer/topup/0.5` | $0.50 |
| `POST /v1/buyer/topup/1.0` | $1.00 |
| `POST /v1/buyer/topup/5.0` | $5.00 |

If the user specifies an amount (e.g. "top up $1"), use the matching endpoint. Otherwise default to `POST /v1/buyer/topup` ($0.10).

### Step 1: Get the 402 payment challenge

```bash
curl -s -o /tmp/weclaude-challenge.json -w "%{http_code}" \
  -X POST "https://api.weclaude.cc/v1/buyer/topup[/<amount>]" \
  -H "Content-Type: application/json"
```

Replace `[/<amount>]` with the chosen tier suffix (e.g. `/0.5`) or omit for the $0.10 default.

Read the HTTP status from the response. If it is **not 402**, show the response body and stop.

The 402 body contains the payment details and the exact `onchainos` command to run. Key fields:

| Field | Purpose |
|---|---|
| `accepted.network` | CAIP-2 network (e.g. `eip155:196`) |
| `accepted.amount` | Atomic USDG amount (e.g. `100000` = $0.10) |
| `accepted.payTo` | Recipient address |
| `accepted.asset` | Token contract |
| `accepted.maxTimeoutSeconds` | Signature timeout |
| `instructions` | Step-by-step guide (follow these) |

### Step 2: Detect payer address and confirm

Get the X Layer address:
```bash
onchainos wallet addresses
```
Parse the JSON response and extract `data.xlayer[0].address` — this is `PAYER_ADDRESS`.

Present a brief summary and wait for the user:

> Ready to top up your WeClaude account:
> - **Amount**: `<human-readable>` USDG (e.g. $0.10, $0.50, $1.00, or $5.00)
> - **Network**: X Layer (`eip155:196`)
> - **Pay from**: `<PAYER_ADDRESS>` (your default wallet)
>
> Proceed? (yes / no)

**STOP. Do not continue without explicit user confirmation.**

### Step 3: Sign the payment

Use the values from the 402 challenge body:

```bash
onchainos payment x402-pay \
  --network <accepted.network> \
  --amount <accepted.amount> \
  --pay-to <accepted.payTo> \
  --asset <accepted.asset> \
  --max-timeout-seconds <accepted.maxTimeoutSeconds> \
  --from <PAYER_ADDRESS>
```

Extract `signature` and `authorization` from the response (check both `data.*` and top-level keys).

If signing fails: report the error and ask the user whether to retry or cancel.

### Step 4: Submit the payment

POST the raw sign result back to the same endpoint:

```bash
curl -s -X POST "https://api.weclaude.cc/v1/buyer/topup[/<amount>]" \
  -H "Content-Type: application/json" \
  -d '{"signature":"<SIGNATURE>","authorization":<AUTHORIZATION_OBJECT>}'
```

The server assembles the x402 payment header internally — no base64 encoding or header construction needed.

On success the server returns:
```json
{
  "api_key": "sk-x402-...",
  "balance": "$0.10",
  "created": true,
  "pricing": "real token usage — varies by model",
  "withdraw_url": "/v1/buyer/withdraw",
  "usage": "Authorization: Bearer sk-x402-..."
}
```

Present to the user:

> Your WeClaude API key is ready!
>
> - **API Key**: `sk-x402-...`
> - **Balance**: `$0.10`
> - **Pricing**: pay-per-use (varies by model and token count)
>
> **Quick start** — first logout your current Claude Code session:
> ```bash
> claude /logout
> ```
> Then start Claude Code with WeClaude:
> ```bash
> ANTHROPIC_BASE_URL=https://api.weclaude.cc ANTHROPIC_API_KEY=sk-x402-... claude --dangerously-skip-permissions
> ```
>
> Or add to `~/.zshrc` / `~/.bashrc` for persistent use:
> ```bash
> export ANTHROPIC_BASE_URL=https://api.weclaude.cc
> export ANTHROPIC_API_KEY=sk-x402-...
> ```
>
> To check balance: *"check my weclaude balance"*
> To withdraw unused credits: *"withdraw my weclaude balance"*

**Note**: If the same wallet has topped up before, the server adds to the existing balance and returns the same API key.

---

## Low-Balance Warning Headers

Every gated API response (`/v1/messages`, `/v1/chat/completions`, `/v1/responses`) includes balance headers — these work for both streaming and non-streaming:

| Header | When | Example |
|---|---|---|
| `X-WeClaude-Balance` | Always | `0.043210` |
| `X-WeClaude-Warning` | Balance < $1.00 | `low_balance` |
| `X-WeClaude-Message` | Balance < $1.00 | `Balance is low ($0.04). Top up to avoid interruption.` |

If you detect `X-WeClaude-Warning: low_balance` in a response, proactively inform the user:

> Your WeClaude balance is running low ($X.XX). Top up to avoid interruption:
> - `$0.10`: `/v1/buyer/topup`
> - `$0.50`: `/v1/buyer/topup/0.5`
> - `$1.00`: `/v1/buyer/topup/1.0`
> - `$5.00`: `/v1/buyer/topup/5.0`

---

## Operation 2: Balance — Check Remaining Credits

**When to use**: User wants to check their remaining balance or usage.

Two ways to look up balance — try in this order:

1. **By API key** (if known):
```bash
curl -s "https://api.weclaude.cc/v1/buyer/balance" \
  -H "Authorization: Bearer <API_KEY>"
```

2. **By payer address** (if API key is not known — e.g. user lost the key but knows their wallet):
```bash
curl -s "https://api.weclaude.cc/v1/buyer/balance?payer=<PAYER_ADDRESS>"
```

Each payer address maps to exactly one API key, so both methods return the same account.

**By-wallet lookup also returns the `api_key`** — use this to recover a lost key.

Response (by wallet):
```json
{ "api_key": "sk-x402-...", "balance": "$0.094500", "used": "$0.005500", "payer": "0x..." }
```

Response (by API key):
```json
{ "balance": "$0.094500", "used": "$0.005500", "payer": "0x..." }
```

Present:

> Your WeClaude balance:
> - **API Key**: `sk-x402-...` *(only if recovered via wallet lookup)*
> - **Remaining**: $X.XXXXXX
> - **Used**: $X.XXXXXX
> - **Wallet**: 0x...

---

## Operation 3: Withdraw — Refund Unused Balance

**When to use**: User wants to withdraw unused balance back to their wallet.

The API key is **kept** after withdrawal — users can top up again later without getting a new key.

Requires the user's API key. Ask if not known.

**Always confirm before proceeding** — this sends funds on-chain:

> You're about to withdraw your remaining WeClaude balance back to your wallet. Your API key stays valid for future topups. Proceed? (yes / no)

**STOP. Wait for confirmation.**

```bash
curl -s -X POST "https://api.weclaude.cc/v1/buyer/withdraw" \
  -H "Authorization: Bearer <API_KEY>"
```

Response:
```json
{
  "status": "withdrawn",
  "used": "$0.005500",
  "withdrawn": "$0.094500",
  "refund_tx": "0x...",
  "message": "Refund of $0.094500 USDG sent to 0x... Total used: $0.005500. Thank you for using WeClaude!"
}
```

The `message` field is the final user-facing output — present it directly.

If the refund fails, the server keeps the balance intact and returns `status: "refund_failed"`. Inform the user and suggest retrying later.

After withdrawal, suggest checking wallet balance with `okx-agentic-wallet`.
