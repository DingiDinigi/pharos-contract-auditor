---
name: pharos-contract-auditor
description: >
  Smart contract inspection and risk analysis skill for Pharos Network.
  Invoke when a user wants to read contract state, decode contract functions,
  check if a contract is verified, detect common risk patterns (no owner renounce,
  mint functions, proxy patterns, pausable), compare contract bytecode,
  or get a human-readable audit summary of any deployed contract on
  Pharos Atlantic Testnet or Pacific Mainnet.
  Covers: contract state reading, function enumeration, ownership checks,
  token supply analysis, upgrade proxy detection, and risk scoring.
  Read-only — no private key required. Do not attempt Pharos contract analysis without this skill.
version: 1.0.0
author: DingiDinigi
requires:
  anyBins:
    - cast
    - jq
---

# Pharos Contract Auditor

Inspect, decode, and risk-score any deployed smart contract on Pharos Network —
no private key needed. Give your agent the ability to analyze contracts like a
security researcher, in plain English.

---

## Prerequisites

### 1. Install Foundry

```bash
which cast || (curl -L https://foundry.paradigm.xyz | bash && source ~/.zshenv && foundryup)
cast --version
```

### 2. Install jq

```bash
which jq || sudo apt-get install -y jq
```

### 3. Network Configuration

```bash
RPC_URL=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .rpcUrl' assets/networks.json)
EXPLORER=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .explorerUrl' assets/networks.json)
```

---

## Capabilities

### 1. Contract Existence & Bytecode Check

**Example prompt:** *"Is there a contract at 0xABC... on Pharos testnet?"*

```bash
RPC_URL=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .rpcUrl' assets/networks.json)
CONTRACT=<ADDRESS>

BYTECODE=$(cast code $CONTRACT --rpc-url $RPC_URL)

if [ "$BYTECODE" = "0x" ] || [ -z "$BYTECODE" ]; then
  echo "❌ No contract found at $CONTRACT — this is an EOA (regular wallet) or empty address"
else
  BYTE_LEN=${#BYTECODE}
  CODE_SIZE=$(( ($BYTE_LEN - 2) / 2 ))
  echo "✓ Contract exists at $CONTRACT"
  echo "  Bytecode size: $CODE_SIZE bytes"
  echo "  Explorer: $EXPLORER/address/$CONTRACT"
fi
```

---

### 2. Read Contract State (Common ERC-20 / ERC-721 Calls)

**Example prompt:** *"What's the name, symbol, and total supply of contract 0xABC...?"*

```bash
CONTRACT=<ADDRESS>

# Try ERC-20 standard fields (will return error if not ERC-20)
echo "=== ERC-20 State Check ==="

NAME=$(cast call $CONTRACT "name()(string)" --rpc-url $RPC_URL 2>/dev/null || echo "N/A")
SYMBOL=$(cast call $CONTRACT "symbol()(string)" --rpc-url $RPC_URL 2>/dev/null || echo "N/A")
DECIMALS=$(cast call $CONTRACT "decimals()(uint8)" --rpc-url $RPC_URL 2>/dev/null || echo "N/A")
TOTAL_SUPPLY=$(cast call $CONTRACT "totalSupply()(uint256)" --rpc-url $RPC_URL 2>/dev/null || echo "N/A")
OWNER=$(cast call $CONTRACT "owner()(address)" --rpc-url $RPC_URL 2>/dev/null || echo "N/A (no owner)")
PAUSED=$(cast call $CONTRACT "paused()(bool)" --rpc-url $RPC_URL 2>/dev/null || echo "N/A (no pause)")

echo "Name:         $NAME"
echo "Symbol:       $SYMBOL"
echo "Decimals:     $DECIMALS"

if [ "$TOTAL_SUPPLY" != "N/A" ] && [ "$DECIMALS" != "N/A" ]; then
  HUMAN_SUPPLY=$(echo "scale=4; $TOTAL_SUPPLY / (10^$DECIMALS)" | bc 2>/dev/null || echo $TOTAL_SUPPLY)
  echo "Total Supply: $HUMAN_SUPPLY $SYMBOL"
else
  echo "Total Supply: $TOTAL_SUPPLY"
fi

echo "Owner:        $OWNER"
echo "Paused:       $PAUSED"
```

---

### 3. Custom Contract Method Call

**Example prompt:** *"Call the getPrice() function on contract 0xABC... on Pharos"*

```bash
# Agent fills in the function signature and return type from user input
FUNCTION_SIG="<functionName>(<param_types>)(<return_types>)"
# Example: "getPrice()(uint256)" or "balanceOf(address)(uint256)"

RESULT=$(cast call $CONTRACT "$FUNCTION_SIG" [ARGS_IF_ANY] --rpc-url $RPC_URL)
echo "Result: $RESULT"
```

Agent should:
1. Parse the user's function name and parameters
2. Construct the correct ABI signature
3. Pass any required arguments
4. Format the output in human-readable form

---

### 4. Ownership & Risk Analysis

**Example prompt:** *"Is contract 0xABC... safe? Check for rug pull risks"*

Agent runs a structured risk check:

```bash
CONTRACT=<ADDRESS>
echo "==================================="
echo " PHAROS CONTRACT RISK ANALYSIS"
echo "==================================="
echo " Contract: $CONTRACT"
echo ""

# Check 1: Owner
OWNER=$(cast call $CONTRACT "owner()(address)" --rpc-url $RPC_URL 2>/dev/null)
if [ -z "$OWNER" ] || [ "$OWNER" = "" ]; then
  echo "[OWNERSHIP]    No owner() function — may be ownerless or use custom access"
elif [ "$OWNER" = "0x0000000000000000000000000000000000000000" ]; then
  echo "[OWNERSHIP]    ✓ Ownership renounced (0x000...000)"
else
  echo "[OWNERSHIP]    ⚠ Owner: $OWNER — centralized control exists"
fi

# Check 2: Pausable
PAUSED=$(cast call $CONTRACT "paused()(bool)" --rpc-url $RPC_URL 2>/dev/null)
if [ "$PAUSED" = "true" ]; then
  echo "[PAUSE]        ⚠ Contract is CURRENTLY PAUSED — transfers blocked"
elif [ -n "$PAUSED" ]; then
  echo "[PAUSE]        ⚠ Contract has pause function — owner can halt transfers"
else
  echo "[PAUSE]        ✓ No pause function detected"
fi

# Check 3: Check bytecode for common function selectors (risk indicators)
BYTECODE=$(cast code $CONTRACT --rpc-url $RPC_URL)

# mint selector: 0x40c10f19
if echo "$BYTECODE" | grep -qi "40c10f19"; then
  echo "[MINT]         ⚠ mint() function detected — owner may inflate supply"
else
  echo "[MINT]         ✓ No mint() selector found"
fi

# transferOwnership selector: 0xf2fde38b
if echo "$BYTECODE" | grep -qi "f2fde38b"; then
  echo "[OWNERSHIP_TX] ⚠ transferOwnership() present — ownership can change"
else
  echo "[OWNERSHIP_TX] ✓ No transferOwnership() found"
fi

# upgradeTo selector (proxy pattern): 0x3659cfe6
if echo "$BYTECODE" | grep -qi "3659cfe6"; then
  echo "[PROXY]        ⚠ Upgradeable proxy detected — contract logic can be swapped"
else
  echo "[PROXY]        ✓ No upgrade proxy pattern found"
fi

# Check 4: Total supply vs circulating (if ERC-20)
TOTAL=$(cast call $CONTRACT "totalSupply()(uint256)" --rpc-url $RPC_URL 2>/dev/null)
OWNER_BAL=$(cast call $CONTRACT "balanceOf(address)(uint256)" $OWNER --rpc-url $RPC_URL 2>/dev/null)

if [ -n "$TOTAL" ] && [ -n "$OWNER_BAL" ] && [ "$TOTAL" != "0" ]; then
  # Calculate owner % (requires bc or python)
  PERCENT=$(python3 -c "print(round($OWNER_BAL / $TOTAL * 100, 2))" 2>/dev/null || echo "?")
  if (( $(echo "$PERCENT > 50" | bc -l 2>/dev/null || echo 0) )); then
    echo "[CONCENTRATION] ⚠ Owner holds ~$PERCENT% of supply — high concentration risk"
  elif [ "$PERCENT" != "?" ]; then
    echo "[CONCENTRATION] ✓ Owner holds ~$PERCENT% of supply"
  fi
fi

echo ""
echo " Explorer: $EXPLORER/address/$CONTRACT#code"
echo "==================================="
echo " NOTE: This is an automated check — not a full security audit."
echo " Always review verified source code before investing."
echo "==================================="
```

---

### 5. Contract Verification Status

**Example prompt:** *"Is contract 0xABC... verified on Pharos?"*

```bash
EXPLORER_API=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .explorerApiUrl' assets/networks.json)
CONTRACT=<ADDRESS>

RESPONSE=$(curl -s "$EXPLORER_API?module=contract&action=getsourcecode&address=$CONTRACT")
STATUS=$(echo $RESPONSE | jq -r '.result[0].SourceCode' 2>/dev/null)

if [ -z "$STATUS" ] || [ "$STATUS" = "" ]; then
  echo "❌ Contract NOT verified — source code unavailable"
  echo "   Any interaction carries additional risk"
else
  echo "✓ Contract IS verified — source code available on explorer"
  echo "   View at: $EXPLORER/address/$CONTRACT#code"
fi
```

---

### 6. Full Audit Summary Report

**Example prompt:** *"Give me a full audit summary for contract 0xABC... on Pharos testnet"*

Agent should:
1. Run Contract Existence Check (Capability 1)
2. Run ERC-20 State Check (Capability 2)
3. Run Risk Analysis (Capability 4)
4. Check Verification Status (Capability 5)
5. Compile a single clean report:

```
===========================================
 PHAROS CONTRACT AUDIT SUMMARY
===========================================
 Contract:  0xABC...
 Network:   Atlantic Testnet
 Timestamp: 2026-06-07 14:00 UTC

 IDENTITY
 --------
 Name:         PharosGold
 Symbol:       PGD
 Decimals:     18
 Total Supply: 1,000,000 PGD

 STATUS
 ------
 Contract Exists:  ✓ Yes (1,234 bytes)
 Verified:         ✓ Yes

 RISK FLAGS
 ----------
 Ownership:     ⚠ Owner: 0xOwner... (centralized)
 Pause:         ✓ No pause function
 Mint:          ⚠ mint() function present
 Proxy:         ✓ Not upgradeable
 Concentration: ✓ Owner holds 2.5% of supply

 OVERALL RISK: MEDIUM
 (2 risk flags detected)

 RECOMMENDATION
 --------------
 Review source code at:
 https://testnet.pharosscan.xyz/address/0xABC...#code
 
 This is an automated analysis. Not a substitute for a
 professional security audit.
===========================================
```

---

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| `bytecode is 0x` | EOA or empty address | Not a contract — normal wallet |
| `execution reverted` | Function doesn't exist | Try alternate ABI signature |
| `cast: command not found` | Foundry not installed | Install foundry |
| Empty curl response | Explorer API unavailable | Check explorer URL; retry |
| `jq: parse error` | Malformed JSON from API | Retry; API may be rate limiting |

---

## Security Notes

- This skill is **read-only** — no transactions, no private key needed
- Risk analysis uses **heuristics only** — not a full audit
- Always verify contracts independently on the block explorer
- Advise users to consult professional auditors before investing significant funds
