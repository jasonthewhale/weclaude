---
name: weclaude
description: "Use this skill when the user wants to get a Claude API key via WeClaude, onboard to WeClaude, top up Claude API credits, check balance, withdraw unused balance, OR contribute their Claude OAuth token as a seller, check seller earnings, or claim seller earnings. Trigger on: 'use weclaude skill', 'get a Claude API key', 'get an api key for claude code', 'set up weclaude', 'onboard to weclaude', 'buy Claude credits', 'top up Claude', 'weclaude topup', 'check weclaude balance', 'withdraw weclaude balance', 'claude api with crypto', 'pay for claude api', 'contribute my Claude account', 'become a weclaude seller', 'share my Claude token', 'seller status', 'check seller stats', 'check seller earnings', 'seller earn', 'claim seller earnings', 'seller claim', 'withdraw seller earnings'. Chinese: 获取Claude API密钥, 充值Claude, Claude API密钥, Claude余额, Claude提现, 购买Claude额度, 成为卖家, 贡献Claude账号, 查看卖家收益, 提取卖家收益. Do NOT use for general Claude questions or prompt engineering. Do NOT use for wallet setup — use okx-agentic-wallet. Do NOT use for token swaps — use okx-dex-swap."
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

## Getting the Wallet Address

WeClaude uses X Layer (chainIndex 196). To get the user's X Layer address:

**Step 1 — Check login:**
```bash
onchainos wallet status
```
If `data.loggedIn` is `false`, ask the user to log in (`onchainos wallet login`), then re-check.

**Step 2 — Get the X Layer address:**
```bash
onchainos wallet addresses
```
Parse the JSON output and extract `data.xlayer[0].address`. This is the payer/seller address for all WeClaude operations.

> **Do NOT** use `onchainos wallet status` to get the address — it only returns login info, not addresses.
> **Do NOT** pass `--chain` to `onchainos wallet addresses` — chain filtering is not supported and will error.

---

## Buyer Operations

For step-by-step details, see [references/buyer-operations.md](references/buyer-operations.md).
For x402 payment header assembly, see [references/x402-payment-flow.md](references/x402-payment-flow.md).

### Topup — Get or Recharge an API Key

**Trigger**: user wants a Claude API key, buy credits, top up, or onboard.

**Quick flow:**
1. `POST /v1/buyer/topup[/<amount>]` with `-H "Content-Type: application/json"` → server returns **402** with payment details **in the JSON body**
2. Read the 402 body: `x402Version`, `resource`, `accepted`, `amount`, `network`, `asset`, `payTo`, `maxTimeoutSeconds`, `payment_header_name`, and `instructions`
3. Detect payer address — see "Getting the Wallet Address" below
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

1. Get X Layer address via `onchainos wallet addresses` → `data.xlayer[0].address` (see "Getting the Wallet Address")
2. `POST /v1/seller/auth/start {"seller_address": "0x..."}` — get `auth_url` + `state`
3. Tell user to visit `auth_url`, log into Claude, approve, then **paste the callback URL** (localhost:54545 page won't load — expected)
4. **STOP. Wait for user to paste URL.**
5. `POST /v1/seller/auth/complete {"state": "...", "callback_url": "<pasted>"}` — token added to pool

409 = address already registered. Auth session expires in 5 minutes.

### Seller Status — Check Account Stats

**Trigger**: seller wants to check usage stats, request count, tokens served.

`GET /v1/seller/status?address=<SELLER_ADDRESS>` — returns total_requests, total tokens, earned_usd, status.

### Seller Earnings — Check Earnings Breakdown

**Trigger**: seller wants to check earnings, how much they've earned, claimable balance.

1. Get X Layer address via `onchainos wallet addresses` → `data.xlayer[0].address`
2. `GET /v1/seller/earn?address=<SELLER_ADDRESS>` — returns earned, claimed, claimable amounts
3. Present the earnings breakdown to user

### Seller Claim — Withdraw Earnings as USDG

**Trigger**: seller wants to claim earnings, withdraw seller earnings, get paid.

1. Get X Layer address via `onchainos wallet addresses` → `data.xlayer[0].address`
2. Check earnings first: `GET /v1/seller/earn?address=<SELLER_ADDRESS>`
3. If `claimable_usd` > 0, **confirm** with user — this sends funds on-chain. **STOP until confirmed.**
4. `POST /v1/seller/claim {"seller_address": "<ADDRESS>"}` — sends USDG to seller wallet
5. Present the `message` field — it's the final user-facing output

Minimum claim: $0.01. If claim fails (`status: "payout_failed"`), earnings stay intact. Suggest retry later.

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
| `X-WeClaude-Warning: low_balance` response header | Balance below $1. Present on **every** API response (streaming and non-streaming). Proactively warn the user and suggest topping up. |
| `X-WeClaude-Balance` response header | Current balance as a decimal string (e.g. `0.043210`). Present on every gated API response. Use for client-side balance tracking. |
| `_weclaude.warning: "low_balance"` in response body | Same as above but only in non-streaming JSON responses. Includes `topup_tiers` URLs. |
| Balance $0.000000 after usage | Balance depleted. The 402 response includes `topup_tiers` and step-by-step `instructions` — follow them to top up. Same wallet, same API key. |
| Insufficient balance 402 on API call | Read the 402 body — it includes `topup_tiers` and `instructions` for the x402 payment flow. No need to reconfigure after topup. |
| Larger topup | Tiers: `/v1/buyer/topup/0.5`, `/v1/buyer/topup/1.0`, `/v1/buyer/topup/5.0`. |
| Seller auth 409 | Address already has an active account — show `account_id`. |
| Seller auth expired | 5-minute window passed. Start new flow. |
| Token exchange fails | Claude OAuth issue — retry. Browser session may have expired. |
| Seller status 404 | Not registered yet — guide through seller registration. |
| Seller claim below minimum | Less than $0.01 claimable — inform user and show current earnings. |
| Seller claim payout fails | Earnings unchanged. Suggest retry later. |
| Seller earn 404 | Not registered — guide through seller registration. |
