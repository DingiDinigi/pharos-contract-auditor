# pharos-contract-auditor

> A Pharos Agent Skill — built for the Pharos Skill Builder Campaign by DingiDinigi

Inspect, decode, and risk-score any deployed smart contract on Pharos Network. No private key needed — purely read-only analysis your agent can run for you.

---

## What This Skill Does

| Capability | Example Prompt |
|---|---|
| Contract existence check | "Is there a contract at 0xABC... on Pharos?" |
| Read ERC-20 state | "What's the name, symbol and supply of 0xABC...?" |
| Custom function call | "Call getPrice() on contract 0xABC... on Pharos" |
| Risk analysis | "Is contract 0xABC... safe? Check for rug pull risks" |
| Verification status | "Is 0xABC... verified on Pharos testnet?" |
| Full audit summary | "Give me a full audit report for 0xABC..." |

Works on **Atlantic Testnet** (Chain ID: 688688) and **Pacific Mainnet** (Chain ID: 1672).  
**No private key required** — all operations are read-only.

---

## Risk Checks Performed

The skill automatically checks for:

- **Ownership**: Is there an owner? Has it been renounced?
- **Mint function**: Can the owner print unlimited tokens?
- **Pausable**: Can the owner freeze all transfers?
- **Proxy/Upgradeable**: Can the contract logic be silently swapped?
- **Supply concentration**: Does the owner hold most of the supply?
- **Verification**: Is the source code publicly visible on explorer?

Each flag is marked ✓ (safe) or ⚠ (risk) with an overall risk score.

---

## Installation Guide (Beginner Friendly)

### Step 1 — Install Foundry

```bash
curl -L https://foundry.paradigm.xyz | bash
source ~/.zshenv && foundryup
cast --version   # confirm it worked
```

### Step 2 — Install jq

```bash
# Ubuntu/Debian/WSL
sudo apt-get install -y jq

# macOS
brew install jq
```

### Step 3 — Install the Skill

**Claude Code:**

```bash
mkdir -p ~/.claude/skills/assets

curl -L https://raw.githubusercontent.com/DingiDinigi/pharos-contract-auditor/main/SKILL.md \
  -o ~/.claude/skills/pharos-contract-auditor.md

curl -L https://raw.githubusercontent.com/DingiDinigi/pharos-contract-auditor/main/assets/networks.json \
  -o ~/.claude/skills/assets/networks.json
```

**Codex CLI:**

```bash
mkdir -p ~/.codex/skills/assets

curl -L https://raw.githubusercontent.com/DingiDinigi/pharos-contract-auditor/main/SKILL.md \
  -o ~/.codex/skills/pharos-contract-auditor.md

curl -L https://raw.githubusercontent.com/DingiDinigi/pharos-contract-auditor/main/assets/networks.json \
  -o ~/.codex/skills/assets/networks.json
```

**OpenClaw:**

```bash
npx skills add https://github.com/DingiDinigi/pharos-contract-auditor
```

### Step 4 — Verify

Type `/skills` in a new session — you should see `pharos-contract-auditor` listed.

---

## How to Use

No setup beyond installation is needed. Just ask:

```
You: Is contract 0x1234...abcd safe on Pharos testnet?

Agent: Using pharos-contract-auditor...

       ===========================================
        PHAROS CONTRACT AUDIT SUMMARY
       ===========================================
        Contract:  0x1234...abcd
        Network:   Atlantic Testnet

        RISK FLAGS
        ----------
        Ownership:     ⚠ Owner: 0xOwner... (centralized)
        Pause:         ✓ No pause function
        Mint:          ⚠ mint() function present
        Proxy:         ✓ Not upgradeable
        Concentration: ✓ Owner holds 2.5% of supply

        OVERALL RISK: MEDIUM

        View source: https://testnet.pharosscan.xyz/address/0x1234...#code
       ===========================================
```

---

## Important Disclaimer

This skill provides **automated heuristic analysis only**. It is not a substitute for a professional smart contract audit. Always review verified source code before committing significant funds to any contract.

---

## Network Details

| Network | Chain ID | RPC | Explorer |
|---|---|---|---|
| Atlantic Testnet | 688688 | https://atlantic.dplabs-internal.com | https://testnet.pharosscan.xyz |
| Pacific Mainnet | 1672 | https://rpc.pharos.xyz | https://pharosscan.xyz |

---

## Dependencies

- [Foundry](https://getfoundry.sh) — `cast` for read calls
- [jq](https://stedolan.github.io/jq/) — JSON config parsing
- No private key required

---

## Supported Frameworks

- Claude Code
- Codex CLI
- OpenClaw

---

## License

MIT — Free to use, modify, and redistribute.

---

## Author

Built by **DingiDinigi** for the Pharos Agent Centre Skill Builder Campaign.

GitHub: [https://github.com/DingiDinigi/pharos-contract-auditor](https://github.com/DingiDinigi/pharos-contract-auditor)
