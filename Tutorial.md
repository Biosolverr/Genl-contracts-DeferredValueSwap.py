DeferredValueSwap — AI-Powered Escrow on GenLayer
Overview

This tutorial demonstrates how to build and test a Deferred Value Swap — an intelligent escrow system designed for AI-driven execution on GenLayer.

It enables two parties to interact without trust:

A client locks funds
A freelancer completes work
The contract enforces execution
Disputes can be resolved (with future AI arbitration support)
⚡ Quickstart (5 Minutes)
Goal

Create a deal → activate it → finalize → recipient gets paid

Result: funds move from locked escrow → recipient

Step 1 — Create Deal

(Account A — client)

recipient: address of B
Value: 11 GEN

Result:

10 GEN = deposit
1 GEN = activation fee
Step 2 — Activate Deal

(Account B — freelancer)

deal becomes ACTIVE
Step 3 — Finalize

(Account B)

Call:

finalize_to_stable
✅ Expected Result
B receives ~9.9 GEN
0.1 GEN goes to protocol fees
Deal state = FINAL_STABLE (3)
What You Are Building

A full escrow system with:

Deposits
Activation flow
Finalization
Cancellation
Dispute resolution
🤖 AI / GenLayer Component

This contract is designed for AI-driven execution on GenLayer.

It aligns with GenLayer principles:

Intelligent execution
Off-chain reasoning
AI-assisted decision workflows
Current Implementation
mode = 1 → manual dispute resolution
Planned (GenLayer-native)
mode = 2 → AI arbitration
Off-chain inference decides outcome
Autonomous execution via AI layer
🧠 Why Python (GenLayer Advantage)

Unlike Solidity systems, this contract is written in Python.

This enables:

Easier AI integration
Flexible condition logic
Native compatibility with off-chain reasoning
Future AI-driven execution

This is a key advantage of GenLayer’s execution model.

Deal Lifecycle

Each deal has a state:

State	Name	Description
0	INACTIVE	Created but not activated
1	ACTIVE	Work in progress
2	FINAL_NFT	Completed without payout
3	FINAL_STABLE	Completed with payout
4	CANCELLED	Cancelled or refunded
5	DISPUTED	Under dispute
Flow
create_deal
    |
INACTIVE ---(cancel_inactive)---> CANCELLED
    |
activate_deal
    |
 ACTIVE ---(finalize_to_stable)---> FINAL_STABLE
    |    ---(finalize_to_nft)------> FINAL_NFT
    |    ---(open_dispute)----------> DISPUTED
    |
 DISPUTED ---(resolve_dispute)-----> CANCELLED
Setup (Owner)

Before running tests:

set_fees
Field	Value
activation_fee	1 GEN
transform_fee_bps	100 (1%)
multiplier_bps	10000
set_timeouts
Field	Value
activation_period	0
finalization_period	0
dispute_period	600 sec
min_delay	0
🧪 Test 1 — Happy Path
Scenario

Client pays → freelancer completes → gets paid

Step 1 — create_deal (A)
Value: 11 GEN
offchain_ref: test_job_1

Deposit:

10 GEN (after fee)
Step 2 — activate_deal (B)
Step 3 — finalize_to_stable (B)
✅ Result
B receives ~9.9 GEN
Protocol gets 0.1 GEN
state = 3 (FINAL_STABLE)
🧪 Test 2 — Cancellation
Scenario

Client cancels before activation

Step 1 — create_deal (A)
Value: 6 GEN
Step 2 — cancel_inactive (A)
✅ Result
A receives 5 GEN
Fee not refunded
state = 4 (CANCELLED)
🧪 Test 3 — Dispute
Scenario

Conflict between client and freelancer

Step 1 — create_deal (A)
Value: 21 GEN
Step 2 — activate_deal (B)
Step 3 — open_dispute (A)

Reason:

freelancer did not deliver the work
Step 4 — challenge_dispute (B)

Reason:

work delivered and client confirmed
Step 5 — resolve_dispute (OWNER)
mode = 1
note = manual resolution (AI arbitration planned in mode=2)
✅ Result
20 GEN returned to A
state = 4 (CANCELLED)
Verifying Results

Call:

get_deal_info(deal_id)
Expected
Deal	State	Result
1	3	Payment success
2	4	Cancelled
3	4	Dispute resolved
Methods Overview
Admin
set_fees
set_timeouts
resolve_dispute
User
create_deal
activate_deal
finalize_to_stable
cancel_inactive
open_dispute
challenge_dispute
View
get_deal_info
get_stats
get_contract_stats
Known Fixes

During testing:

Fixed missing UserError
Removed invalid Address.ZERO usage
Fixed deposit calculation logic
Resolved fractional GEN issue
🖥 Example Output (GenLayer Studio)
After create_deal
Tx Status: ACCEPTED

Method: create_deal
Args:
  recipient: 0xB...
  value: 1
  offchain_ref: test_job_1

Value Sent: 11 GEN

Result:
  deal_id: 1
  state: INACTIVE (0)
  deposit: 10 GEN
  activation_fee: 1 GEN
After activate_deal
Tx Status: ACCEPTED

Method: activate_deal
Args:
  deal_id: 1

Result:
  state: ACTIVE (1)
  activated_by: 0xB...
After finalize_to_stable
Tx Status: ACCEPTED

Method: finalize_to_stable
Args:
  deal_id: 1

Result:
  state: FINAL_STABLE (3)
  payout_to: 0xB...
  amount: 9.9 GEN
  protocol_fee: 0.1 GEN
📸 Optional Screenshot

You can include a real screenshot from GenLayer Studio:

![GenLayer Studio Result](./assets/genlayer-result.png)

Recommended:

create_deal transaction
finalize_to_stable result
get_deal_info output
💡 Why This Matters

This tutorial demonstrates:

end-to-end reproducible execution
escrow logic with deterministic states
AI-ready architecture design

This is a foundation for AI-powered execution systems on GenLayer.

Conclusion

You built an escrow system that:

Locks value securely
Executes conditionally
Handles disputes
Prepares for AI arbitration

This is a real foundation for AI-powered contracts on GenLayer.
