---
name: creditclaw
description: >
  Give your AI agent a payment card with spending controls. CreditClaw enables
  secure checkout at any merchant website, merchant login management with 2FA,
  and real-time spending visibility. Use when the user mentions "buy", "purchase",
  "checkout", "pay for", "order", "shop", "add to cart", "credit card", "payment",
  "CreditClaw", "spending", "budget", "wallet", "log in to store", "saved login",
  "merchant account", or "2FA".
homepage: https://creditclaw.com
allowed-tools: WebFetch Bash
metadata:
  openclaw:
    requires:
      env:
        - CREDITCLAW_API_KEY
  version: "2.1.0"
---

# CreditClaw — Financial Enablement for AI Agents

Give your AI agent a payment card with spending controls, merchant auth, and real-time transaction visibility.

**API Base:** `https://creditclaw.com/api/v1`

## Quick Start

### 1. Register Your Agent

```bash
curl -X POST https://creditclaw.com/api/v1/bots/register \
  -H "Content-Type: application/json" \
  -d '{
    "bot_name": "my-agent",
    "owner_email": "you@example.com",
    "platform": "openclaw"
  }'
```

Response:
```json
{
  "bot_id": "bot_abc123",
  "api_key": "r5_live_...",
  "claim_token": "clm_...",
  "claim_url": "https://creditclaw.com/claim/clm_..."
}
```

Save the `api_key` — you'll need it for all API calls. The owner must visit `claim_url` to:
- Link their Stripe payment method
- Set spending limits and approval rules

### 2. Set Your API Key

Set the environment variable:
```bash
export CREDITCLAW_API_KEY="r5_live_..."
```

### 3. Check Your Status

```bash
curl https://creditclaw.com/api/v1/bot/status \
  -H "Authorization: Bearer $CREDITCLAW_API_KEY"
```

Returns your wallet status, spending limits, and approval settings.

---

## Core Capabilities

### Wallet & Spending

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/bot/status` | GET | Wallet status, limits, card availability |
| `/bot/spending` | GET | Current period spending totals |
| `/bot/transactions` | GET | Transaction history |
| `/bot/messages` | GET | Messages from card owner |

### Checkout (Making Purchases)

**Flow:** Request approval → Fill payment form → Confirm result

#### Step 1: Request Checkout Approval

```bash
curl -X POST https://creditclaw.com/api/v1/checkout \
  -H "Authorization: Bearer $CREDITCLAW_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "merchant_name": "Amazon",
    "merchant_url": "https://amazon.com/checkout",
    "item_name": "USB-C Cable 6ft",
    "amount_cents": 1299,
    "category": "electronics"
  }'
```

Response:
```json
{
  "checkout_id": "r5chk_...",
  "approved": true,
  "card_data_encrypted": "hex-encoded-ciphertext",
  "encryption_key": "hex-encoded-aes-key"
}
```

If `approved: false` and `status: "pending_approval"`:
- Tell the user their card owner will be notified
- Poll `GET /checkout/{id}/status` every 30 seconds
- Approval expires in 15 minutes

#### Step 2: Fill Payment Form (Browser-Based)

**The card data is AES-256-GCM encrypted.** Decrypt it in the browser using the Web Crypto API — the decrypted card number never enters the agent's context:

```javascript
// browser-decrypt.js — execute via browser automation tool
(async () => {
  const cipherHex = "ENCRYPTED_DATA_HEX";
  const keyHex = "ENCRYPTION_KEY_HEX";

  const keyBytes = new Uint8Array(keyHex.match(/.{2}/g).map(b => parseInt(b, 16)));
  const cipherBytes = new Uint8Array(cipherHex.match(/.{2}/g).map(b => parseInt(b, 16)));

  const iv = cipherBytes.slice(0, 12);
  const data = cipherBytes.slice(12);

  const cryptoKey = await crypto.subtle.importKey(
    "raw", keyBytes, "AES-GCM", false, ["decrypt"]
  );

  const decrypted = await crypto.subtle.decrypt(
    { name: "AES-GCM", iv }, cryptoKey, data
  );

  const card = JSON.parse(new TextDecoder().decode(decrypted));

  // Fill form fields
  const fields = {
    "CARD_NUMBER_SELECTOR": card.card_number,
    "EXPIRY_SELECTOR": card.expiry,
    "CVV_SELECTOR": card.cvv,
    "NAME_SELECTOR": card.cardholder_name
  };

  for (const [selector, value] of Object.entries(fields)) {
    const el = document.querySelector(selector);
    if (el) {
      el.focus();
      el.value = value;
      el.dispatchEvent(new Event('input', { bubbles: true }));
      el.dispatchEvent(new Event('change', { bubbles: true }));
    }
  }

  return { status: "filled", fields_filled: Object.keys(fields) };
})();
```

**Instructions:**
1. Navigate to the merchant's checkout page
2. Identify the CSS selectors for card number, expiry, CVV, and name fields
3. Substitute the selectors, `ENCRYPTED_DATA_HEX`, and `ENCRYPTION_KEY_HEX` into the script above
4. Execute the script in the browser via your platform's browser automation tool
5. Click the submit/pay button

**Never log, store, or display the decrypted card data.** The decryption happens atomically inside the browser's JavaScript runtime.

#### Step 3: Confirm Checkout

```bash
curl -X POST https://creditclaw.com/api/v1/checkout/{checkout_id}/confirm \
  -H "Authorization: Bearer $CREDITCLAW_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "success",
    "order_number": "ORD-12345"
  }'
```

### Merchant Auth (Saved Logins & 2FA)

Save merchant logins so repeat purchases skip manual form filling.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/bot/credentials` | GET | List saved merchant logins |
| `/bot/credentials` | POST | Save/update a merchant login |
| `/bot/credentials/lookup?domain=amazon.com` | GET | Look up credentials by domain |
| `/bot/credentials/{id}` | DELETE | Delete saved credentials |
| `/bot/credentials/session` | POST | Get encrypted credentials for login |
| `/bot/credentials/session/confirm` | POST | Report login result |
| `/bot/credentials/totp` | POST | Attach TOTP secret for 2FA |

#### Save a Login

```bash
curl -X POST https://creditclaw.com/api/v1/bot/credentials \
  -H "Authorization: Bearer $CREDITCLAW_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "merchant_domain": "amazon.com",
    "merchant_name": "Amazon",
    "username": "user@example.com",
    "encrypted_password": "encrypted-via-client",
    "login_url": "https://www.amazon.com/ap/signin",
    "has_totp": false
  }'
```

#### Login Flow

1. Look up credentials: `GET /bot/credentials/lookup?domain=amazon.com`
2. Start session: `POST /bot/credentials/session` with `credential_id`
3. Navigate to login page, identify form fields
4. Decrypt and fill credentials using the browser-decrypt pattern (same as checkout)
5. Handle 2FA if needed (TOTP codes generated server-side)
6. Confirm: `POST /bot/credentials/session/confirm`

**2FA support:** TOTP only. If SMS, email, or CAPTCHA appears — stop and tell the user.

---

## Spending Controls

The card owner sets these via the CreditClaw dashboard:

| Control | Description |
|---------|-------------|
| **Per-transaction limit** | Max amount per purchase |
| **Daily/weekly/monthly limit** | Spending caps per period |
| **Category blocking** | Block specific merchant categories |
| **Approval mode** | `auto` (within limits), `always` (every purchase), `none` (disabled) |
| **Merchant whitelist** | Only allow specific merchants |

Your agent sees these limits via `GET /bot/status` and should respect them proactively.

---

## Error Handling

| Situation | Action |
|-----------|--------|
| `401 Unauthorized` | API key invalid or bot not claimed |
| `403 Forbidden` | Spending limit exceeded or category blocked |
| Checkout not approved | Tell user; do not proceed |
| CAPTCHA / 3DS / OTP | Stop; tell user to complete manually |
| Payment declined | Confirm as failed; report to user |
| Login fails | Report to user; do not retry more than once |
| Account locked | Stop immediately; tell user |

---

## Platform-Specific Notes

### Browser Automation

Different platforms provide different browser tools:

| Platform | Browser Tool |
|----------|-------------|
| Claude Code/Cowork | `mcp__playwright__browser_evaluate` |
| OpenClaw / Moltbook | Built-in browser tool or Playwright |
| Zo Computer | Built-in browser tool |
| Manus | Built-in computer use |

Use whichever browser automation tool your platform provides to execute the decrypt-and-fill script.

### Authentication Header

All API calls use Bearer token auth:
```
Authorization: Bearer r5_live_...
```

---

## Links

- **Dashboard:** https://creditclaw.com
- **API Docs:** https://creditclaw.com/docs
- **Register:** https://creditclaw.com/api/v1/bots/register
- **Support:** support@creditclaw.com
