# DeferredValueSwap — AI-Powered Escrow on GenLayer

A step-by-step implementation guide for building and deploying an intelligent escrow contract on GenLayer.

## What This Contract Does

Two parties interact without needing to trust each other:

- A **client** locks funds into escrow
- A **freelancer** completes work and claims payment
- The contract enforces the rules deterministically
- Disputes can be opened by either party and resolved by the contract owner (with AI arbitration planned for mode 2)

---

## Contract Header (Required by GenVM)

The **first line** of every GenLayer contract must be a runner comment with the exact SDK hash. Using `latest` is not allowed in non-debug mode and will cause `invalid_contract absent_runner_comment`:

```python
# { "Depends": "py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6" }
```

This is equivalent to Solidity's `pragma solidity` — it pins the exact Python runtime version.

---

## Full Contract Structure

```python
# { "Depends": "py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6" }

from genlayer import *
import time
import json


@gl.evm.contract_interface
class _EOARecipient:
    class View:
        pass
    class Write:
        pass


class DeferredValueSwap(gl.Contract):
    # Storage — all fields must be declared as class-level annotations
    next_deal_id: u256
    deal_sender: TreeMap[u256, Address]
    deal_recipient: TreeMap[u256, Address]
    deal_state: TreeMap[u256, u256]
    deal_deposit: TreeMap[u256, u256]
    deal_current_value: TreeMap[u256, u256]
    # ... (full storage declaration in contract file)

    # State machine constants
    STATE_INACTIVE   = 0
    STATE_ACTIVE     = 1
    STATE_FINAL_NFT  = 2
    STATE_FINAL_STABLE = 3
    STATE_CANCELLED  = 4
    STATE_DISPUTED   = 5
```

### Key implementation decisions

**Why `@gl.evm.contract_interface` for transfers?**

Senders and recipients are EOA wallets, not Intelligent Contracts. To send GEN to an EOA you must use an external message via the EVM interface. Using `gl.get_contract_at(addr).emit_transfer()` only works for IC→IC transfers and will silently fail for EOAs.

```python
# ❌ Wrong — only works IC→IC
proxy = gl.get_contract_at(to)
proxy.emit_transfer(value=amount)

# ✅ Correct — works for EOA/EVM addresses
@gl.evm.contract_interface
class _EOARecipient:
    class View:
        pass
    class Write:
        pass

def _send_native(self, to: Address, amount: u256) -> None:
    if amount == u256(0):
        return
    _EOARecipient(to).emit_transfer(value=amount)
```

**Why `gl.vm.UserError` instead of a custom exception class?**

The original contract used a `try/except NameError` hack to define `UserError`. GenVM has its own error type that integrates with the consensus mechanism:

```python
# ❌ Wrong
try:
    UserError
except NameError:
    class UserError(Exception):
        pass

# ✅ Correct
raise gl.vm.UserError("Only owner")
```

**Why state constants as class attributes, not methods?**

```python
# ❌ Wrong — GenVM may expose these as public schema methods
def _STATE_INACTIVE(self) -> u256:
    return u256(0)

# ✅ Correct — plain class attributes, not visible in schema
STATE_INACTIVE = 0
```

**Storage field types**

GenLayer enforces specific types for persistent storage:

| Python type | GenLayer type |
|-------------|---------------|
| `int`       | `u256`, `u32`, etc. |
| `list[T]`   | `DynArray[T]` |
| `dict[K,V]` | `TreeMap[K,V]` |

All persistent fields must be declared as **class-level annotations**, not created dynamically in `__init__` with `self.field = value`.

---

## Deal Lifecycle

```
create_deal()
    │
    ▼
INACTIVE (0)
    │
    ├─ cancel_inactive() ──────────────────► CANCELLED (4)
    │
    ▼
activate_deal()
    │
    ▼
ACTIVE (1)
    │
    ├─ finalize_to_stable() ───────────────► FINAL_STABLE (3)
    ├─ finalize_to_nft() ─────────────────► FINAL_NFT (2)
    └─ open_dispute() ────────────────────► DISPUTED (5)
                                                │
                                                ▼
                                          resolve_dispute()
                                                │
                                                ▼
                                          CANCELLED (4)
```

---

## Deploying in GenLayer Studio

### Step 0 — Verify the runner comment

The schema panel in Studio will show `invalid_contract` if the first line is wrong or missing. The correct header:

```
# { "Depends": "py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6" }
```

### Step 1 — Deploy

Load the contract file in Studio → Deploy. Constructor has no parameters.

### Step 2 — Configure (owner account)

Call `set_fees`:

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `activation_fee` | `1000000000000000000` | 1 GEN in wei |
| `transform_fee_bps` | `100` | 1% payout fee |
| `multiplier_bps` | `10000` | 1.0x (no multiplier) |

Call `set_timeouts`:

| Parameter | Value |
|-----------|-------|
| `activation_period` | `0` (no deadline) |
| `finalization_period` | `0` (no deadline) |
| `dispute_period` | `600` (10 minutes) |
| `min_delay` | `0` |

---

## Test 1 — Happy Path

**Goal:** client pays → freelancer completes → gets paid

### create_deal (Account A)

```
recipient_hex: "0x<address_of_B>"
value: 1
offchain_ref: "test_job_1"
Value sent: 11000000000000000000  (11 GEN in wei)
```

Result: 1 GEN activation fee collected, 10 GEN locked as deposit. Returns `deal_id = 1`.

Verify with `get_deal_info(1)`:
```json
{
  "state": "0",
  "deposit": "10000000000000000000",
  "sender": "0xA...",
  "recipient": "0xB..."
}
```

### activate_deal (Account B)

```
deal_id_int: 1
```

State changes from `0` (INACTIVE) → `1` (ACTIVE).

### finalize_to_stable (Account B)

```
deal_id_int: 1
```

- `transform_fee` = 10 GEN × 1% = 0.1 GEN
- `net_amount` sent to B = 9.9 GEN
- Deal state → `3` (FINAL_STABLE)

---

## Test 2 — Cancellation

### create_deal (Account A)

```
Value sent: 6000000000000000000  (6 GEN)
```

Deposit = 5 GEN (after 1 GEN fee).

### cancel_inactive (Account A)

5 GEN returned to A. State → `4` (CANCELLED). The activation fee is not refunded.

---

## Test 3 — Dispute Resolution

### create_deal (Account A)

```
Value sent: 21000000000000000000  (21 GEN)
```

Deposit = 20 GEN.

### activate_deal (Account B)

### open_dispute (Account A)

```
deal_id_int: 3
reason: "freelancer did not deliver the work"
```

State → `5` (DISPUTED).

### challenge_dispute (Account B)

```
deal_id_int: 3
reason: "work delivered, client confirmed on Slack"
```

### resolve_dispute (Owner)

```
deal_id_int: 3
mode: 1
note: "manual resolution — mode=2 will use AI arbitration"
```

20 GEN returned to A. State → `4` (CANCELLED).

---

## Verifying Results

```python
get_deal_info(deal_id)
```

| deal_id | Expected state | Meaning |
|---------|---------------|---------|
| 1 | `3` | FINAL_STABLE — payment sent |
| 2 | `4` | CANCELLED |
| 3 | `4` | Dispute resolved, refunded |

```python
get_contract_stats()
# {"total_deals": "3", "protocol_fees": "1100000000000000000", "owner": "0x..."}
```

---

## AI Arbitration — Future Work

The contract has `deal_resolution_mode` storage for each deal:

- `mode = 1` → manual resolution by contract owner (current)
- `mode = 2` → AI arbitration (planned)

In mode 2, `resolve_dispute` would call an LLM via `gl.eq_principle` to read both `deal_claim_reason` and `deal_challenge_reason` and decide the outcome autonomously — without any human intervention. This is the core GenLayer value proposition: deterministic consensus over non-deterministic AI judgment.

```python
# Planned AI arbitration (mode 2)
@gl.public.write
def resolve_dispute_ai(self, deal_id_int: int) -> None:
    claim = self.deal_claim_reason[deal_id]
    challenge = self.deal_challenge_reason[deal_id]

    def arbitrate() -> str:
        result = gl.exec_prompt(
            f"Claim: {claim}\nChallenge: {challenge}\n"
            "Who is right? Reply SENDER or RECIPIENT."
        )
        return result.strip()

    verdict = gl.eq_principle.strict_eq(arbitrate)
    # route funds based on verdict
```

---

## Methods Reference

| Category | Method | Who calls it |
|----------|--------|-------------|
| Admin | `set_fees` | Owner |
| Admin | `set_timeouts` | Owner |
| Admin | `blacklist` | Owner |
| Admin | `set_limits` | Owner |
| Admin | `resolve_dispute` | Owner |
| User | `create_deal` | Client (payable) |
| User | `activate_deal` | Either party |
| User | `cancel_inactive` | Client only |
| User | `expire_deal` | Either party |
| User | `finalize_to_stable` | Recipient only |
| User | `finalize_to_nft` | Recipient only |
| User | `open_dispute` | Either party |
| User | `challenge_dispute` | Either party |
| View | `get_deal_info` | Anyone |
| View | `get_stats` | Anyone |
| View | `get_contract_stats` | Anyone |

---

## Implementation Bugs Fixed During Development

| Bug | Symptom | Fix |
|-----|---------|-----|
| `# { "Depends": "py-genlayer:latest" }` | `absent_runner_comment` / `invalid_contract` in Studio | Replace with pinned hash |
| Runner comment not on line 1 | Schema panel empty | Move comment to line 1, nothing before it |
| `gl.get_contract_at(to).emit_transfer()` for EOA | Silent transfer failure | Use `@gl.evm.contract_interface` pattern |
| Custom `UserError` class via `try/except NameError` | Errors not properly handled by GenVM | Use `gl.vm.UserError` |
| State machine as `def _STATE_X(self)` methods | Schema parsing issues | Replace with class-level integer constants |
| `self.protocol_fees_collected += fee` | Potential issue with storage `+=` | Rewrite as explicit `= self.x + fee` |
