# CreditClaw — Financial Enablement for AI Agents

Give your AI agent a payment card with spending controls.

CreditClaw lets AI agents make purchases at any merchant website with human-in-the-loop approval, budget limits, and encrypted card handling.

## Works On

| Platform | Install Method |
|----------|---------------|
| **Claude Code** | `/plugin install creditclaw` |
| **Claude Cowork** | Install from Plugins directory |
| **OpenClaw** | `clawhub install creditclaw` |
| **Moltbook** | `clawhub install creditclaw` |
| **Zo Computer** | Copy `skills/creditclaw/` to your Skills folder |
| **Manus** | Provide SKILL.md as context |
| **Perplexity** | Agent API function definitions |
| **Any MCP client** | Connect to CreditClaw MCP server |

## Quick Start

### 1. Install the Skill

Pick your platform above and install.

### 2. Register Your Agent

Your agent will automatically register with CreditClaw when first activated. Or register manually:

```bash
curl -X POST https://creditclaw.com/api/v1/bots/register \
  -H "Content-Type: application/json" \
  -d '{"bot_name": "my-agent", "owner_email": "you@example.com"}'
```

### 3. Claim Your Agent

Visit the `claim_url` from the registration response to:
- Link your Stripe payment method
- Set spending limits
- Configure approval rules

### 4. Start Shopping

Tell your agent: *"Buy me a USB-C cable from Amazon under $15"*

The agent will:
1. Search for the item
2. Request checkout approval (you get notified)
3. Complete the purchase with your CreditClaw card
4. Report back with the order confirmation

## Features

- **Secure checkout** — Card data encrypted with AES-256-GCM, decrypted only inside the browser
- **Spending controls** — Per-transaction, daily, weekly, monthly limits
- **Category blocking** — Block specific merchant categories
- **Approval modes** — Auto-approve within limits, require approval for everything, or disable
- **Merchant auth** — Save logins for repeat purchases with TOTP 2FA support
- **Transaction history** — Full visibility into what your agent buys

## Security

Card data **never enters the AI's context window**. The flow:

```
CreditClaw API → encrypted blob → browser's crypto.subtle.decrypt → form fill
```

The AI agent only sees encrypted hex strings. Decryption happens atomically inside the browser's JavaScript runtime using the Web Crypto API (AES-256-GCM).

## Links

- **Website:** https://creditclaw.com
- **API Docs:** https://creditclaw.com/docs
- **Support:** support@creditclaw.com
- **GitHub:** https://github.com/jononovo/creditclaw-skill
