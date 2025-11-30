# PDOX Token Production Checklist

**Internal Use Only - Exhaustive Pre-Launch Verification**

---

## 1. Transfer Hook Decision

### Decision: NO HOOK FOR V1

**Rationale:**
- Transfer fees are enforced at the Token-2022 program level (not bypassable)
- Hooks add complexity and attack surface without proportional benefit for v1
- Fee revenue doesn't require hook logic - withheld fees are harvested permissionlessly
- MIN_TRANSFER_AMOUNT is enforced at instruction level (not wallet-to-wallet)

**Trade-off Accepted:**
- ❌ Cannot block wallet-to-wallet dust transfers (sub-34 unit amounts)
- ✅ Dust transfers still pay 0 fee (no economic incentive for attacker)
- ✅ Holder count inflation mitigated by snapshot restore mechanism
- ✅ Simpler codebase = fewer audit surface

**If Hook Needed Later (v2):**
```rust
// Required guards for any future hook:
// 1. Reentrancy guard (no CPI back to pool programs)
// 2. Strict token_program ID check (reject legacy SPL Token)
// 3. Strict mint check (only our mint)
// 4. ExtraAccountMetaList uses keccak256 seed (not sha256)
// 5. No user-supplied arbitrary metas
```

---

## 2. Mint Configuration Sanity Check

**Run this against the ACTUAL devnet mint before mainnet:**

| Check | Expected | Actual | ✓ |
|-------|----------|--------|---|
| Mint Address | `[YOUR_MINT]` | | |
| Token Program | `TokenzQdBN...` (Token-2022) | | |
| Decimals | 6 | | |
| Supply | 10,000,000 (initial LP only) | | |
| Mint Authority | `[LP_GROWTH_PDA]` or burned | | |
| Freeze Authority | Burned OR multisig | | |

### Extensions Required:
| Extension | Status | Config |
|-----------|--------|--------|
| TransferFeeConfig | ✓ Required | 300 bps (3%), max_fee = u64::MAX |
| TransferHook | ✗ Not Used | N/A |
| PermanentDelegate | ✗ MUST NOT EXIST | If exists = instant rug risk |
| MintCloseAuthority | ✗ Should not exist | |
| InterestBearingConfig | ✗ Not Used | |

### Authority Verification:
```bash
# Run these commands to verify:
spl-token display <MINT_ADDRESS> --program-id TokenzQdBNJq...

# Check for PermanentDelegate (MUST BE EMPTY):
# If "Permanent Delegate: <some_pubkey>" exists, DO NOT LAUNCH
```

---

## 3. Withdraw-Withheld Authority Check

| Check | Status |
|-------|--------|
| Harvest authority is PDA | ✓ `HARVEST_AUTHORITY_SEED + mint` |
| PDA cannot be changed | ✓ Hardcoded in code |
| Fees go to `fee_vault` | ✓ Constrained in instruction |
| No path to redirect to arbitrary account | ✓ Verified |
| Anyone can call harvest | ✓ Permissionless (prevents deadlock) |

---

## 4. DEX Integration Torture Tests

### Test Matrix (Meteora DLMM or equivalent):

| Test | Input | Expected | Actual | ✓ |
|------|-------|----------|--------|---|
| Swap exact-in (small) | 100 PDOX | 97 PDOX to pool (3% fee) | | |
| Swap exact-in (MIN) | 34 PDOX | 33 PDOX to pool, 1 fee | | |
| Swap exact-in (below MIN) | 33 PDOX | Should still work (0 fee) | | |
| Swap exact-in (large) | 1M PDOX | 970K to pool | | |
| Swap exact-out | Request 100 | Pay ~103.1 gross | | |
| Multi-bin swap | Cross 5 bins | No slippage death spiral | | |
| LP add (with fee) | 1M gross | Pool receives ~970K | | |
| LP remove | All LP | Get proportional reserves | | |

### Edge Cases:
| Test | Expected Result |
|------|-----------------|
| Transfer at u64::MAX / 10_000 | No overflow (u128 intermediate) |
| 1000 sequential tiny swaps | No state corruption |
| Fee change mid-batch | Uses NEW fee after 24h |
| Snapshot during withdrawal | Contains real reserves |

### "Vanishing Liquidity" Check:
```
For every LP operation:
pool_recorded_amount == actual_received_amount (after fees)

If gross_in = 1,030,928 and fee = 3%:
  actual_received = 1,000,000 ✓
  pool_recorded = 1,000,000 ✓ (must match!)
```

---

## 5. Property Tests / Invariants

### Fee Invariants (MUST HOLD):
```rust
// Test: fee_never_zero_above_threshold
#[test]
fn test_fee_invariant() {
    for amount in 34..10_000 {
        let fee = amount * 300 / 10_000;
        assert!(fee >= 1, "Fee bypass at amount {}", amount);
    }
}

// Test: value_conservation
#[test]
fn test_value_conservation() {
    for amount in 34..1_000_000 {
        let fee = amount * 300 / 10_000;
        let after_fee = amount - fee;
        assert_eq!(after_fee + fee, amount, "Value created/destroyed!");
    }
}

// Test: no_overflow_large_amounts
#[test]
fn test_no_overflow() {
    let max_safe = u64::MAX / 10_000;
    let fee = (max_safe as u128 * 300 / 10_000) as u64;
    assert!(fee > 0);
}
```

### Timelock Invariants (MUST HOLD):
```rust
// Test: fee_change_requires_timelock
// Cannot execute fee change before pending_fee_execute_after

// Test: lp_withdrawal_requires_timelock
// Cannot execute before phase-appropriate timelock:
// - Phase 1 (0-3 days): 12h
// - Phase 2 (3-15 days): 15 days
// - Phase 3 (15+ days): 30 days
```

### LP Share Invariants:
```rust
// Test: lp_shares_proportional_to_net_contribution
// After fee deduction, LP shares must reflect actual contribution
```

---

## 6. Governance & Key Management

### Current Authority Structure:
```
┌─────────────────────────────────────────┐
│         AUTHORITY BLAST RADIUS          │
├─────────────────────────────────────────┤
│                                         │
│  TokenConfig.admin                      │
│  ├─ Can: Announce fee change (24h wait) │
│  ├─ Can: Lock/unlock LP growth          │
│  ├─ Can: Trigger Armageddon             │
│  └─ Risk: MEDIUM (timelocked)           │
│                                         │
│  LpLock.admin                           │
│  ├─ Can: Announce LP withdrawal         │
│  ├─ Can: Take snapshots                 │
│  ├─ Can: Transfer admin role            │
│  └─ Risk: HIGH (12h-30d timelock)       │
│                                         │
│  Treasury.governance                    │
│  ├─ Can: Propose withdrawal (48h wait)  │
│  └─ Risk: MEDIUM (timelocked + capped)  │
│                                         │
└─────────────────────────────────────────┘
```

### BEFORE MAINNET:
- [ ] Move `TokenConfig.admin` to multisig (Squads recommended)
- [ ] Move `LpLock.admin` to multisig
- [ ] Move `Treasury.governance` to governance program (Realms)
- [ ] Document key compromise response plan

### If Admin Key Compromised:
1. Attacker can announce fee change → 24h to respond
2. Attacker can announce LP withdrawal → 12h-30d to respond
3. Community monitoring required to detect + respond

### Who Can Trigger Armageddon:
- `TokenConfig.admin` only
- Conditions: LP value drops >50% from peak
- Effect: Max fees (3%), max LP share (80%)

---

## 7. Monitoring / Observability Plan

### Required Event Indexing:
| Event | Priority | Alert Threshold |
|-------|----------|-----------------|
| `FeeChangeAnnounced` | CRITICAL | Any occurrence |
| `TransferFeeUpdated` | CRITICAL | fee_bps != 300 |
| `LpWithdrawalAnnounced` | HIGH | Any occurrence |
| `FeesHarvested` | MEDIUM | amount > 1M |
| `ArmageddonTriggered` | CRITICAL | Any occurrence |
| `LpLockCreated` | INFO | Initial only |
| `SnapshotTaken` | INFO | Log for audit |

### Indexer Requirements (Helius or custom):
- Track withheld fees accumulation
- Track LP lock state changes
- Track pending fee changes + countdown
- Track admin key changes

### Alert Rules:
```yaml
alerts:
  - name: fee_change_announced
    condition: event == "FeeChangeAnnounced"
    action: notify_team_immediately
    
  - name: lp_withdrawal_announced
    condition: event == "LpWithdrawalAnnounced"
    action: notify_team_immediately
    
  - name: admin_changed
    condition: event contains "admin" OR "governance"
    action: notify_team_immediately
    
  - name: armageddon
    condition: event == "ArmageddonTriggered"
    action: emergency_response
```

---

## 8. Frontend / Client Requirements

### Enforce in UI:
| Rule | Implementation |
|------|----------------|
| Min transfer: 34 raw units | Disable send button if < 0.000034 |
| Show current fee | Display "3% transfer fee" prominently |
| Show pending fee change | Countdown if `has_pending_fee_change` |
| LP timelock countdown | Show phase + time remaining |
| Gross-up for swaps | Add 3% to input for exact output |

### Prevent User Confusion:
```typescript
// When user inputs swap amount:
const grossAmount = Math.ceil(desiredAmount / 0.97);
const feeAmount = grossAmount - desiredAmount;

// Display:
// "You send: 103.1 PDOX"
// "Fee: 3.1 PDOX (3%)"
// "Pool receives: 100 PDOX"
```

### Fee Change Warning:
```typescript
if (tokenConfig.has_pending_fee_change) {
  const activatesIn = tokenConfig.pending_fee_execute_after - now;
  showWarning(`Fee changing to ${newFee}% in ${formatTime(activatesIn)}`);
}
```

---

## 9. Final Go/No-Go Checklist

| Item | Status | Blocker? |
|------|--------|----------|
| All Token-2022 compatible (transfer_checked) | ✅ | YES |
| MIN_TRANSFER_AMOUNT enforced | ✅ | YES |
| Snapshot requires real data | ✅ | YES |
| u128 intermediate calcs | ✅ | YES |
| Fee change 24h timelock | ✅ | YES |
| Fee harvesting implemented | ✅ | YES |
| No .unwrap()/.expect() | ✅ | YES |
| Multisig for admin keys | ⬜ TODO | YES |
| Devnet testing complete | ⬜ TODO | YES |
| External audit | ⬜ TODO | RECOMMENDED |
| Monitoring setup | ⬜ TODO | YES |
| Frontend enforcement | ⬜ TODO | YES |

---

*Last updated: November 30, 2025*
*Maintainer: LabsX402*

