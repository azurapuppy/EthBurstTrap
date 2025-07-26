Drosera Trap — ETH Transaction Burst Detector

🎯 Objective
Detect and respond to abnormal surges of outgoing ETH transactions from a monitored wallet. This trap is useful for:

detecting exploits,

preventing mass outflows from treasury wallets,

flagging bots or contract bugs.

⚠️ Problem
DAO treasuries, governance-controlled contracts, and cold wallets typically move funds infrequently.
If a wallet suddenly performs 15+ outgoing ETH transactions in 10 minutes, it could indicate:

a compromise (key leak),

an automated exploit,

or misconfigured bot logic.

✅ Solution
Track outgoing ETH transactions off-chain.
Periodically update the trap with:

blocksSince: how many blocks ago the burst started,

txCount: how many outgoing tx occurred since then.

The trap triggers if both:

blocksSince ≤ 50 (≈10 minutes),

txCount ≥ 15.

🧠 Trap Logic Summary
Trap Contract: EthBurstTrap.sol


// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface ITrap {
    function collect() external view returns (bytes memory);
    function shouldRespond(bytes[] calldata data) external pure returns (bool, bytes memory);
}

contract EthBurstTrap is ITrap {
    uint256 public constant maxBlocksSince = 50;
    uint256 public constant txThreshold = 15;

    struct Snapshot {
        uint256 blocksSince;
        uint256 txCount;
    }

    Snapshot public snapshot;

    function updateSnapshot(uint256 blocksSince, uint256 txCount) external {
        snapshot = Snapshot(blocksSince, txCount);
    }

    function collect() external view override returns (bytes memory) {
        return abi.encode(snapshot.blocksSince, snapshot.txCount);
    }

    function shouldRespond(bytes[] calldata data) external pure override returns (bool, bytes memory) {
        if (data.length < 1) return (false, "Missing data");

        (uint256 blocksSince, uint256 txCount) = abi.decode(data[0], (uint256, uint256));

        if (blocksSince > maxBlocksSince) {
            return (false, "Snapshot too old");
        }

        if (txCount >= txThreshold) {
            return (true, "Burst of outgoing ETH transactions detected");
        }

        return (false, "");
    }
}
📢 Response Contract

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract LogAlertReceiver {
    event Alert(string message);

    function logAnomaly(string calldata message) external {
        emit Alert(message);
    }
}
🚀 Deployment & Setup Instructions
1. Deploy contracts
bash
Копировать
Редактировать
forge create src/EthBurstTrap.sol:EthBurstTrap \\
  --rpc-url https://ethereum-hoodi-rpc.publicnode.com \\
  --private-key 0x...

forge create src/LogAlertReceiver.sol:LogAlertReceiver \\
  --rpc-url https://ethereum-hoodi-rpc.publicnode.com \\
  --private-key 0x...
2. Update drosera.toml
toml
Копировать
Редактировать
[traps.eth_burst]
path = "out/EthBurstTrap.sol/EthBurstTrap.json"
response_contract = "<LogAlertReceiver address>"
response_function = "logAnomaly(string)"
3. Apply the trap
bash

export DROSERA_PRIVATE_KEY=0x...
drosera apply
🧪 Off-chain Activity Feed
You must run a script (e.g., burst_monitor.py) that:

Scans the last 50 blocks,

Counts from == target outgoing ETH txs,

Calculates blocksSince = currentBlock - firstTxBlock,

Calls:

solidity

trap.updateSnapshot(blocksSince, txCount)
🧩 Extensions & Improvements
Allow configuring thresholds via setThreshold().

Monitor ERC-20 transfers (not just native ETH).

Auto-throttle or freeze wallet after alert.

Chain multiple traps together in a TrapManager.
