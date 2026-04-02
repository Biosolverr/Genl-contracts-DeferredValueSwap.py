DeferredValueSwap — AI-Powered Escrow on GenLayer

Complete tutorial with real tested values


1. What We Are Building
DeferredValueSwap is a GenLayer intelligent contract for two-party escrow deals. Think of a freelance scenario:

A client (account A) wants work done and is ready to pay
A freelancer (account B) wants guaranteed payment
Neither side fully trusts the other

With this contract:

A creates a deal and locks GEN on the contract
B activates the deal and does the work
If everything is fine — B gets paid via finalize_to_stable
If there is a conflict — either side opens a dispute, the owner resolves it


💡 The contract is written in Python and deployed as a single file. No oracles, no Solidity.


2. Deal Lifecycle
Every deal moves through states. These numeric values appear in get_deal_info:
#StateDescription0INACTIVECreated and funded, not yet activated1ACTIVEWork in progress2FINAL_NFTCompleted as a certificate (no payout)3FINAL_STABLECompleted with payout to recipient4CANCELLEDCancelled or expired, deposit returned to sender5DISPUTEDFrozen in dispute until resolved
State transition diagram:
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

3. Setup Before Tests (OWNER)
After deployment the contract uses zero defaults. Run these two methods as the owner before any test.

⚠️ All amounts are in wei. 1 GEN = 1,000,000,000,000,000,000 (10¹⁸) wei.

Step 1 — set_fees [OWNER account]
FieldValueactivation_fee1000000000000000000 (1 GEN)transform_fee_bps100 (1%)multiplier_bps10000 (1.0x, no change)
Click Send Transaction
Step 2 — set_timeouts [OWNER account]
FieldValueactivation_period0 (no deadline)finalization_period0 (no deadline)dispute_period600 (10 minutes)min_delay0 (no throttle)
Click Send Transaction

4. Test 1 — Happy Path
Scenario: A creates a deal, B activates it and receives payment.
Step 1.1 — create_deal [Account A — client]
FieldValuerecipient_hex(address of account B)value1offchain_reftest_job_1Value (GEN)11 (= 10 deposit + 1 activation_fee)
Click Send Transaction

💡 The deposit is determined by the Value (GEN) field, not the value argument.
deposit = Value (GEN) − activation_fee → 11 − 1 = 10 GEN

Step 1.2 — activate_deal [Account B — freelancer]
FieldValuedeal_id_int1
Click Send Transaction
Step 1.3 — finalize_to_stable [Account B — freelancer]
FieldValuedeal_id_int1
Click Send Transaction
Expected result:

B receives ~9.9 GEN (10 minus 1% transform_fee)
0.1 GEN goes to protocol_fees
state = 3 (FINAL_STABLE)
B gets +5 XP and +1 win


5. Test 2 — Cancellation
Scenario: A creates a deal, then cancels it before B activates.
Step 2.1 — create_deal [Account A — client]
FieldValuerecipient_hex(address of account B)value1offchain_refcancel_testValue (GEN)6 (= 5 deposit + 1 activation_fee)
Click Send Transaction
Step 2.2 — cancel_inactive [Account A — client]
FieldValuedeal_id_int2
Click Send Transaction
Expected result:

A gets back 5 GEN (the deposit)
1 GEN activation_fee is not refunded
state = 4 (CANCELLED)


6. Test 3 — Dispute
Scenario: A creates a deal, B activates it, A opens a dispute, B responds, owner resolves.
Step 3.1 — create_deal [Account A — client]
FieldValuerecipient_hex(address of account B)value1offchain_refdispute_testValue (GEN)21 (= 20 deposit + 1 activation_fee)
Click Send Transaction
Step 3.2 — activate_deal [Account B — freelancer]
FieldValuedeal_id_int3
Click Send Transaction
Step 3.3 — open_dispute [Account A — client]
FieldValuedeal_id_int3reasonfreelancer did not deliver the work
Click Send Transaction
Step 3.4 — challenge_dispute [Account B — freelancer]
FieldValuedeal_id_int3reasonwork delivered and client confirmed
Click Send Transaction

⏳ Wait 10 minutes (dispute_period = 600 sec) before the next step.

Step 3.5 — resolve_dispute [OWNER account]
FieldValuedeal_id_int3mode1 (owner_manual)noteai arbitration
Click Send Transaction

ℹ️ mode=1 is a manual owner decision. In the current version the deposit always returns to the sender (A). mode=2 is reserved for future AI arbitration.

Expected result:

20 GEN deposit returned to account A
state = 4 (CANCELLED)
deal_resolution_mode = 1
deal_resolution_note = ai arbitration


7. Verifying Results
After each test call get_deal_info to inspect the final state:
get_deal_info [any account]
FieldValuedeal_id_int1 / 2 / 3
Expected state values:
Deal IDstateWhat happened13 — FINAL_STABLEB received payment24 — CANCELLEDA got deposit back34 — CANCELLEDDispute resolved, funds returned to A

8. Method Reference
Admin methods (owner only)
MethodDescriptionset_feesSet activation_fee, transform_fee_bps, multiplier_bpsset_timeoutsSet activation, finalization, dispute periods and min_delayset_limitsSet max_deal_value (0 = no limit)blacklistBlock or unblock an addressresolve_disputeResolve a dispute
User methods
MethodWhoDescriptioncreate_dealAnyoneCreate a deal and lock depositactivate_dealA or BMove deal to ACTIVEcancel_inactiveACancel before activation, refund depositexpire_dealA or BExpire by deadlinefinalize_to_stableBComplete with payout to Bfinalize_to_nftBComplete as certificate (no payout)open_disputeA or BOpen a disputechallenge_disputeA or BRespond to a dispute
View methods
MethodReturnsget_deal_info(deal_id_int)Full deal data: state, parties, timestamps, dispute infoget_stats(addr_hex)XP and wins for an addressget_contract_stats()Total deals, protocol fees collected, owner address

9. Bugs Fixed in This Version
The following issues were discovered and fixed during live testing on GenLayer Studio:

UserError not imported — added a compatibility shim at the top of the contract
Address.ZERO does not exist — replaced with .get() is not None checks
create_deal strict value match — reworked: deposit = msg.value − fee, the value argument is no longer used for deposit calculation
Fractional GEN rejected (1e-17 BigInt error) — Value (GEN) field is now the source of truth for deposit amount
