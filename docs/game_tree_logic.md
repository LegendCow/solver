# Bet Sizing Logic: From Spot to Action List

This document describes the complete process of mapping a postflop game tree spot to a finalized list of bet size options. Board texture is irrelevant -- sizing arrays cover all boards, and the solver resolves frequency differences across textures.

---

## 1. Inputs That Define a Spot

Every spot in the tree is fully determined by these variables:

| Input | Description |
|---|---|
| **Pot Type** | SRP, 3BP, or 4BP |
| **Aggressor Position** | Whether OOP or IP was the preflop aggressor |
| **Street** | Flop, Turn, or River |
| **Active Player** | OOP (0) or IP (1) -- who is acting now |
| **Action History** | What happened on previous streets (flat, check-through, raised) |
| **Is Facing Bet** | Whether the active player must respond to an opponent's bet |
| **Pot Size** | Current pot in chips (includes all bets already made) |
| **Effective Stack** | Active player's remaining stack |
| **Current Bet** | The opponent's bet amount that must be matched (0 if opening) |
| **Raises This Street** | How many raises have occurred on the current street |

From pot and stack, we derive:
- **SPR** = min(stacks) / pot
- **SPR Bucket**: LOW (< 2.0), MEDIUM (2.0-6.0), HIGH (> 6.0)

---

## 2. Size Sets by Pot Type

Each pot type has a set of standard bet sizes expressed as percentages of the pot:

| Pot Type | Standard Sizes | Overbet | Typical SPR |
|---|---|---|---|
| **SRP** | 33%, 75%, 125% | 250% | 6-8 |
| **3BP** | 25%, 50%, 75% | -- | 3-5 |
| **4BP** | 10%, 25%, 50% | -- | 1-2 |

Not every size is available in every spot. The action history and player role determine which subset applies (see Section 5).

---

## 3. The Sizing Pipeline

Given a spot, the pipeline produces a final action list:

### Step 1: Determine Base Actions

If facing a bet (`current_bet > 0`):
- Always add **FOLD** and **CALL** (call amount = min(current_bet, effective_stack))

If not facing a bet (`current_bet == 0`):
- Always add **CHECK**

### Step 2: Gate Check -- Can We Bet/Raise?

Betting/raising is allowed only if:
- `raises_this_street < max_raises` (default: 3)
- `effective_stack > current_bet` (have chips beyond what's needed to call)

If not, skip to Step 7 (no bet/raise options added).

### Step 3: Look Up Raw Percentages

Using the spot's context (pot type, street, action history, player role, is_facing_bet), look up the applicable percentage array from the sizing tables in Section 5.

Example: SRP, IP Aggressor, Flop, IP facing OOP check → `[0.33, 0.75, 1.25]`

### Step 4: Convert Percentages to Chip Amounts

**For bets (opening, not facing a bet):**
```
bet_amount = pct * pot
```

**For raises (facing a bet):**
Using the GTO Wizard standard formula:
```
raise_to = current_bet + pct * (current_bet + pot)
```

This ensures an X% raise lays the same pot odds as an X% open bet.

**Enforce minimum raise:** If facing a bet and the calculated raise is less than `2 * current_bet`, force it to `2 * current_bet` (the legal minimum raise in NLHE).

### Step 5: All-In Clipping

For each calculated amount:

1. If `amount >= effective_stack * ALL_IN_THRESHOLD` (65%): **clip to all-in** (replace with full stack)
2. If `amount >= effective_stack` but below threshold: also clips to all-in (can't bet more than you have)
3. If `amount < effective_stack * ALL_IN_THRESHOLD`: **keep as valid bet**

### Step 6: Add All-In Option

All-in is appended as a default option in every spot, UNLESS:
- **SPR is too high (> 8.0)** and no bet on this street yet: all-in is nonsensical (shoving 800% pot makes no strategic sense). Remove it.
- If SPR <= 8.0 or there's already a bet this street: all-in stays.

### Step 7: Deduplicate and Sort

- Remove duplicate chip amounts (keep the first percentage label encountered)
- Sort ascending by chip amount
- Label each action: **BET** (opening), **RAISE** (facing bet), or **ALL_IN** (amount equals stack)

### Step 8: Output Final Action List

The result is a sorted list of actions, e.g.:
```
CHECK, BET 8.75 (B33), BET 19.88 (B75), BET 33.13 (B125), ALL_IN 87.00
```

---

## 4. Raise Formula Deep Dive

The raise formula deserves special attention because it's unintuitive.

### Formula
```
raise_to = current_bet + pct * (current_bet + pot)
```

Where:
- `pot` = current pot (already includes the opponent's bet from this street)
- `current_bet` = the incremental amount the opponent wagered
- `pct` = the raise percentage from the sizing table

### Why This Works

An X% pot-sized raise lays the same pot odds as an X% pot-sized open bet. This makes the percentages consistent whether you're betting or raising.

### Examples

**Example 1:** Pot is 100 (before opponent acts). Opponent bets 50. Now pot = 150, current_bet = 50.

| Raise % | Calculation | Raise To | Bet Multiplier |
|---|---|---|---|
| R50 | 50 + 0.50 * (50 + 150) = 50 + 100 | **150** | 3.0x |
| R100 | 50 + 1.00 * (50 + 150) = 50 + 200 | **250** | 5.0x |

**Example 2:** Pot is 26.50. OOP bets 8.75 (B33). Now pot = 35.25, current_bet = 8.75.

| Raise % | Calculation | Raise To | Bet Multiplier |
|---|---|---|---|
| R50 | 8.75 + 0.50 * (8.75 + 35.25) = 8.75 + 22 | **30.75** | 3.5x |
| R100 | 8.75 + 1.00 * (8.75 + 35.25) = 8.75 + 44 | **52.75** | 6.0x |

### Translation: Pot % to Bet Multiplier

The multiplier depends on the ratio of bet to pot, so it's not a fixed conversion. But roughly:
- R33 ≈ 2.0-2.5x the bet
- R50 ≈ 2.5-3.5x the bet
- R100 ≈ 4.0-6.0x the bet

---

## 5. Complete Sizing Tables by Pot Type

### Terminology
- **Aggressor**: The preflop raiser/3-bettor/4-bettor
- **Defender**: The preflop caller
- **Donk**: Defender bets into the aggressor (always smallest size only)
- **Probe**: Betting after previous street checked through (full standard sizes)
- **Delayed C-bet**: Aggressor bets after previous street checked through (full standard sizes)
- **Barrel**: Aggressor continues betting on a new street (full standard sizes + overbet if SRP)

### Re-Raise Logic

When a player faces a raise (3rd+ action on a street):
- If `SPR < 3.0`: single raise size (R50) or all-in only
- If `SPR >= 3.0`: R50 available, all-in if within threshold
- Max raises per street: 3 (configurable)
- Re-raises beyond the 2nd raise on a street: all-in only (if SPR allows)

---

### POT TYPE 1: SRP, IP Aggressor

*Example: BTN opens, BB calls. SPR ~6-8.*

#### Flop

| Scenario | Player | Role | Bet Options | Raise Options |
|---|---|---|---|---|
| First to act | OOP | Defender | Check, B33 | -- |
| Facing OOP check | IP | Aggressor | Check, B33, B75, B125 | -- |
| Facing OOP donk (B33) | IP | Aggressor | Fold, Call | R100 |
| Facing IP c-bet | OOP | Defender | Fold, Call | XR50, XR100 |
| Facing OOP check-raise | IP | Aggressor | Fold, Call | R50 |

#### Turn (Flat Flop -- bet was called, no raise)

| Scenario | Player | Role | Bet Options | Raise Options |
|---|---|---|---|---|
| First to act (called flop bet) | OOP | Defender | Check, B33 | -- |
| Facing OOP check (was flop bettor) | IP | Aggressor | Check, B33, B75, B125, B250 | -- |
| Facing OOP donk | IP | Aggressor | Fold, Call | R50, R100 |
| Facing IP barrel | OOP | Defender | Fold, Call | XR50, XR100 |
| Re-raise | Either | -- | Fold, Call | Single size or all-in |

#### Turn (Flop Check-Through)

| Scenario | Player | Role | Bet Options | Raise Options |
|---|---|---|---|---|
| First to act (probe) | OOP | Defender | Check, B33, B75, B125 | -- |
| Facing OOP check (delayed c-bet) | IP | Aggressor | Check, B33, B75, B125 | -- |
| Facing OOP probe | IP | Aggressor | Fold, Call | R50, R100 |
| Facing IP delayed c-bet | OOP | Defender | Fold, Call | XR50, XR100 |
| Re-raise | Either | -- | Fold, Call | Single size or all-in |

#### Turn (Flop Was Raised)

*Pot is bloated, SPR significantly lower.*

| Scenario | Player | Role | Bet Options | Raise Options |
|---|---|---|---|---|
| First to act (was raiser) | OOP/IP | Raiser | Check, B33, B75, B125, B250 | -- |
| Facing check (called the raise) | OOP/IP | Caller | Check, B33, B75, B125, B250 | -- |
| Facing bet | Either | -- | Fold, Call | R50, R100 |
| Re-raise | Either | -- | Fold, Call | Single size or all-in |

#### River (SPR-Dependent)

River sizing is driven by remaining SPR after turn action. The standard size set (B33, B75, B125) is filtered:

| SPR Range | Available Open Sizes | Available Raise Sizes | All-In |
|---|---|---|---|
| **> 6.0** | B33, B75, B125 | R50, R100 | No (too deep) |
| **3.0 - 6.0** | B33, B75, B125, B250 | R50, R100 | Yes |
| **2.0 - 3.0** | B33, B75, B125, B250 | R50 | Yes |
| **1.0 - 2.0** | B33, B75 | R50 | Yes (most sizes clip) |
| **< 1.0** | B33 | -- | Yes (almost everything clips) |

River action history context:
- **After turn bet-call**: Last bettor gets barrel sizes (standard + B250 if SPR allows), caller gets donk (B33) or check
- **After turn check-through**: Both get probe/delayed c-bet sizes (full standard)
- **After turn raise**: Both get full standard + B250, but SPR is likely low enough that most clip

---

### POT TYPE 2: SRP, OOP Aggressor

*Example: SB opens, BB calls. SPR ~6-8.*

Same structure as Pot Type 1 with roles swapped:
- **OOP is the aggressor** (c-bet options: B33, B75, B125)
- **IP is the defender** (donk = B33 only)

#### Flop

| Scenario | Player | Role | Bet Options | Raise Options |
|---|---|---|---|---|
| First to act (c-bet) | OOP | Aggressor | Check, B33, B75, B125 | -- |
| Facing OOP c-bet | IP | Defender | Fold, Call | R50, R100 |
| Facing OOP check | IP | Defender | Check, B33 | -- |
| Facing IP donk (B33) | OOP | Aggressor | Fold, Call | R100 |
| Facing IP raise of c-bet | OOP | Aggressor | Fold, Call | R50 |

#### Turn / River

Follow the same branching logic as Pot Type 1 (flat / check-through / raised) with OOP and IP roles swapped. OOP barrels with standard + B250, IP donks with B33 only, etc.

---

### POT TYPE 3: 3BP, IP Aggressor

*Example: BTN 3-bets CO open, CO calls. SPR ~3-5.*

Size set: B25, B50, B75. No overbets.

#### Flop

| Scenario | Player | Role | Bet Options | Raise Options |
|---|---|---|---|---|
| First to act | OOP | Defender | Check, B25 | -- |
| Facing OOP check | IP | Aggressor | Check, B25, B50, B75 | -- |
| Facing OOP donk | IP | Aggressor | Fold, Call | R50 |
| Facing IP c-bet | OOP | Defender | Fold, Call | XR50 |
| Facing OOP XR | IP | Aggressor | Fold, Call | All-in (if SPR allows) |

#### Turn (Flat Flop)

| Scenario | Player | Role | Bet Options | Raise Options |
|---|---|---|---|---|
| First to act | OOP | Defender | Check, B25 | -- |
| Facing OOP check | IP | Aggressor | Check, B25, B50, B75 | -- |
| Facing OOP donk | IP | Aggressor | Fold, Call | R50 |
| Facing IP barrel | OOP | Defender | Fold, Call | XR50 |
| Re-raise | Either | -- | Fold, Call | All-in only |

#### Turn (Flop Check-Through)

| Scenario | Player | Role | Bet Options | Raise Options |
|---|---|---|---|---|
| Probe / Delayed c-bet | Either | -- | Check, B25, B50, B75 | -- |
| Facing bet | Either | -- | Fold, Call | R50 |

#### Turn (Flop Was Raised)

| Scenario | Player | Role | Bet Options | Raise Options |
|---|---|---|---|---|
| Either player first | Either | -- | Check, B25, B50, B75 | -- |
| Facing bet | Either | -- | Fold, Call | R50 or all-in |

#### River

SPR is typically very low after 3BP action. Most sizes clip to all-in.

| SPR Range | Available Sizes | All-In |
|---|---|---|
| **> 3.0** | B25, B50, B75 | Yes |
| **1.5 - 3.0** | B25, B50 | Yes (B75 likely clips) |
| **< 1.5** | B25 | Yes (most clip) |

Raise options: R50 if SPR allows, otherwise all-in only.

---

### POT TYPE 4: 3BP, OOP Aggressor

*Example: BB 3-bets BTN open, BTN calls. SPR ~3-5.*

Same structure as Pot Type 3 with roles swapped:
- **OOP is the 3-bettor** (c-bet options: B25, B50, B75)
- **IP is the defender** (donk = B25 only)

All branching logic mirrors Pot Type 3. OOP tends to size larger to compensate for positional disadvantage, but the available size set is the same.

---

### POT TYPE 5: 4BP, IP Aggressor

*Example: BTN 4-bets BB's 3-bet, BB calls. SPR ~1-2.*

Size set: B10, B25, B50. At this SPR, most bets clip to all-in.

#### Flop

| Scenario | Player | Role | Bet Options | Raise Options |
|---|---|---|---|---|
| First to act | OOP | Defender | Check, B10 | -- |
| Facing OOP check | IP | Aggressor | Check, B10, B25, B50 | -- |
| Facing OOP donk | IP | Aggressor | Fold, Call | All-in |
| Facing IP c-bet | OOP | Defender | Fold, Call | All-in |

#### Turn

SPR is extremely low. Available options depend heavily on remaining stacks.

| SPR Range | Available Sizes | Raise Options |
|---|---|---|
| **> 1.5** | B10, B25 | All-in only |
| **< 1.5** | B10 | All-in only |

#### River

Almost always Check or All-in. If somehow deep enough: B10, B25.

---

### POT TYPE 6: 4BP, OOP Aggressor

*Example: BB 4-bets BTN 3-bet, BTN calls. SPR ~1-2.*

Same as Pot Type 5 with roles swapped:
- **OOP is the 4-bettor** (c-bet: B10, B25, B50)
- **IP is the defender** (donk = B10 only)

SPR is so low that the trees are nearly identical regardless of who is aggressor.

---

## 6. All-In Logic Summary

### When All-In Is Added
- **Default**: All-in is always available as an option
- **Removed when**: SPR > 8.0 and no bet on this street (nonsensical shove into a massive remaining stack)

### Clipping Threshold
- **65% of effective stack**: Any calculated bet at or above this threshold is replaced with all-in
- This prevents near-all-in bets that are strategically equivalent to shoving

### Clipping Walkthrough

Given: effective_stack = 87, pot = 60, all_in_threshold = 0.65

| Raw Bet | vs Threshold (87 * 0.65 = 56.55) | Result |
|---|---|---|
| B33 = 19.80 | Below | Keep: BET 19.80 |
| B75 = 45.00 | Below | Keep: BET 45.00 |
| B125 = 75.00 | Above | Clip: ALL_IN 87.00 |
| B250 = 150.00 | Above | Clip: ALL_IN 87.00 (deduped) |

Final: CHECK, BET 19.80, BET 45.00, ALL_IN 87.00

### Min-Raise Enforcement

When facing a bet, the legal minimum raise is 2x the current bet. If a calculated raise falls below this, it's forced up to the minimum.

---

## 7. SPR-Dependent Rules

SPR dynamically shapes the action tree:

| SPR | Effect |
|---|---|
| **< 1.0** | Almost all bets clip to all-in. Typically only smallest size + all-in survive. |
| **1.0 - 2.0** | Most larger sizes clip. 1-2 bet sizes + all-in. |
| **2.0 - 3.0** | Standard sizes available but overbets clip. Raise options limited. |
| **3.0 - 6.0** | Full standard sizes. Overbets available in SRP. Raises viable. |
| **> 6.0** | All sizes available. All-in removed if > 8.0 SPR and no bet yet. |
| **> 8.0** | All-in removed from opening options (nonsensical). Still available as raise response. |

---

## 8. Edge Cases

### Multiple Raises on One Street
- Max 3 raises per street (configurable)
- 1st raise: full raise options (R50, R100 in SRP; R50 in 3BP/4BP)
- 2nd raise (re-raise): single size (R50) or all-in, depending on SPR
- 3rd raise: all-in only (if stacks allow)
- After max raises reached: only Fold/Call available

### Near-Zero Stack After Call
If calling would leave a player with < 0.1% of pot remaining, treat the call as all-in (no further action possible).

### Identical Clipped Amounts
When multiple raw sizes all clip to the same all-in amount, only one ALL_IN action appears in the final list. The first percentage encountered is kept as the label.

### Raise Into All-In
If the minimum raise (2x current_bet) exceeds or clips to the effective stack, the only raise option is all-in. No intermediate raise sizes exist.
