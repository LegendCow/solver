# BTN vs BB SRP Sizing Pipeline

This document defines a tightened action-selection spec for `BTN vs BB` single-raised pots with:

- starting pot = `5.5`
- starting effective stacks = `97.5`
- preflop aggressor = `IP` (`BTN`)

The purpose of this file is to answer one question:

> Given a BTN vs BB postflop spot, what actions and sizes are allowed?

This is a size-availability spec only. It does not try to solve frequencies or board-class strategy.

---

## 1. Spot Inputs

To return the correct menu, the spot needs these fields:

- `street`: `FLOP`, `TURN`, `RIVER`
- `active_player`: `OOP` or `IP`
- `pot`
- `stacks`
- `current_bet`
- `raises_this_street`
- `flop_history`: `FLAT`, `CHECK_THRU`, `RAISED`
- `turn_history`: `FLAT`, `CHECK_THRU`, `RAISED`
- `last_street_bettor`: `OOP`, `IP`, or `None`

Derived values:

- `is_facing_bet = current_bet > 0`
- `effective_stack = stacks[active_player]`
- `spr = min(stacks) / pot`
- `is_aggressor = active_player == IP`
- `is_defender = active_player == OOP`

For BTN vs BB:

- `IP = BTN = aggressor`
- `OOP = BB = defender`

Root flop reference:

- `pot = 5.5`
- `stacks = (97.5, 97.5)`
- `spr = 17.73`

---

## 2. Base Size Vocabulary

### Flop opening sizes

- `B33`
- `B50`
- `B75`
- `B125`

### Turn opening sizes

Turn removes `B50` and uses the large overbet instead:

- `B33`
- `B75`
- `B125`
- `B250`

### River opening sizes

River removes `B125` and replaces it with `B150`:

- `B33`
- `B75`
- `B150`
- `B250`

### Raise sizes

- `R50`
- `R100`

### Special role rule

OOP/BB donk size:

- `B33` only

---

## 3. Universal Pipeline

### Step 1: Add mandatory non-bet actions

If opening:

- add `CHECK`

If facing a bet:

- add `FOLD`
- add `CALL`

### Step 2: Gate whether betting or raising is allowed

Do not add bet or raise options if:

- `raises_this_street >= 3`, or
- `effective_stack <= current_bet`

### Step 3: Select a raw size menu

The raw menu depends on:

- street
- role
- whether facing a bet
- prior street history
- last street bettor
- current street raise layer
- current river SPR tier

### Step 4: Convert labels to chip amounts

Open bet:

```text
bet_amount = pct * pot
```

Raise:

```text
raise_to = current_bet + pct * (current_bet + pot)
```

Enforce legal min-raise:

```text
raise_to = max(raise_to, 2 * current_bet)
```

### Step 5: Clip near-all-in sizes

If a computed amount is at least `65%` of the effective stack, clip it to all-in:

```text
if amount >= 0.65 * effective_stack:
    amount = effective_stack
```

### Step 6: Add discretionary all-in option

There are two separate all-in mechanisms:

- clipped all-in: created when a normal size reaches the `65%` stack threshold
- discretionary all-in: added as its own extra option

The discretionary all-in should be suppressed aggressively.

Use this rule:

```text
only add a discretionary all-in if
effective_stack <= 5.0 * pot
```

So:

- clipped all-ins stay
- discretionary all-in is only added when the shove is at most `500%` pot

### Step 7: Deduplicate and sort

- remove duplicate chip amounts
- sort ascending
- label final actions as `BET`, `RAISE`, or `ALL_IN`

---

## 4. Flop Rules

### OOP/BB first to act

- `CHECK`
- `B33`

### IP/BTN after OOP checks

- `CHECK`
- `B33`
- `B50`
- `B75`
- `B125`

### IP/BTN facing BB flop donk

- `FOLD`
- `CALL`
- `R100`
- `ALL_IN` only if clipped or if shove is `<= 500%` pot

### OOP/BB facing BTN flop c-bet

- `FOLD`
- `CALL`
- `R50`
- `R100`
- `ALL_IN` only if clipped or if shove is `<= 500%` pot

### IP/BTN facing flop check-raise

- `FOLD`
- `CALL`
- `R50`
- `ALL_IN` only if clipped or if shove is `<= 500%` pot

### OOP/BB facing flop 3-bet

- `FOLD`
- `CALL`
- `ALL_IN` only

---

## 5. Turn Rules

Turn uses `B33`, `B75`, `B125`, `B250`.

`B50` does not exist on turn.

### 5.1 Turn after flop `FLAT`

This means flop bet-call.

If the active player was the flop bettor:

- `CHECK`
- `B33`
- `B75`
- `B125`
- `B250`

If the active player called the flop bet and now leads:

- `CHECK`
- `B33`

If facing a turn bet at the first raise layer:

- `FOLD`
- `CALL`
- `R50`
- `R100`
- `ALL_IN` only if clipped or if shove is `<= 500%` pot

If facing a turn raise after betting:

- `FOLD`
- `CALL`
- `R50`
- `ALL_IN`

If facing a turn 3-bet:

- `FOLD`
- `CALL`
- `ALL_IN` only

### 5.2 Turn after flop `CHECK_THRU`

Both players get the full non-donk turn menu:

- `CHECK`
- `B33`
- `B75`
- `B125`
- `B250`

If facing a turn bet at the first raise layer:

- `FOLD`
- `CALL`
- `R50`
- `R100`
- `ALL_IN` only if clipped or if shove is `<= 500%` pot

If facing a turn raise after betting:

- `FOLD`
- `CALL`
- `R50`
- `ALL_IN`

If facing a turn 3-bet:

- `FOLD`
- `CALL`
- `ALL_IN` only

### 5.3 Turn after flop `RAISED`

After a raised flop, both players start from the full turn menu:

- `CHECK`
- `B33`
- `B75`
- `B125`
- `B250`

If facing a first raise:

- `FOLD`
- `CALL`
- `R50`
- `R100`
- `ALL_IN` only if clipped or if shove is `<= 500%` pot

If facing a re-raise:

- `FOLD`
- `CALL`
- `R50`
- `ALL_IN`

If facing a third raise:

- `FOLD`
- `CALL`
- `ALL_IN` only

---

## 6. River Rules

River uses `B33`, `B75`, `B150`, `B250`.

River never uses `B125`.

### 6.1 OOP/BB river rule

OOP only gets a wide river betting menu if OOP was not the turn caller.

If OOP called a turn bet:

- OOP opening menu is only:
  - `CHECK`
  - `B33`
  - `ALL_IN` may still appear if clipped or if shove is `<= 500%` pot

If OOP did not call a turn bet, OOP gets the full river menu:

- `CHECK`
- `B33`
- `B75`
- `B150`
- `B250`
- `ALL_IN` subject to SPR filtering, clipping, and the `<= 500%` pot discretionary-jam rule

That means OOP gets the full river menu in these branches:

- turn checked through
- OOP bet turn and got called
- turn was raised

### 6.2 IP/BTN river rule

IP can always use the aggressive river menu, but never `B33`.

IP river opening menu:

- `CHECK`
- `B75`
- `B150`
- `B250`
- `ALL_IN` subject to SPR filtering, clipping, and the `<= 500%` pot discretionary-jam rule

### 6.3 River raise rules

Raise options on river are still SPR-sensitive.

If `SPR > 3.0`:

- `R50`
- `R100`

If `2.0 <= SPR <= 3.0`:

- `R50`

If `SPR < 2.0`:

- no non-all-in raise sizes
- `ALL_IN` only

### 6.4 River SPR filtering for opens

This section filters the raw river menu before chip conversion and clipping.

#### `SPR > 6.0`

OOP full-menu spots:

- `B33`
- `B75`
- `B150`

IP:

- `B75`
- `B150`

No default opening jam.

#### `3.0 < SPR <= 6.0`

OOP full-menu spots:

- `B33`
- `B75`
- `B150`
- `B250`

IP:

- `B75`
- `B150`
- `B250`

Default opening jam allowed.

#### `2.0 < SPR <= 3.0`

OOP full-menu spots:

- `B33`
- `B75`
- `B150`
- `B250`

IP:

- `B75`
- `B150`
- `B250`

Default opening jam allowed.

#### `1.0 < SPR <= 2.0`

OOP full-menu spots:

- `B33`
- `B75`

IP:

- `B75`
- `B150`

Larger sizes usually clip anyway.

#### `SPR <= 1.0`

OOP full-menu spots:

- `B33`

IP:

- `B75`

At this depth, most larger options collapse into all-in.

---

## 7. Compact Implementation Rules

If you want the shortest implementation summary, use this:

1. Add `CHECK` or `FOLD/CALL`.
2. Stop if raises are capped or the stack cannot raise.
3. Use flop menu `33/50/75/125` for BTN c-bets.
4. Use turn menu `33/75/125/250`.
5. Use river menu `33/75/150/250`, but:
   OOP after calling turn may only lead `33`.
   IP never gets `33`.
6. Convert labels to chips.
7. Enforce min-raise.
8. Clip any size at or above `65%` stack to all-in.
9. Add discretionary all-in only if shove is `<= 500%` pot.
10. Deduplicate and sort.

---

## 8. BTN vs BB SRP Spot Tables

These tables are label-level menus before chip conversion and clipping.

`AI` below means the all-in option may exist after clipping and the discretionary `<= 500%` pot jam rule are applied.

Coverage count:

- `45` non-terminal BTN vs BB spot classes are listed below
- `29` of those are raise-capable response rows
- there are `4` distinct raise menus in the entire spec:
  - `R100`
  - `R50, R100`
  - `R50`
  - `AI only`

### 8.1 Flop

| # | Spot | Actions |
|---|---|---|
| 1 | OOP first to act | `CHECK`, `B33` |
| 2 | IP after OOP check | `CHECK`, `B33`, `B50`, `B75`, `B125` |
| 3 | IP facing OOP donk | `FOLD`, `CALL`, `R100` |
| 4 | OOP facing IP c-bet | `FOLD`, `CALL`, `R50`, `R100` |
| 5 | IP facing OOP check-raise | `FOLD`, `CALL`, `R50`, `AI` |
| 6 | OOP facing IP 3-bet | `FOLD`, `CALL`, `AI` |

### 8.2 Turn After Flop `FLAT`

| # | Spot | Actions |
|---|---|---|
| 7 | OOP first to act after calling flop bet | `CHECK`, `B33` |
| 8 | IP after OOP check as flop bettor | `CHECK`, `B33`, `B75`, `B125`, `B250` |
| 9 | IP facing OOP donk | `FOLD`, `CALL`, `R50`, `R100` |
| 10 | OOP facing IP barrel | `FOLD`, `CALL`, `R50`, `R100` |
| 11 | IP facing OOP raise | `FOLD`, `CALL`, `R50`, `AI` |
| 12 | OOP facing IP 3-bet | `FOLD`, `CALL`, `AI` |

### 8.3 Turn After Flop `CHECK_THRU`

| # | Spot | Actions |
|---|---|---|
| 13 | OOP probe option | `CHECK`, `B33`, `B75`, `B125`, `B250` |
| 14 | IP delayed c-bet option | `CHECK`, `B33`, `B75`, `B125`, `B250` |
| 15 | IP facing OOP probe | `FOLD`, `CALL`, `R50`, `R100` |
| 16 | OOP facing IP delayed c-bet | `FOLD`, `CALL`, `R50`, `R100` |
| 17 | IP facing OOP raise | `FOLD`, `CALL`, `R50`, `AI` |
| 18 | OOP facing IP 3-bet | `FOLD`, `CALL`, `AI` |

### 8.4 Turn After Flop `RAISED`

| # | Spot | Actions |
|---|---|---|
| 19 | First player to act | `CHECK`, `B33`, `B75`, `B125`, `B250` |
| 20 | Opponent after check | `CHECK`, `B33`, `B75`, `B125`, `B250` |
| 21 | Facing turn bet | `FOLD`, `CALL`, `R50`, `R100`, `AI` |
| 22 | Facing turn raise | `FOLD`, `CALL`, `R50`, `AI` |
| 23 | Facing turn 3-bet | `FOLD`, `CALL`, `AI` |

### 8.5 River After Turn `FLAT` And IP Was Turn Bettor

This is the branch where OOP called the turn bet.

| # | Spot | Actions |
|---|---|---|
| 24 | OOP first to act as turn caller | `CHECK`, `B33`, `AI` |
| 25 | IP after OOP check | `CHECK`, `B75`, `B150`, `B250`, `AI` |
| 26 | IP facing OOP lead | `FOLD`, `CALL`, river raise menu by SPR, `AI` |
| 27 | OOP facing IP river bet | `FOLD`, `CALL`, river raise menu by SPR, `AI` |
| 28 | IP facing OOP river raise | `FOLD`, `CALL`, `R50` or `AI` by SPR |
| 29 | OOP facing IP river 3-bet | `FOLD`, `CALL`, `AI` |

### 8.6 River After Turn `FLAT` And OOP Was Turn Bettor

This is the branch where IP called the turn bet.

| # | Spot | Actions |
|---|---|---|
| 30 | OOP first to act as prior turn bettor | `CHECK`, `B33`, `B75`, `B150`, `B250`, `AI` |
| 31 | IP after OOP check as turn caller | `CHECK`, `B75`, `B150`, `B250`, `AI` |
| 32 | IP facing OOP river bet | `FOLD`, `CALL`, river raise menu by SPR, `AI` |
| 33 | OOP facing IP river raise | `FOLD`, `CALL`, `R50` or `AI` by SPR |
| 34 | IP facing OOP river 3-bet | `FOLD`, `CALL`, `AI` |

### 8.7 River After Turn `CHECK_THRU`

| # | Spot | Actions |
|---|---|---|
| 35 | OOP first to act | `CHECK`, `B33`, `B75`, `B150`, `B250`, `AI` |
| 36 | IP after OOP check | `CHECK`, `B75`, `B150`, `B250`, `AI` |
| 37 | IP facing OOP river bet | `FOLD`, `CALL`, river raise menu by SPR, `AI` |
| 38 | OOP facing IP river bet | `FOLD`, `CALL`, river raise menu by SPR, `AI` |
| 39 | IP facing OOP river raise | `FOLD`, `CALL`, `R50` or `AI` by SPR |
| 40 | OOP facing IP river 3-bet | `FOLD`, `CALL`, `AI` |

### 8.8 River After Turn `RAISED`

| # | Spot | Actions |
|---|---|---|
| 41 | First player to act | `CHECK`, `B33`, `B75`, `B150`, `B250`, `AI` |
| 42 | Opponent after check | `CHECK`, `B75`, `B150`, `B250`, `AI` |
| 43 | Facing river bet | `FOLD`, `CALL`, river raise menu by SPR, `AI` |
| 44 | Facing river raise | `FOLD`, `CALL`, `R50` or `AI` by SPR |
| 45 | Facing river 3-bet | `FOLD`, `CALL`, `AI` |

---

## 9. River Raise Menu Reference

When a table row says `river raise menu by SPR`, use:

| SPR | Raise Menu |
|---|---|
| `> 3.0` | `R50`, `R100`, `AI` |
| `2.0 - 3.0` | `R50`, `AI` |
| `< 2.0` | `AI` only |

When a table row says `R50 or AI by SPR`, use:

| SPR | Re-raise Menu |
|---|---|
| `> 2.0` | `R50`, `AI` |
| `<= 2.0` | `AI` only |

---

## 10. Implementation Note

A correct BTN vs BB SRP generator cannot be keyed only by:

- street
- SPR bucket
- whether facing a bet

That state is too coarse.

To match this spec, the engine must also know:

- aggressor versus defender role
- flop history
- turn history
- last street bettor
- current street raise layer
- river SPR tier

Without those fields, turn and river menus will be wrong in multiple branches.
