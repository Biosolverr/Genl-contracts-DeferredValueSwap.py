# GenLayer Quickstart: Deferred Value Swap with AI Conditions

## Overview

This quickstart shows how to build a **Deferred Value Swap** where execution depends on a condition evaluated later — simulating how GenLayer enables **AI-driven smart contract execution**.

By the end, you will:

* Deploy a contract
* Create a deferred swap
* Execute it based on a condition

---

## What You’re Building

A minimal system that:

1. Locks ETH
2. Stores a condition
3. Evaluates it (mock AI)
4. Executes only if valid

---

## Quickstart (5 Minutes)

### 1. Clone & Build

```bash
git clone https://github.com/Biosolverr/contracts-DeferredValueSwap.py
cd contracts-DeferredValueSwap.py
forge build
```

---

### 2. Run Local Node

```bash
anvil
```

---

### 3. Deploy Contract

```bash
forge create src/DeferredValueSwap.sol:DeferredValueSwap \
  --private-key <YOUR_PRIVATE_KEY> \
  --rpc-url http://127.0.0.1:8545
```

Save the deployed contract address.

---

### 4. Create a Deferred Swap

```bash
cast send <CONTRACT_ADDRESS> \
  "createSwap(address,string)" \
  <RECIPIENT_ADDRESS> "ALLOW" \
  --value 1ether \
  --private-key <YOUR_PRIVATE_KEY>
```

What happens:

* ETH is locked
* Condition `"ALLOW"` is stored
* Swap is pending

---

### 5. Execute the Swap

```bash
cast send <CONTRACT_ADDRESS> \
  "executeSwap(uint256)" 0 \
  --private-key <YOUR_PRIVATE_KEY>
```

If condition is valid:
→ ETH is transferred

---

## Smart Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract DeferredValueSwap {

    struct Swap {
        address sender;
        address recipient;
        uint256 amount;
        string condition;
        bool executed;
    }

    mapping(uint256 => Swap) public swaps;
    uint256 public swapId;

    function createSwap(address recipient, string memory condition) external payable {
        require(msg.value > 0, "No ETH sent");

        swaps[swapId] = Swap({
            sender: msg.sender,
            recipient: recipient,
            amount: msg.value,
            condition: condition,
            executed: false
        });

        swapId++;
    }

    function executeSwap(uint256 id) external {
        Swap storage s = swaps[id];

        require(!s.executed, "Already executed");
        require(_evaluateCondition(s.condition), "Condition not met");

        s.executed = true;
        payable(s.recipient).transfer(s.amount);
    }

    // Mock AI / GenLayer-style evaluation
    function _evaluateCondition(string memory condition) internal pure returns (bool) {
        return keccak256(bytes(condition)) == keccak256(bytes("ALLOW"));
    }
}
```

---

## How This Maps to GenLayer

In this tutorial:

* Condition = string
* Evaluation = mock function

In GenLayer:

* Condition → AI-readable input
* Evaluation → off-chain inference
* Execution → autonomous trigger

This pattern demonstrates:
→ **“execute when condition becomes true”**

---

## Upgrade to Real GenLayer Logic

Replace:

```solidity
_evaluateCondition(...)
```

With:

* AI oracle response
* Off-chain inference result
* GenLayer execution hook

---

## Expected Result

After running the steps:

* Swap is created
* Execution succeeds only if condition == `"ALLOW"`
* Funds are transferred

---

## Troubleshooting

**Transaction fails?**

* Check Anvil is running
* Verify private key
* Ensure contract address is correct

**Execution fails?**

* Condition must be `"ALLOW"`

---

## Next Steps

* Replace mock with oracle
* Add time-based conditions
* Integrate AI evaluation
* Build autonomous agents

---

## Repository

https://github.com/Biosolverr/contracts-DeferredValueSwap.py
