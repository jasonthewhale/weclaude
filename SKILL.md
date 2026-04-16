---
name: weclaude
description: "Use this skill when the user wants to get a Claude API key via WeClaude, onboard to WeClaude, top up Claude API credits, check balance, withdraw unused balance, OR contribute their Claude OAuth token as a seller. Trigger on: 'use weclaude skill', 'get a Claude API key', 'get an api key for claude code', 'set up weclaude', 'onboard to weclaude', 'buy Claude credits', 'top up Claude', 'weclaude topup', 'check weclaude balance', 'withdraw weclaude balance', 'claude api with crypto', 'pay for claude api', 'contribute my Claude account', 'become a weclaude seller', 'share my Claude token', 'seller status', 'check seller stats'. Chinese: 获取Claude API密钥, 充值Claude, Claude API密钥, Claude余额, Claude提现, 购买Claude额度, 成为卖家, 贡献Claude账号. Do NOT use for general Claude questions or prompt engineering. Do NOT use for wallet setup — use okx-agentic-wallet. Do NOT use for token swaps — use okx-dex-swap."
metadata:
  author: weclaude
  version: "2.0.0"
---

# WeClaude — Buyer & Seller Operations

WeClaude is a pay-per-use Claude API proxy. **Buyers** pay USDG on X Layer via x402, get an API key, and use it with any Anthropic or OpenAI-compatible SDK. **Sellers** contribute their Claude OAuth tokens to the pool and earn usage stats. Pricing is real token usage. Unused balance can be withdrawn at any time.

## Skill Routing

- Wallet setup / login / token balance → use `okx-agentic-wallet`
- Token swaps → use `okx-dex-swap`
- x402 payments to other services (not WeClaude) → use `okx-x402-payment`

## Pre-flight Checks

> Before the first `onchainos` command this session, read and follow: [_shared/preflight.md](_shared/preflight.md)

## Server

**`https://api.weclaude.cc`** — verify with `curl -s "https://api.weclaude.cc/health"` before proceeding.

---

## Buyer Operations

For step-by-step details, see [references/buyer-operations.md](references/buyer-operations.md).
For x402 payment header assembly, see [references/x402-payment-flow.md](references/x402-payment-flow.md).

### Topup — Get or Recharge an API Key

**Trigger**: user wants a Claude API key, buy credits, top up, or onboard.

**Quick flow:**
1. `POST /v1/buyer/topup[/<amount>]` with `-H "Content-Type: application/json"` → server returns **402** with payment details **in the JSON body**
2. Read the 402 body: `x402Version`, `resource`, `accepted`, `amount`, `network`, `asset`, `payTo`, `maxTimeoutSeconds`, `payment_header_name`, and `instructions`
3. Detect payer address: `onchainos wallet status`
4. **Confirm** amount + addresses with user — **STOP until confirmed**
5. Sign: `onchainos payment x402-pay --accepts '[<accepted_object>]' --from <PAYER_ADDRESS>` (wrap the `accepted` object from step 2 in a JSON array)
6. Extract `signature` and `authorization` from sign response (check both `data.*` and top-level)
7. Build payload using `x402Version`, `resource`, and `accepted` from the 402 body, base64-encode, replay with **both** `Content-Type: application/json` and the payment header — **follow the `instructions` array from the 402 body**
8. Present API key + configure Claude Code (`ANTHROPIC_BASE_URL=https://api.weclaude.cc` + `ANTHROPIC_API_KEY`)

**Important**: The server returns **402 for all payment failures** — missing header, malformed header, and verification failure all produce the same 402 response. Do not try to diagnose failure mode from the status code; instead check that the header is present, correctly structured, and base64-encoded.

Tiers: `$0.10` (default), `$0.50`, `$1.00`, `$5.00`. Same wallet tops up existing balance, returns same key.

For full payload assembly details, see [references/x402-payment-flow.md](references/x402-payment-flow.md).

### Balance — Check Remaining Credits

**Trigger**: user wants to check balance or usage.

- By API key: `curl -s "https://api.weclaude.cc/v1/buyer/balance" -H "Authorization: Bearer <KEY>"`
- By wallet: `curl -s "https://api.weclaude.cc/v1/buyer/balance?payer=<ADDRESS>"` — also returns the `api_key` (use this to recover a lost key)

### Withdraw — Refund Unused Balance

**Trigger**: user wants to withdraw, refund, or get remaining USDG back.

1. **Confirm** with user — this sends funds on-chain. **STOP until confirmed.**
2. `POST /v1/buyer/withdraw` with Bearer token
3. Present the `message` field directly — it's the final user-facing output
4. API key is **kept** after withdrawal — user can top up again later

If refund fails (`status: "refund_failed"`), balance stays intact. Suggest retry later.

---

## Seller Operations

For step-by-step details, see [references/seller-operations.md](references/seller-operations.md).

Each seller wallet maps to exactly one OAuth account (1:1, like buyer wallet → API key).

### Register as Seller — Contribute Claude OAuth Token

**Trigger**: user wants to contribute their Claude account, become a seller, share their token.

1. Detect wallet via `onchainos wallet status`
2. `POST /v1/seller/auth/start {"seller_address": "0x..."}` — get `auth_url` + `state`
3. Tell user to visit `auth_url`, log into Claude, approve, then **paste the callback URL** (localhost:54545 page won't load — expected)
4. **STOP. Wait for user to paste URL.**
5. `POST /v1/seller/auth/complete {"state": "...", "callback_url": "<pasted>"}` — token added to pool

409 = address already registered. Auth session expires in 5 minutes.

### Seller Status — Check Account Stats

**Trigger**: seller wants to check usage stats, request count, tokens served.

`GET /v1/seller/status?address=<SELLER_ADDRESS>` — returns total_requests, total tokens, status.

---

## API Compatibility

| Format | Endpoint |
|---|---|
| Anthropic (native) | `POST /v1/messages` |
| OpenAI-compatible | `POST /v1/chat/completions` |
| Responses API | `POST /v1/responses` |

Point any SDK at `https://api.weclaude.cc` as the base URL. List models: `GET /v1/models`.

## Edge Cases

| Situation | Action |
|---|---|
| Server unreachable | `curl https://api.weclaude.cc/health` — may be temporarily down. |
| HTTP status is not 402 on topup | Show body — x402 middleware may be misconfigured. |
| Repeated 402 on replay (topup) | The server returns 402 for ALL payment failures. Checklist: (1) Is `PAYMENT-SIGNATURE` header present? (2) Is the value base64-encoded JSON (not raw hex/binary)? (3) Does the JSON have `x402Version`, `resource`, `accepted`, and `payload` fields? (4) Is `Content-Type: application/json` set on the replay request? |
| Wallet not logged in | Guide user: `onchainos wallet login` |
| Signing fails | Report error, offer retry or cancel. |
| Zero balance on withdraw | Nothing to withdraw — inform user. |
| Same wallet topups again | Server tops up existing balance, returns same API key. |
| Invalid or lost API key | Look up by wallet: `GET /v1/buyer/balance?payer=0x...` — returns `api_key`. For withdraw, topup first. |
| `_weclaude.warning: "low_balance"` in response | Balance below $1. Proactively top up before next call fails. Follow `topup_tiers` URLs in the warning. |
| Balance $0.000000 after usage | Balance depleted. The 402 response includes `topup_tiers` and `instructions` — follow them to top up. Same wallet, same API key. |
| Insufficient balance 402 on API call | Read the 402 body — it includes `topup_tiers` and `instructions` for the x402 payment flow. No need to reconfigure after topup. |
| Larger topup | Tiers: `/v1/buyer/topup/0.5`, `/v1/buyer/topup/1.0`, `/v1/buyer/topup/5.0`. |
| Seller auth 409 | Address already has an active account — show `account_id`. |
| Seller auth expired | 5-minute window passed. Start new flow. |
| Token exchange fails | Claude OAuth issue — retry. Browser session may have expired. |
| Seller status 404 | Not registered yet — guide through seller registration. |
