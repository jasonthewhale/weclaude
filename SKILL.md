---
name: weclaude
description: "Use this skill when the user wants to get a Claude API key via WeClaude, onboard to WeClaude, top up Claude API credits, check balance, or withdraw unused balance. Trigger on: 'use weclaude skill', 'get a Claude API key', 'get an api key for claude code', 'set up weclaude', 'onboard to weclaude', 'buy Claude credits', 'top up Claude', 'weclaude topup', 'check weclaude balance', 'withdraw weclaude balance', 'claude api with crypto', 'pay for claude api'. Chinese: 获取Claude API密钥, 充值Claude, Claude API密钥, Claude余额, Claude提现, 购买Claude额度. Do NOT use for general Claude questions or prompt engineering. Do NOT use for wallet setup — use okx-agentic-wallet. Do NOT use for token swaps — use okx-dex-swap."
metadata:
  author: weclaude
  version: "1.0.0"
---

# WeClaude — Buyer Onboarding

WeClaude is a pay-per-use Claude API proxy. Pay USDG on X Layer via x402, get an API key, and use it with any Anthropic or OpenAI-compatible SDK. Pricing is real token usage. Unused balance can be withdrawn at any time.

For x402 payment header assembly details, see [references/x402-payment-flow.md](references/x402-payment-flow.md).

## Skill Routing

- Wallet setup / login / token balance → use `okx-agentic-wallet`
- Token swaps → use `okx-dex-swap`
- x402 payments to other services (not WeClaude) → use `okx-x402-payment`

## Pre-flight Checks

> Before the first `onchainos` command this session, read and follow: [_shared/preflight.md](_shared/preflight.md)

## Server URL

- If the user provides a server URL (e.g., an ngrok URL like `https://abc123.ngrok-free.app`), use it for the session.
- Otherwise, **default to `http://127.0.0.1:4021`** — the local WeClaude server.
- Verify the server is reachable before proceeding:
  ```bash
  curl -s "<SERVER_URL>/health"
  ```
  If this fails, inform the user and suggest starting the server:
  ```bash
  cd ~/Documents/OKX-Hackthon/weclaude-402 && bun run start
  ```

---

## Operation 1: Topup — Get or Recharge an API Key

**When to use**: User wants to get a Claude API key, buy Claude credits, top up, or onboard to WeClaude.

This operation is **fully automatic** after a single confirmation. The agent handles the entire x402 payment flow.

### Step 1: Get the 402 payment challenge

```bash
curl -s -D /tmp/weclaude-headers.txt -o /tmp/weclaude-body.txt \
  -X POST "<SERVER_URL>/v1/topup" \
  -H "Content-Type: application/json"
```

Read the HTTP status from the response. If it is **not 402**, show the response body and stop.

### Step 2: Decode the challenge

Extract and decode the `PAYMENT-REQUIRED` header:

```bash
PAYMENT_REQUIRED=$(grep -i "^payment-required:" /tmp/weclaude-headers.txt \
  | cut -d' ' -f2- | tr -d '\r\n')
echo "$PAYMENT_REQUIRED" | base64 -d > /tmp/weclaude-challenge.json
```

Parse the JSON. From `decoded.accepts[0]` extract:

| Field | CLI param | Example |
|---|---|---|
| `.network` | `--network` | `eip155:196` |
| `.price.amount` | `--amount` | `100000` (6-decimal USDG) |
| `.payTo` | `--pay-to` | seller address |
| `.price.asset` | `--asset` | USDG contract address |
| `.maxTimeoutSeconds` | `--max-timeout-seconds` | `600` |

Human-readable amount = `price.amount ÷ 1_000_000` USDG.

Also capture `decoded.x402Version` and `decoded.resource` for Step 5.

### Step 3: Detect payer address

```bash
onchainos wallet status
```

- **Logged in** → extract the active account `address` field. This is `PAYER_ADDRESS`.
- **Not logged in** → ask the user to log in (`onchainos wallet login`), then re-run status.
- If the user explicitly provided a different address, use that instead.

### Step 4: Confirm payment (one-time gate)

Present a brief summary and wait for the user:

> Ready to top up your WeClaude account:
> - **Amount**: `<human-readable>` USDG
> - **Network**: X Layer (`eip155:196`)
> - **Pay to**: `<payTo>`
> - **Pay from**: `<PAYER_ADDRESS>` (your default wallet)
>
> Proceed? (yes / no)

**STOP. Do not continue without explicit user confirmation.**

Once confirmed, proceed automatically without further prompts.

### Step 5: Sign and submit the x402 payment

Sign:
```bash
onchainos payment x402-pay \
  --network <network> \
  --amount <amount> \
  --pay-to <payTo> \
  --asset <asset> \
  --max-timeout-seconds <maxTimeoutSeconds> \
  --from <PAYER_ADDRESS>
```

Extract `signature` and `authorization` from the response (check both `data.*` and top-level keys).

If signing fails: report the error and ask the user whether to retry or cancel. Do not silently give up.

Build the payment header and replay the request per [references/x402-payment-flow.md](references/x402-payment-flow.md).

### Step 6: Present result and configure Claude Code

On success the server returns:
```json
{
  "api_key": "sk-x402-...",
  "balance": "$0.10",
  "pricing": "real token usage — varies by model",
  "close_url": "/v1/close",
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
> export ANTHROPIC_BASE_URL=<SERVER_URL>
> export ANTHROPIC_API_KEY=sk-x402-...
> ```
>
> **Test it:**
> ```bash
> curl <SERVER_URL>/v1/messages \
>   -H "Authorization: Bearer sk-x402-..." \
>   -H "Content-Type: application/json" \
>   -d '{"model":"claude-sonnet-4-6","max_tokens":256,"messages":[{"role":"user","content":"Hello!"}]}'
> ```
>
> To check balance: *"check my weclaude balance"*
> To withdraw unused credits: *"withdraw my weclaude balance"*

**Note**: If the same wallet has topped up before, the server adds to the existing balance and returns the same API key.

---

## Operation 2: Balance — Check Remaining Credits

**When to use**: User wants to check their remaining balance or usage.

Requires the user's API key. Ask if not known.

```bash
curl -s "<SERVER_URL>/v1/balance" \
  -H "Authorization: Bearer <API_KEY>"
```

Response:
```json
{ "balance": "$0.094500", "used": "$0.005500", "topup": "$0.10" }
```

Present:

> Your WeClaude balance:
> - **Remaining**: $X.XXXXXX
> - **Used**: $X.XXXXXX
> - **Per topup**: $X.XX

---

## Operation 3: Close — Withdraw Unused Balance

**When to use**: User wants a refund or to close their session.

Requires the user's API key. Ask if not known.

**Always confirm before proceeding** — this sends funds on-chain:

> You're about to withdraw your remaining WeClaude balance back to your wallet. Your API key stays valid for future topups. Proceed? (yes / no)

**STOP. Wait for confirmation.**

```bash
curl -s -X POST "<SERVER_URL>/v1/close" \
  -H "Authorization: Bearer <API_KEY>"
```

Response:
```json
{
  "status": "withdrawn",
  "used": "$0.005500",
  "withdrawn": "$0.094500",
  "message": "Withdrew $0.094500 to 0x... Total used: $0.005500."
}
```

After withdrawal, suggest checking wallet balance with `okx-agentic-wallet`.

---

## Supported Models

All Anthropic models are supported. List them:
```bash
curl -s "<SERVER_URL>/v1/models"
```

## API Compatibility

| Format | Endpoint |
|---|---|
| Anthropic (native) | `POST /v1/messages` |
| OpenAI-compatible | `POST /v1/chat/completions` |
| Responses API | `POST /v1/responses` |

Point any Anthropic or OpenAI SDK at `<SERVER_URL>` by setting its base URL.

## Edge Cases

| Situation | Action |
|---|---|
| Server unreachable | Run `curl <SERVER_URL>/health`. Start server if needed. |
| HTTP status is not 402 | Show body — x402 middleware may be misconfigured. |
| No `PAYMENT-REQUIRED` header in 402 | Show raw response and stop. |
| Wallet not logged in | Guide user: `onchainos wallet login` |
| Signing fails | Report error, offer retry or cancel. |
| Zero balance on close | Nothing to withdraw — inform user. |
| Same wallet topups again | Server tops up existing balance, returns same API key. |
| User specifies a payer address | Use it in `--from` instead of the auto-detected default. |
| Invalid API key on balance/close | Key may be wrong. Suggest topup to get a valid key. |
