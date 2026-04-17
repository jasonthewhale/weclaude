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
curl -s -D /tmp/weclaude-headers.txt -o /tmp/weclaude-body.txt \
  -X POST "https://api.weclaude.cc/v1/buyer/topup[/<amount>]" \
  -H "Content-Type: application/json"
```

Replace `[/<amount>]` with the chosen tier suffix (e.g. `/0.5`) or omit for the $0.10 default.

Read the HTTP status from the response. If it is **not 402**, show the response body and stop.

### Step 2: Decode the challenge

The 402 response body is JSON containing the payment challenge. Save it for later steps:

```bash
cp /tmp/weclaude-body.txt /tmp/weclaude-challenge.json
```

From the body, extract these key fields:

| Field | Purpose | Example |
|---|---|---|
| `x402Version` | Determines payment header name (Step 5) | `2` |
| `resource` | Included in payment replay header | `POST /v1/buyer/topup` |
| `accepted` | Payment option object — pass to `--accepts` wrapped in `[]` | `{"scheme":"exact","network":"eip155:196",...}` |

The `accepted` object contains `network`, `payTo`, `price.amount`, `price.asset`, and `maxTimeoutSeconds`.

Human-readable amount = `accepted.price.amount / 1_000_000` USDG.

### Step 3: Detect payer address

First, check login status:
```bash
onchainos wallet status
```
If `data.loggedIn` is `false`, ask the user to log in (`onchainos wallet login`), then re-check.

Then get the X Layer address:
```bash
onchainos wallet addresses
```
Parse the JSON response and extract `data.xlayer[0].address` — this is `PAYER_ADDRESS`.

> **Important**: `onchainos wallet status` returns login info only — it does NOT contain any address. Always use `onchainos wallet addresses` for the address.
> **Do NOT** pass `--chain` to filter — it is not supported and will error. Parse the full JSON output instead.

If the user explicitly provided a different address, use that instead.

### Step 4: Confirm payment (one-time gate)

Present a brief summary and wait for the user:

> Ready to top up your WeClaude account:
> - **Amount**: `<human-readable>` USDG (e.g. $0.10, $0.50, $1.00, or $5.00)
> - **Network**: X Layer (`eip155:196`)
> - **Pay to**: `<payTo>`
> - **Pay from**: `<PAYER_ADDRESS>` (your default wallet)
>
> Proceed? (yes / no)

**STOP. Do not continue without explicit user confirmation.**

Once confirmed, proceed automatically without further prompts.

### Step 5: Sign and submit the x402 payment

Sign using the `accepted` object from the 402 body, wrapped in an array:
```bash
onchainos payment x402-pay \
  --accepts "$(python3 -c "import json; body=json.load(open('/tmp/weclaude-challenge.json')); print(json.dumps([body['accepted']]))")" \
  --from <PAYER_ADDRESS>
```

Extract `signature` and `authorization` from the response (check both `data.*` and top-level keys).

If signing fails: report the error and ask the user whether to retry or cancel. Do not silently give up.

Build the payment header and replay the request per [x402-payment-flow.md](x402-payment-flow.md). **Critical**: the replay request MUST include `Content-Type: application/json` — without it the payment middleware may not process the header correctly.

### Step 6: Present result and configure Claude Code

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
> **Configure Claude Code** — run these commands (or add to `~/.zshrc` / `~/.bashrc`):
> ```bash
> export ANTHROPIC_BASE_URL=https://api.weclaude.cc
> export ANTHROPIC_API_KEY=sk-x402-...
> ```
>
> **Test it:**
> ```bash
> curl https://api.weclaude.cc/v1/messages \
>   -H "Authorization: Bearer sk-x402-..." \
>   -H "Content-Type: application/json" \
>   -d '{"model":"claude-sonnet-4-6","max_tokens":256,"messages":[{"role":"user","content":"Hello!"}]}'
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
