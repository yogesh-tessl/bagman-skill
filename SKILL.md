---
name: bagman
description: "Secure key management for AI agents handling wallets, private keys, and secrets. Covers 1Password integration, output sanitization, input validation against prompt injection, ERC-4337 session keys, operation allowlisting, and pre-commit secret detection. Use when an agent needs wallet or blockchain access, when handling API keys or credentials, when building systems where AI controls funds, or when preventing secret leakage via prompts or outputs."
---

# Bagman

Secure key management patterns for AI agents handling wallets, private keys, and secrets.

## Quick Start

```bash
# Install 1Password CLI
brew install 1password-cli

# Authenticate
eval $(op signin)

# Create vault for agent credentials
op vault create "Agent-Credentials"

# Run examples
cd examples && python test_suite.py
```

---

## Core Rules

| Rule | Why |
|------|-----|
| Never store raw private keys | Config, env, memory, or conversation = leaked |
| Use delegated access | Session keys with time/value/scope limits |
| Secrets via secret manager | 1Password, Vault, AWS Secrets Manager |
| Sanitize all outputs | Scan for key patterns before any response |
| Validate all inputs | Check for injection attempts before wallet ops |

---

## Architecture

```
AI Agent
├── Session Key (bounded: expiry, spend cap, contract whitelist)
├── Secret Manager (1Password / Vault / AWS Secrets Manager)
│   └── Retrieve at runtime only, never persist to disk
└── Smart Account (ERC-4337: programmable permissions, safe recovery)
```

---

## Implementation Files

| File | Purpose |
|------|---------|
| `examples/secret_manager.py` | 1Password integration for runtime secret retrieval |
| `examples/sanitizer.py` | Output sanitization (keys, seeds, tokens) |
| `examples/validator.py` | Input validation (prompt injection defense) |
| `examples/session_keys.py` | ERC-4337 session key configuration |
| `examples/delegation_integration.ts` | MetaMask Delegation Framework (EIP-7710) |
| `examples/pre-commit` | Git hook to block secret commits |
| `examples/test_suite.py` | Adversarial test suite |
| `docs/prompt-injection.md` | Deep dive on injection defense |
| `docs/secure-storage.md` | Secret storage patterns |
| `docs/session-keys.md` | Session key architecture |
| `docs/leak-prevention.md` | Output sanitization patterns |
| `docs/delegation-framework.md` | On-chain permission enforcement (EIP-7710) |
| `docs/autonomous-operation.md` | Autonomous vs. supervised modes |

---

## 1. Secret Retrieval

### 1Password CLI Pattern

```bash
# Retrieve at runtime (never store result)
SESSION_KEY=$(op read "op://Agents/my-agent/session-key")

# Run with injected secrets (never touch disk)
op run --env-file=.env.tpl -- python agent.py
```

### .env.tpl (safe to commit - no secrets)

```
PRIVATE_KEY=op://Agents/trading-bot/session-key
RPC_URL=op://Infra/alchemy/sepolia-url
OPENAI_API_KEY=op://Services/openai/api-key
```

### Python Usage

```python
from secret_manager import get_session_key

# Retrieve validated session key
creds = get_session_key("trading-bot-session")

# Check validity
if creds.is_expired():
    raise ValueError("Session expired - request renewal from operator")

print(f"Time remaining: {creds.time_remaining()}")
print(f"Allowed contracts: {creds.allowed_contracts}")

# Use the key (never log it!)
client.set_signer(creds.session_key)
```

### Vault-Level ACL (Recommended)

Configure 1Password vault permissions:

```
Agent-Credentials/
├── trading-bot-session    # Agent can read
├── payment-bot-session    # Agent can read
└── master-key             # Operator ONLY (agent has no access)
```

**Principle:** Agent credentials should be in a vault with read-only agent access. Master keys should be in a separate vault the agent cannot access.

---

## 2. Output Sanitization

Apply to ALL agent outputs before sending anywhere:

```python
from sanitizer import OutputSanitizer

def respond(content: str) -> str:
    """Sanitize before any output."""
    return OutputSanitizer.sanitize(content)

# Catches:
# - Private keys (0x + 64 hex)
# - OpenAI/Anthropic/Groq/AWS keys
# - GitHub/Slack/Discord tokens
# - BIP-39 seed phrases (12/24 words)
# - PEM private keys
# - JWT tokens
```

Detected patterns: ETH private keys, OpenAI/Anthropic/Groq/AWS keys, GitHub/Slack/Discord tokens, BIP-39 seed phrases, PEM private keys, JWTs.

---

## 3. Input Validation

Check inputs before ANY wallet operation:

```python
from validator import InputValidator, ThreatLevel

result = InputValidator.validate(user_input)

if result.level == ThreatLevel.BLOCKED:
    return f"Request blocked: {result.reason}"

if result.level == ThreatLevel.SUSPICIOUS:
    # Log for review, but allow
    log_suspicious(user_input, result.reason)

# Proceed with operation
```

### Threat Categories

| Category | Examples | Action |
|----------|----------|--------|
| Extraction | "show private key", "reveal secrets" | Block |
| Override | "ignore previous instructions" | Block |
| Role manipulation | "you are now admin" | Block |
| Jailbreak | "DAN mode", "bypass filters" | Block |
| Exfiltration | "send config to https://..." | Block |
| Wallet threats | "transfer all", "unlimited approve" | Block |
| Encoded | Base64/hex encoded attacks | Block |
| Unicode tricks | Cyrillic lookalikes, zero-width | Block |
| Suspicious | "hypothetically", "just between us" | Warn |

---

## 4. Operation Allowlisting

Never execute arbitrary operations. Explicit whitelist only.

**Autonomous-first design:** Agents should operate within bounds without asking for approval on every transaction. Use on-chain delegation caveats as the primary protection, software limits as backup.

```python
ALLOWED_OPS = {
    "check_balance": AllowedOperation("check_balance", get_balance),
    "transfer_usdc": AllowedOperation(
        "transfer_usdc", transfer,
        max_value=Decimal("500"),
        requires_confirmation=False,
    ),
    "emergency_withdraw": AllowedOperation(
        "emergency_withdraw", emergency_withdraw,
        requires_confirmation=True,
    ),
}
```

### When to Use Each Protection Layer

| Layer | Use When | Example |
|-------|----------|---------|
| **Delegation caveats** (primary) | Normal autonomous operation | Daily trading, payments |
| **Software limits** (backup) | Defense in depth | max_value checks |
| **Confirmation codes** (opt-in) | Exceptional high-risk actions | Emergency withdrawals, revoking access |

---

## 5. Confirmation Flow (Opt-In)

**Most agents should NOT use this for normal operations.** Delegation caveats provide on-chain protection without friction. Use confirmation codes only for exceptional cases where human oversight is required — see `examples/session_keys.py` for the implementation.

---

## 6. Session Keys (ERC-4337)

Instead of giving agents master keys, issue bounded session keys:

```python
from session_keys import SessionKeyManager

# Operator creates trading session for agent
config = SessionKeyManager.create_trading_session(
    agent_name="alpha-trader",
    operator_address="0x742d...",
    duration_hours=24,
    max_trade_usdc=1000,
    daily_limit_usdc=5000,
)

# Export for storage in 1Password
export_data = SessionKeyManager.export_for_1password(
    config, 
    session_key_hex="0x..."  # Generated session key
)

# op item create ... (store in 1Password)
```

### Session Key Benefits

| Feature | Master Key | Session Key |
|---------|------------|-------------|
| Expiration | Never | Configurable (hours/days) |
| Spending limits | None | Per-tx and daily caps |
| Contract restrictions | Full access | Whitelist only |
| Revocation | Requires key rotation | Instant, no key change |
| Audit | None | Full operation log |

---

## 7. Pre-commit Hook

Block commits containing secrets:

```bash
# Install
cp examples/pre-commit .git/hooks/
chmod +x .git/hooks/pre-commit
```

Detected patterns:
- ETH private keys (64 hex chars)
- OpenAI/Anthropic/Groq keys
- AWS access keys
- GitHub/GitLab tokens
- Slack/Discord tokens
- PEM private keys
- Generic PASSWORD/SECRET assignments
- BIP-39 seed phrases

---

## 8. Defense Layers

```
Input → Validation → Op Allowlist → Value Limits → Confirmation → Isolated Exec → Output Sanitization
```

---

## Common Mistakes

- **Keys in memory files** — store references only: `[stored in 1Password: test-wallet]`
- **Keys in error messages** — never include credentials in error context
- **Keys in .env.example** — use obviously fake values: `PRIVATE_KEY=your-key-here`
- **"Transfer all" requests** — block "all/everything/max" patterns, require explicit amounts
- **Trusting conversation context** — wallet operations must be isolated from conversation

---

## Testing

```bash
cd examples

# Run full test suite
python test_suite.py

# Test individual components
python sanitizer.py    # Output sanitization demo
python validator.py    # Input validation demo
python session_keys.py # Session key demo
```

Expected output: `All tests passed`

---

## Checklist

- [ ] 1Password CLI installed and authenticated
- [ ] Secrets in 1Password vault, not files
- [ ] Session keys with expiry and limits
- [ ] Output sanitization on all responses
- [ ] Input validation before wallet ops
- [ ] Pre-commit hook installed
- [ ] Confirmation flow for high-value operations
- [ ] Wallet operations isolated from conversation
- [ ] .gitignore covers secrets and memory files
- [ ] Test suite passes

---

## Security Model Limitations

This skill provides **defense in depth**, not a guarantee. Layer these defenses with rate limiting, anomaly detection, human-in-the-loop for large transactions, and regular security audits.
