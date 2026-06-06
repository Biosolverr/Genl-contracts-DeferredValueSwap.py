# DeferredValueSwap Test Report

## Environment Setup

### 1. Contract Deployment

-   Status: FINALIZED
-   Result: SUCCESS
-   Contract deployed successfully.

### 2. Fee Configuration

-   activation_fee = 1 GEN
-   transform_fee_bps = 100
-   multiplier_bps = 10000

Status: FINALIZED

### 3. Initial Contract Stats

``` json
{
  "owner": "0x3c163AeF7207dA6Dd839d839Ba1b7b068b128649",
  "protocol_fees": "0",
  "total_deals": "0"
}
```

### 4. Timeout Configuration

-   activation_period = 0
-   finalization_period = 0
-   dispute_period = 600
-   min_delay = 0

Status: FINALIZED

------------------------------------------------------------------------

# Test 1 --- Happy Path

## Create Deal

Deal ID: 1

-   Deposit: 10 GEN
-   State: 0 (INACTIVE)

## Activate Deal

-   Status: SUCCESS
-   State: 1 (ACTIVE)

## Finalize Deal

-   Status: SUCCESS
-   State: 3 (FINAL_STABLE)

### Result

  Metric               Value
  -------------------- ------------------
  Deposit              10 GEN
  Transform Fee        0.1 GEN
  Recipient Received   9.9 GEN
  Final State          FINAL_STABLE (3)

✅ PASSED

------------------------------------------------------------------------

# Test 2 --- Cancellation

## Create Deal

Deal ID: 2

-   Deposit: 5 GEN
-   State: 0 (INACTIVE)

## Cancel Deal

-   Status: SUCCESS
-   State: 4 (CANCELLED)

### Result

  Metric                    Value
  ------------------------- ---------------
  Refunded to Sender        5 GEN
  Activation Fee Refunded   No
  Final State               CANCELLED (4)

✅ PASSED

------------------------------------------------------------------------

# Test 3 --- Dispute Resolution

## Create Deal

Deal ID: 3

-   Deposit: 20 GEN
-   State: 0 (INACTIVE)

## Activate Deal

SUCCESS

## Open Dispute

Claim: "freelancer did not deliver the work"

SUCCESS

## Challenge Dispute

Challenge: "work delivered, client confirmed on Slack"

SUCCESS

## Resolve Dispute

Mode: 1

SUCCESS

Final State:

-   claim_by = Sender
-   challenge_by = Recipient
-   state = 4 (CANCELLED)

### Result

  Metric               Value
  -------------------- ---------------
  Deposit              20 GEN
  Returned to Sender   20 GEN
  Paid to Recipient    0 GEN
  Final State          CANCELLED (4)

✅ PASSED

------------------------------------------------------------------------

# Final Contract Statistics

``` json
{
  "owner": "0x3c163AeF7207dA6Dd839d839Ba1b7b068b128649",
  "protocol_fees": "3100000000000000000",
  "total_deals": "3"
}
```

## Protocol Fee Breakdown

  Source                    Amount
  ------------------------- ---------
  Activation Fee (Deal 1)   1 GEN
  Activation Fee (Deal 2)   1 GEN
  Activation Fee (Deal 3)   1 GEN
  Transform Fee (Deal 1)    0.1 GEN
  Total                     3.1 GEN

------------------------------------------------------------------------

# Summary

  Deal ID   Scenario             Final State
  --------- -------------------- ------------------
  1         Happy Path           FINAL_STABLE (3)
  2         Cancellation         CANCELLED (4)
  3         Dispute Resolution   CANCELLED (4)

All tests completed successfully on GenLayer Studio.
