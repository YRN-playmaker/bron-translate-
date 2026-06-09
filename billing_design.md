# Billing Engine — Dual-Pool Quota Design

> Pseudocode + design notes. Concrete thresholds, prices, and tier rules are
> intentionally omitted — this document shows the *system design*, not a
> runnable implementation.

## Problem

A usage-metered translation gateway needs to bill premium users for real-time
transcription time. A naive single balance ("you have N minutes") fails two
real-world requirements:

1. **Subscription minutes should expire monthly** (use-it-or-lose-it), so they
   can't be hoarded indefinitely.
2. **Top-up minutes the user paid for outright should never expire.**

Mixing both into one number makes it impossible to apply different expiry rules.
The solution is a **dual-pool** model.

## Model

Each premium account holds two independent balances:

```
account:
    permanent_balance    # paid outright, never expires
    monthly_balance      # subscription grant, resets + expires every cycle
    monthly_expiry       # timestamp; after this, monthly_balance is void
```

### Spending order

The core rule: **spend the perishable pool first.** This maximizes the value the
user gets from their subscription before touching minutes that never expire.

```
function deduct(account, amount):
    # 1. drain monthly (perishable) first
    if account.monthly_balance > 0:
        take = min(amount, account.monthly_balance)
        account.monthly_balance -= take
        amount -= take

    # 2. only then touch permanent (non-expiring)
    if amount > 0:
        account.permanent_balance -= amount

    clamp_both_to_zero(account)
```

### Expiry check (lazy evaluation)

Rather than running a scheduled job to reset every account, expiry is evaluated
**on access** — cheaper and stateless:

```
function on_access(account, now):
    if account.monthly_expiry > 0 and now > account.monthly_expiry:
        account.monthly_balance = 0
        account.monthly_expiry  = 0
    # permanent_balance is untouched, always
```

### Time-based metering

Real-time sessions bill by elapsed wall-clock time, not by request count. The
elapsed time since the last update is converted to minutes and run through
`deduct()`:

```
function meter_session(account, now):
    elapsed = now - account.last_update_time
    elapsed = min(elapsed, SESSION_CAP)      # guard against clock jumps / idle gaps
    minutes = elapsed / 60 * TIER_MULTIPLIER # multiplier omitted (per-tier)
    deduct(account, minutes)
    account.last_update_time = now
```

> `SESSION_CAP` and `TIER_MULTIPLIER` are tuned per tier and deliberately not
> shown.

## Provisioning (issuing / renewing)

When a user purchases, the two pools update with **different semantics**:

```
function provision(account, permanent_add, monthly_grant):
    account.permanent_balance += permanent_add      # ACCUMULATES
    account.monthly_balance    = monthly_grant       # OVERWRITES (fresh cycle)
    account.monthly_expiry     = now + CYCLE_LENGTH
```

- Permanent minutes **stack** across purchases.
- Monthly minutes **reset** to the new grant (you don't accumulate stale
  subscription minutes).

### Tier transitions

Switching a user between tiers (e.g. standard → premium) reissues their access
credential and zeroes stale balances, preventing quota from leaking across tiers
with incompatible billing rules:

```
function on_tier_change(account, new_tier):
    if account.credential_prefix != new_tier.prefix:
        account.credential       = new_credential(new_tier)
        account.permanent_balance = 0
        account.monthly_balance   = 0
        account.monthly_expiry    = 0
```

## Why this design

| Decision | Rationale |
|---|---|
| Two pools instead of one | Different expiry rules can't coexist in one number |
| Spend perishable first | Maximizes subscriber value before consuming non-expiring credit |
| Lazy expiry on access | No cron job, no per-account scheduler — stateless and cheap on constrained hardware |
| Multiplier per tier | One engine bills different tiers at different rates without branching logic |
| Reissue credential on tier change | Hard isolation between billing regimes; no cross-tier quota leakage |

## Notes

- All balance writes are clamped to `>= 0` before persistence.
- Reads and writes happen in a single transaction per session tick to avoid
  race conditions under concurrent connections from the same account.
