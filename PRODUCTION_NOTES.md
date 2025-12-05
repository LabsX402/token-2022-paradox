# PDOX Production Notes

> Public version. Full checklist internal.

---

## Transfer Hook

**Decision:** No hook for v1

Why:
- Fees enforced at Token-2022 level already
- Less attack surface
- Harvesting is permissionless anyway

---

## Authority Setup

| Role | Risk | Notes |
|------|------|-------|
| TokenConfig admin | Medium | 24h timelock on changes |
| LpLock admin | High | Progressive timelock |
| Treasury gov | Medium | 48h + caps |

Before mainnet: [redacted]

---

## Monitoring

Track these events:
- FeeChangeAnnounced
- LpWithdrawalAnnounced  
- ArmageddonTriggered

Alert config: [redacted]

---

## Frontend Notes

- Show 3% fee prominently
- Enforce min 34 units in UI
- Show pending fee changes with countdown
- Gross-up swap inputs (+3%)

---

## Launch Status

| Item | Done? |
|------|-------|
| Token-2022 compat | ✅ |
| Min transfer enforced | ✅ |
| u128 math | ✅ |
| Fee timelock | ✅ |
| Devnet tested | 🔄 |
| [redacted] | ⬜ |
| [redacted] | ⬜ |

---

*Nov 2025*

