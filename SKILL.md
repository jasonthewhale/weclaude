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

1. Send `POST /v1/buyer/topup[/<amount>]` — get 402 challenge
2. Decode `PAYMENT-REQUIRED` header, extract payment params
3. Detect payer address via `onchainos wallet status`
4. **Confirm** amount + addresses with user — **STOP until confirmed**
5. Sign with `onchainos payment x402-pay`, build header, replay request
6. Present API key + configure Claude Code (`ANTHROPIC_BASE_URL` + `ANTHROPIC_API_KEY`)

Tiers: `$0.10` (default), `$0.50`, `$1.00`, `$5.00`. Same wallet tops up existing balance, returns same key.

### Balance — Check Remaining Credits

**Trigger**: user wants to check balance or usage.

- By API key: `curl -s "https://api.weclaude.cc/v1/buyer/balance" -H "Authorization: Bearer <KEY>"`
- By wallet: `curl -s "https://api.weclaude.cc/v1/buyer/balance?payer=<ADDRESS>"`

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
| Wallet not logged in | Guide user: `onchainos wallet login` |
| Signing fails | Report error, offer retry or cancel. |
| Zero balance on withdraw | Nothing to withdraw — inform user. |
| Same wallet topups again | Server tops up existing balance, returns same API key. |
| Invalid API key | For balance, try `?payer=0x...` instead. For withdraw, topup first. |
| Balance $0.000000 after usage | Large response consumed all balance (clamped at zero). Top up again. |
| Larger topup | Tiers: `/v1/buyer/topup/0.5`, `/v1/buyer/topup/1.0`, `/v1/buyer/topup/5.0`. |
| Seller auth 409 | Address already has an active account — show `account_id`. |
| Seller auth expired | 5-minute window passed. Start new flow. |
| Token exchange fails | Claude OAuth issue — retry. Browser session may have expired. |
| Seller status 404 | Not registered yet — guide through seller registration. |
