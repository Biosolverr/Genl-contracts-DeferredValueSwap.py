# DeferredValueSwap — Verified Test Results & Deployment Guide

## Overview

DeferredValueSwap is an escrow-style Intelligent Contract built on GenLayer that enables trust-minimized value exchange between two parties.

The contract supports:

* Escrow deposits
* Deal activation by the recipient
* Settlement and payout
* Deal cancellation
* Dispute creation and resolution
* Protocol fee collection

The following guide is based on actual executions performed in GenLayer Studio.

---

# Contract Deployment

## Required Runner Header

The first line of the contract must contain the pinned GenLayer runtime:

```python
# { "Depends": "py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6" }
```

Using `latest` is not supported for production deployments and may result in:

* `invalid_contract`
* `absent_runner_comment`

---

# Contract Configuration

After deployment, configure the contract with the following values.

## Fees

| Parameter         | Value |
| ----------------- | ----- |
| activation_fee    | 1 GEN |
| transform_fee_bps | 100   |
| multiplier_bps    | 10000 |

## Timeouts

| Parameter           | Value |
| ------------------- | ----- |
| activation_period   | 0     |
| finalization_period | 0     |
| dispute_period      | 600   |
| min_delay           | 0     |

---

# Test 1 — Happy Path

## Objective

Client creates a deal, recipient activates it, and receives payment.

### Step 1 — Create Deal

Account A creates a deal.

Parameters:

```text
recipient_hex = Account B
value         = 1
offchain_ref  = test_job_1
```

Transaction value:

```text
11 GEN
```

Result:

```json
{
  "id": "1",
  "state": "0",
  "initial_value": "10000000000000000000"
}
```

Deal enters:

```text
INACTIVE (0)
```

Deposit locked:

```text
10 GEN
```

---

### Step 2 — Activate Deal

Account B calls:

```text
activate_deal(1)
```

Verified result:

```json
{
  "state": "1",
  "activated_at": "1780720266"
}
```

Deal enters:

```text
ACTIVE (1)
```

---

### Step 3 — Finalize

Account B calls:

```text
finalize_to_stable(1)
```

Verified result:

```json
{
  "state": "3",
  "finalized_at": "1780720431"
}
```

Deal enters:

```text
FINAL_STABLE (3)
```

### Financial Outcome

| Item               | Amount  |
| ------------------ | ------- |
| Deposit            | 10 GEN  |
| Transform Fee      | 0.1 GEN |
| Recipient Receives | 9.9 GEN |

Test Result:

✅ PASSED

---

# Test 2 — Cancellation

## Objective

Client cancels an inactive deal.

### Step 1 — Create Deal

Account A creates:

```text
offchain_ref = test_job_2
```

Transaction value:

```text
6 GEN
```

Result:

```json
{
  "id": "2",
  "state": "0",
  "initial_value": "5000000000000000000"
}
```

Deposit:

```text
5 GEN
```

---

### Step 2 — Cancel

Account A calls:

```text
cancel_inactive(2)
```

Verified result:

```json
{
  "state": "4",
  "finalized_at": "1780725283"
}
```

Deal enters:

```text
CANCELLED (4)
```

### Financial Outcome

| Item                    | Amount |
| ----------------------- | ------ |
| Refunded to Client      | 5 GEN  |
| Activation Fee Refunded | No     |

Test Result:

✅ PASSED

---

# Test 3 — Dispute Resolution

## Objective

Verify dispute creation, challenge, and resolution flow.

### Step 1 — Create Deal

Account A creates:

```text
offchain_ref = test_job_3
```

Transaction value:

```text
21 GEN
```

Result:

```json
{
  "id": "3",
  "state": "0",
  "initial_value": "20000000000000000000"
}
```

Deposit:

```text
20 GEN
```

---

### Step 2 — Activate

Account B calls:

```text
activate_deal(3)
```

Result:

```text
SUCCESS
```

State:

```text
ACTIVE (1)
```

---

### Step 3 — Open Dispute

Account A calls:

```text
open_dispute(
    3,
    "freelancer did not deliver the work"
)
```

Result:

```text
SUCCESS
```

---

### Step 4 — Challenge Dispute

Account B calls:

```text
challenge_dispute(
    3,
    "work delivered, client confirmed on Slack"
)
```

Result:

```text
SUCCESS
```

---

### Step 5 — Resolve Dispute

Owner calls:

```text
resolve_dispute(
    3,
    1,
    "manual resolution - AI arbitration planned in mode 2"
)
```

Verified result:

```json
{
  "state": "4",
  "claim_by": "Account A",
  "challenge_by": "Account B"
}
```

Deal enters:

```text
CANCELLED (4)
```

### Financial Outcome

| Item               | Amount |
| ------------------ | ------ |
| Refunded to Sender | 20 GEN |
| Paid to Recipient  | 0 GEN  |

Test Result:

✅ PASSED

---

# Final Contract Statistics

Verified after completion of all tests:

```json
{
  "owner": "0x3c163AeF7207dA6Dd839d839Ba1b7b068b128649",
  "protocol_fees": "3100000000000000000",
  "total_deals": "3"
}
```

## Fee Breakdown

| Source                  | Amount  |
| ----------------------- | ------- |
| Activation Fee (Deal 1) | 1 GEN   |
| Activation Fee (Deal 2) | 1 GEN   |
| Activation Fee (Deal 3) | 1 GEN   |
| Transform Fee (Deal 1)  | 0.1 GEN |
| Total                   | 3.1 GEN |

---

# State Transition Summary

```text
create_deal
    │
    ▼
INACTIVE (0)
    │
    ├─ cancel_inactive()
    │       ▼
    │   CANCELLED (4)
    │
    └─ activate_deal()
            ▼
        ACTIVE (1)
            │
            ├─ finalize_to_stable()
            │       ▼
            │   FINAL_STABLE (3)
            │
            └─ open_dispute()
                    ▼
                DISPUTED (5)
                    │
                    └─ resolve_dispute()
                            ▼
                        CANCELLED (4)
```

---

# Verified Results Summary

| Deal ID | Scenario           | Final State      |
| ------- | ------------------ | ---------------- |
| 1       | Happy Path         | FINAL_STABLE (3) |
| 2       | Cancellation       | CANCELLED (4)    |
| 3       | Dispute Resolution | CANCELLED (4)    |

All contract flows were successfully executed and verified on GenLayer Studio.

