# CreditClaw — Perplexity Agent API Integration

Use CreditClaw as custom function tools in the Perplexity Agent API.

## Setup

```python
import requests

response = requests.post("https://api.perplexity.ai/v1/agent", json={
    "model": "sonar",
    "messages": [{"role": "user", "content": "Buy me a USB-C cable under $15"}],
    "tools": [
        {
            "type": "function",
            "name": "creditclaw_checkout",
            "description": "Request purchase approval with CreditClaw card",
            "parameters": {
                "type": "object",
                "properties": {
                    "merchant_name": {"type": "string"},
                    "item_name": {"type": "string"},
                    "amount_cents": {"type": "integer"}
                },
                "required": ["merchant_name", "item_name", "amount_cents"]
            }
        }
    ]
}, headers={"Authorization": f"Bearer {PERPLEXITY_API_KEY}"})
```

See `functions.json` for the complete function definitions.
