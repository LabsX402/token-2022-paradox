# Armageddon Mode

Emergency circuit breaker for the LP pool.

---

## wtf is it

When shit hits the fan and LP value dumps >50% from its peak, admin can flip the Armageddon switch. It's the "oh fuck" button.

---

## what happens when triggered

| Setting | Normal | Armageddon |
|---------|--------|------------|
| Transfer fee | 3% | **MAX (configurable)** |
| LP share of fees | normal % | **80%** |
| Goal | balanced growth | emergency LP recovery |

Basically: jacks fees up and funnels most of it into the LP to stop the bleeding.

---

## who can trigger it

Only `TokenConfig.admin` - and only when:
- LP value dropped >50% from the all-time high snapshot
- That's it. Can't just yolo trigger it.

---

## why it exists

DEXs can get rekt by:
- Coordinated dumps
- Flash loan attacks
- Market panic

Armageddon buys time. Higher fees = slower bleed + faster LP recovery. It's not a fix, it's a tourniquet.

---

## recovery

Once LP recovers above threshold, admin can disable Armageddon and return to normal fee structure.

---

## risks

- Admin could trigger it maliciously (that's why multisig before mainnet)
- High fees might kill volume (acceptable tradeoff during emergency)
- Doesn't prevent the dump, just limits damage

---

*tl;dr: panic button that maxes fees to save the LP when everything's on fire*

