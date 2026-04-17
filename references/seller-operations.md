# Seller Operations — Detailed Steps

Sellers contribute their Claude OAuth tokens to the WeClaude pool. Each seller wallet address maps to exactly one OAuth account (1:1, like buyer address -> API key).

## Operation 4: Register as Seller — Contribute Claude OAuth Token

**When to use**: User wants to contribute their Claude account to the WeClaude pool, become a seller, or share their Claude token.

This is a two-step flow: start the OAuth process, then complete it after the user authenticates in their browser.

### Step 1: Detect seller address

```bash
onchainos wallet status
```

Extract the active account `address` field. This is `SELLER_ADDRESS`.

### Step 2: Start the OAuth flow

```bash
curl -s -X POST "https://api.weclaude.cc/v1/seller/auth/start" \
  -H "Content-Type: application/json" \
  -d '{"seller_address": "<SELLER_ADDRESS>"}'
```

Response:
```json
{
  "auth_url": "https://claude.ai/oauth/authorize?...",
  "state": "abc123...",
  "expires_in": 300,
  "instructions": ["1. Visit auth_url...", "..."]
}
```

If the server returns 409, this address already has an active OAuth account.

### Step 3: Present the auth URL to the user

> To contribute your Claude account to WeClaude:
>
> 1. **Open this URL in your browser**: `<auth_url>`
> 2. Log into Claude and approve access
> 3. After the redirect, your browser will show a page that **can't load** (localhost:54545) — **this is expected**
> 4. **Copy the full URL** from your browser's address bar and paste it here

**STOP. Wait for the user to paste the callback URL.**

### Step 4: Complete the OAuth flow

Extract the callback URL the user pasted. Send it to the server:

```bash
curl -s -X POST "https://api.weclaude.cc/v1/seller/auth/complete" \
  -H "Content-Type: application/json" \
  -d '{"state": "<state from step 2>", "callback_url": "<URL pasted by user>"}'
```

Response:
```json
{
  "status": "ok",
  "account_id": "user@example.com",
  "seller_address": "0x...",
  "message": "OAuth token added to pool. Your account is now serving requests."
}
```

Present:

> Your Claude account has been added to the WeClaude pool!
>
> - **Account**: `<account_id>`
> - **Seller address**: `<seller_address>`
>
> Your account will now serve buyer requests. Check your stats anytime with *"check my seller status"*.

---

## Operation 5: Seller Status — Check Account Stats

**When to use**: Seller wants to check their account's usage stats, request count, or token consumption.

### Step 1: Detect seller address

```bash
onchainos wallet status
```

### Step 2: Query seller status

```bash
curl -s "https://api.weclaude.cc/v1/seller/status?address=<SELLER_ADDRESS>"
```

Response:
```json
{
  "account_id": "user@example.com",
  "seller_address": "0x...",
  "source": "seller",
  "status": "active",
  "total_requests": 142,
  "total_input_tokens": 523400,
  "total_output_tokens": 187200,
  "total_cache_creation_tokens": 12000,
  "total_cache_read_tokens": 45000,
  "earned_usd": 0.0385,
  "claimed_usd": 0,
  "created_at": "2026-04-16 02:37:58"
}
```

Present:

> Your WeClaude seller stats:
> - **Account**: `<account_id>`
> - **Status**: `<status>`
> - **Earned**: $`<earned_usd>` (claimed: $`<claimed_usd>`)
> - **Total requests served**: `<total_requests>`
> - **Total tokens**: `<total_input_tokens + total_output_tokens>` (in: `<total_input_tokens>`, out: `<total_output_tokens>`)
> - **Member since**: `<created_at>`

---

## Operation 6: Seller Earnings — Check Earnings Breakdown

**When to use**: Seller wants to check how much they've earned, what's been claimed, and what's claimable.

### Step 1: Detect seller address

```bash
onchainos wallet status
```

### Step 2: Query seller earnings

```bash
curl -s "https://api.weclaude.cc/v1/seller/earn?address=<SELLER_ADDRESS>"
```

Response:
```json
{
  "account_id": "user@example.com",
  "seller_address": "0x...",
  "source": "seller",
  "status": "active",
  "earned_usd": 1.234,
  "claimed_usd": 0.5,
  "claimable_usd": 0.734,
  "total_requests": 542,
  "created_at": "2026-04-16 02:37:58"
}
```

Present:

> Your WeClaude earnings:
> - **Account**: `<account_id>`
> - **Total earned**: $`<earned_usd>`
> - **Already claimed**: $`<claimed_usd>`
> - **Claimable now**: $`<claimable_usd>`
> - **Total requests served**: `<total_requests>`
>
> To withdraw your earnings as USDG, say *"claim my seller earnings"*.

---

## Operation 7: Seller Claim — Withdraw Earnings as USDG

**When to use**: Seller wants to claim accumulated earnings, withdraw seller earnings, get paid out.

### Step 1: Detect seller address

```bash
onchainos wallet status
```

### Step 2: Check claimable amount

```bash
curl -s "https://api.weclaude.cc/v1/seller/earn?address=<SELLER_ADDRESS>"
```

If `claimable_usd` is 0 or below $0.01, inform the user there's nothing to claim yet.

### Step 3: Confirm with user

> You have **$`<claimable_usd>`** in claimable earnings. This will send USDG to your wallet `<SELLER_ADDRESS>`.
>
> **Proceed with claim?**

**STOP. Wait for user confirmation.**

### Step 4: Submit the claim

```bash
curl -s -X POST "https://api.weclaude.cc/v1/seller/claim" \
  -H "Content-Type: application/json" \
  -d '{"seller_address": "<SELLER_ADDRESS>"}'
```

Success response:
```json
{
  "status": "claimed",
  "account_id": "user@example.com",
  "claimed_amount": "$0.734000",
  "total_earned": "$1.234000",
  "total_claimed": "$1.234000",
  "payout_tx": "...",
  "message": "$0.734000 USDG sent to 0x..."
}
```

Present the `message` field directly — it's the final user-facing output.

If `status: "payout_failed"`, earnings are unchanged. Suggest retry later.
If `status: "nothing_to_claim"`, inform user the minimum claim is $0.01.
