# octra program bug registry

octra program bug registry is a list of common bug patterns to check when auditing octra / aml programs.

this registry is about program logic, templates, and examples.

it is not a claim that aml itself is broken.

## bug index

| id | bug pattern | where to look | main risk | severity |
|---|---|---|---|---|
| [opbr-000](#opbr-000-signed-amount-in-money-logic) | signed amount in money logic | transfers, withdraws, swaps, mint/burn | negative amount can mint value or corrupt balances | critical |
| [opbr-001](#opbr-001-unsafe-signed-delta-accounting) | unsafe signed delta accounting | pnl, funding, rebalance, slash/reward logic | signed delta can create negative state or fake credit | critical / high |
| [opbr-002](#opbr-002-negative-allowance) | negative allowance | grant, approve, pull, transfer_from | allowance can increase or funds can move incorrectly | critical |
| [opbr-003](#opbr-003-negative-withdraw) | negative withdraw | withdraw, unstake, redeem, remove_liquidity | fake deposits, reversed accounting, vault drain | critical |
| [opbr-004](#opbr-004-unchecked-constructor-parameters) | unchecked constructor parameters | constructor, init, deploy params | program starts with broken supply, reserves, fees, owner, or threshold | critical / high |
| [opbr-005](#opbr-005-broken-supply-invariant) | broken supply invariant | transfer, mint, burn, bridge mint/burn | total_supply stops matching balances | critical |
| [opbr-006](#opbr-006-broken-vault-accounting) | broken vault accounting | deposit, withdraw, stake, redeem, harvest | vault insolvency, wrong shares, stuck funds | critical / high |
| [opbr-007](#opbr-007-missing-or-wrong-auth) | missing or wrong auth | mint, set_owner, set_oracle, pause, execute | anyone can run admin logic | critical |
| [opbr-008](#opbr-008-unsafe-oracle--price-feed-usage) | unsafe oracle / price feed usage | mint, redeem, borrow, liquidate, swap | stale or bad price breaks accounting | critical / high |
| [opbr-009](#opbr-009-checks-effects-interactions-violation) | checks-effects-interactions violation | withdraw, claim, execute, swap, redeem | state updated too late, double action or stale state | critical / high |
| [opbr-010](#opbr-010-replay--double-execution) | replay / double execution | claims, bridge messages, signatures, proposals | same action can be used more than once | critical |
| [opbr-011](#opbr-011-unchecked-call-or-transfer-result) | unchecked call or transfer result | call(...), transfer(...), execute, swap, bridge_call | program continues after failed external action | critical / high |
| [opbr-012](#opbr-012-reentrancy--missing-nonreentrant-guard) | reentrancy / missing nonreentrant guard | withdraw, claim, redeem, execute, payout | callback can repeat an action before the first call finishes | critical / high |
| [opbr-014](#opbr-014-missing-exists-flag--default-map-state) | missing exists flag / default map state | proposals, claims, orders, positions, messages | unknown id can pass checks using default map values | critical / high |
| [opbr-015](#opbr-015-broken-accounting--economic-invariant) | broken accounting / economic invariant | tokens, vaults, AMMs, bridges, governance | checks pass but core accounting rule becomes false | critical / high |
| [opbr-016](#opbr-016-rounding--zero-shares) | rounding / zero shares | vault deposits, redeems, swaps, share math | deposit or swap is accepted while output rounds to zero | high / medium |
| [opbr-017](#opbr-017-bad-id-bounds--negative-id) | bad id bounds / negative id | proposals, claims, models, positions, orders | negative id can pass upper-bound checks | high / medium |
| [opbr-018](#opbr-018-duplicate-signer--bad-threshold) | duplicate signer / bad threshold | multisigs, governance, committees, bridges | quorum can be bypassed or made impossible | critical / high |
| [opbr-019](#opbr-019-pause--emergency-bypass) | pause / emergency bypass | pause, emergency withdraw, admin recovery | paused systems can still move funds through alternate paths | critical / high |
| [opbr-020](#opbr-020-unbounded-loops--resource-dos) | unbounded loops / resource DoS | batch actions, rewards, owners, proposals | user-controlled loops can make calls too expensive or impossible | high / medium |

## scope

this registry covers bugs in:

- token programs
- vault programs
- amms
- multisigs
- bridges
- staking programs
- official or unofficial templates
- copied example code

## entry format

each bug entry should include:

- id
- name
- where to look
- why it is dangerous
- bad example
- fixed example
- audit check

---

# opbr-000: signed amount in money logic

## summary

money values should not be negative.

prefer `u128` or another unsigned integer type for asset quantities and ids where the target compiler/runtime supports it

use signed types only for explicit deltas, changes, or values that are allowed to go below zero

if a program uses `int` for amounts, balances, reserves, deposits, supply, or allowances, the program must check that the value is non-negative or positive

do not use a `u128` annotation as a replacement for checks around zero amounts, insufficient balances, missing authorization, replay, arithmetic bounds, or accounting invariants

## where to look

check functions like:

- `transfer`
- `withdraw`
- `pull`
- `grant`
- `mint`
- `burn`
- `swap`
- `add_liquidity`
- `remove_liquidity`

look for parameters like:

```aml
amount: int
amt: int
value: int
supply: int
reserve: int
shares: int
```

## why it is dangerous

a common balance check is:

```aml
require(balance >= amount, "insufficient balance")
```

but if `amount` is negative, this check passes:

```text
100 >= -50
```

that is true.

then this line becomes dangerous:

```aml
balance - amount
```

because:

```text
100 - (-50) = 150
```

so a negative amount can increase the user's balance or corrupt accounting.

## bad example

```aml
fn transfer(to: address, amount: int): bool {
  let bal = self.balances[caller]

  require(bal >= amount, "insufficient balance")

  self.balances[caller] = bal - amount
  self.balances[to] += amount

  return true
}
```

## fixed example

```aml
fn transfer(to: address, amount: int): bool {
  require(amount > 0, "invalid amount")

  let bal = self.balances[caller]
  require(bal >= amount, "insufficient balance")

  self.balances[caller] = bal - amount
  self.balances[to] += amount

  return true
}
```

## main fix, where supported

use `u128` or another unsigned type for money values where the target compiler/runtime supports it:

```aml
fn transfer(to: address, amount: u128): bool {
  require(amount > 0, "invalid amount")

  let bal = self.balances[caller]
  require(bal >= amount, "insufficient balance")

  self.balances[caller] = bal - amount
  self.balances[to] += amount

  return true
}
```

## audit check

flag any externally callable function where:

```text
amount is int
and amount can be negative
and amount is used in balance/supply/reserve accounting
and there is no require(amount > 0) or require(amount >= 0)
```

prefer changing money values from signed `int` to an unsigned type where the target compiler/runtime supports it, then keep explicit checks for zero, balances, permissions, arithmetic bounds, and invariants.

if an unsigned parameter only lacks `amount > 0`, classify that as a warning unless zero can consume replay state, create misleading events, or break accounting. unchecked arithmetic like `10 - 20` belongs in the arithmetic bounds category, not the zero-guard warning category.

## severity

critical if the bug can mint value, drain reserves, or corrupt balances

high if it only breaks accounting but does not directly drain funds

# opbr-001: unsafe signed delta accounting

## summary

not every signed value is bad.

sometimes a program needs negative numbers for deltas:

```text
profit / loss
price change
balance adjustment
funding payment
oracle deviation
```

but signed deltas are dangerous when they are mixed with balances, supply, reserves, or user funds without strict rules.

## where to look

check functions like:

- `apply_delta`
- `settle_pnl`
- `rebalance`
- `update_position`
- `apply_funding`
- `adjust_balance`
- `slash`
- `reward`

look for parameters like:

```aml
delta: int
pnl: int
change: int
adjustment: int
price_change: int
```

## why it is dangerous

signed values can be valid, but only if the program clearly defines what negative means.

bad delta logic can:

```text
turn a loss into a credit
increase balance when it should decrease
decrease reserves below zero
mint supply by accident
make accounting unrecoverable
```

## bad example

```aml
state {
  balances: map[address]int
  total_supply: int
}

fn apply_delta(user: address, delta: int): bool {
  self.balances[user] += delta
  self.total_supply += delta

  return true
}
```

why bad:

```text
delta can be negative.
balance can go below zero.
total_supply can go below zero.
any caller can apply arbitrary balance changes if auth is also missing.
```

## fixed example

```aml
state {
  owner: address
  balances: map[address]int
  total_supply: int
}

fn apply_delta(user: address, delta: int): bool {
  require(caller == self.owner, "not owner")
  assert_address(user)

  let new_balance = self.balances[user] + delta
  let new_supply = self.total_supply + delta

  require(new_balance >= 0, "negative balance")
  require(new_supply >= 0, "negative supply")

  self.balances[user] = new_balance
  self.total_supply = new_supply

  return true
}
```

## better pattern, where unsigned integers are supported

keep balances unsigned, and handle direction explicitly:

```aml
state {
  owner: address
  balances: map[address]u128
  total_supply: u128
}

fn increase_balance(user: address, amount: u128): bool {
  require(caller == self.owner, "not owner")
  require(amount > 0, "invalid amount")
  assert_address(user)

  self.balances[user] += amount
  self.total_supply += amount

  return true
}

fn decrease_balance(user: address, amount: u128): bool {
  require(caller == self.owner, "not owner")
  require(amount > 0, "invalid amount")
  assert_address(user)

  let bal = self.balances[user]
  require(bal >= amount, "insufficient balance")

  self.balances[user] = bal - amount
  self.total_supply -= amount

  return true
}
```

## audit check

flag signed delta logic where:

```text
delta can be user-controlled
delta is applied directly to balances
delta is applied directly to total_supply
delta can make balance negative
delta can make reserves negative
delta can make supply negative
auth is missing
negative meaning is not documented
```

also check:

```text
does negative delta mean loss, slash, burn, or debt?
does positive delta mean reward, mint, or credit?
can the same delta be applied twice?
does total accounting still balance after applying it?
```

## rule of thumb

```text
signed int is okay for deltas.
signed int is not okay for final asset quantities.
```

## severity

critical if signed delta can mint value, drain funds, or create negative reserves

high if it can corrupt balances, debt, pnl, or supply

medium if it only causes wrong accounting display

# opbr-002: negative allowance

## summary

allowance is permission to spend some amount on behalf of another account.

allowance should never be negative.

if a program allows negative allowance, `pull` / `transfer_from` style logic can behave incorrectly and corrupt balances or approvals.

## where to look

check functions like:

- `grant`
- `approve`
- `allowance`
- `pull`
- `transfer_from`

look for state like:

```aml
grants: map[address]map[address]int
allowances: map[address]map[address]int
```

and parameters like:

```aml
amount: int
amt: int
allowance: int
```

## why it is dangerous

a common allowance check is:

```aml
require(allowed >= amount, "insufficient allowance")
```

but if `amount` is negative, this check passes:

```text
0 >= -50
```

that is true.

then this line becomes dangerous:

```aml
self.grants[from][caller] = allowed - amount
```

because:

```text
0 - (-50) = 50
```

a negative pull amount can increase allowance instead of decreasing it.

## bad example

```aml
fn grant(spender: address, amount: int): bool {
  self.grants[caller][spender] = amount
  return true
}

fn pull(from: address, to: address, amount: int): bool {
  let allowed = self.grants[from][caller]
  require(allowed >= amount, "insufficient allowance")

  let bal = self.balances[from]
  require(bal >= amount, "insufficient balance")

  self.grants[from][caller] = allowed - amount
  self.balances[from] = bal - amount
  self.balances[to] += amount

  return true
}
```

## fixed example

```aml
fn grant(spender: address, amount: int): bool {
  require(amount >= 0, "invalid allowance")

  self.grants[caller][spender] = amount
  return true
}

fn pull(from: address, to: address, amount: int): bool {
  require(amount > 0, "invalid amount")

  let allowed = self.grants[from][caller]
  require(allowed >= amount, "insufficient allowance")

  let bal = self.balances[from]
  require(bal >= amount, "insufficient balance")

  self.grants[from][caller] = allowed - amount
  self.balances[from] = bal - amount
  self.balances[to] += amount

  return true
}
```

## better fix, where unsigned integers are supported

use unsigned types for allowances and transfer amounts:

```aml
grants: map[address]map[address]u128
```

```aml
fn grant(spender: address, amount: u128): bool {
  self.grants[caller][spender] = amount
  return true
}

fn pull(from: address, to: address, amount: u128): bool {
  require(amount > 0, "invalid amount")

  let allowed = self.grants[from][caller]
  require(allowed >= amount, "insufficient allowance")

  let bal = self.balances[from]
  require(bal >= amount, "insufficient balance")

  self.grants[from][caller] = allowed - amount
  self.balances[from] = bal - amount
  self.balances[to] += amount

  return true
}
```

## audit check

flag any `grant`, `approve`, `pull`, or `transfer_from` logic where:

```text
allowance is int
or amount is int
and amount can be negative
and allowance is checked with allowed >= amount
and there is no require(amount > 0)
```

## severity

critical if negative allowance can move funds or increase approval.

high if it only corrupts approval state.

# opbr-003: negative withdraw

## summary

withdraw amounts should always be positive.

if a vault or staking program accepts a negative withdraw amount, balance checks can pass and accounting can be reversed.

## where to look

check functions like:

- `withdraw`
- `unstake`
- `redeem`
- `claim`
- `remove_liquidity`

look for state like:

```aml
deposits: map[address]int
stakes: map[address]int
shares: map[address]int
total: int
total_locked: int
```

and parameters like:

```aml
amount: int
amt: int
shares: int
```

## why it is dangerous

a common withdraw check is:

```aml
require(user_balance >= amount, "insufficient balance")
```

but if `amount` is negative, this check passes:

```text
0 >= -100
```

that is true.

then this line becomes dangerous:

```aml
self.deposits[caller] = user_balance - amount
```

because:

```text
0 - (-100) = 100
```

the withdraw can increase the user's recorded deposit instead of reducing it.

## bad example

```aml
state {
  deposits: map[address]int
  total_locked: int
}

payable fn deposit(): int {
  require(value > 0, "must send oct")

  self.deposits[caller] += value
  self.total_locked += value

  return self.deposits[caller]
}

nonreentrant fn withdraw(amount: int): bool {
  let bal = self.deposits[caller]
  require(bal >= amount, "insufficient deposit")

  self.deposits[caller] = bal - amount
  self.total_locked -= amount

  transfer(caller, amount)
  return true
}
```

## fixed example

```aml
state {
  deposits: map[address]int
  total_locked: int
}

payable fn deposit(): int {
  require(value > 0, "must send oct")

  self.deposits[caller] += value
  self.total_locked += value

  return self.deposits[caller]
}

nonreentrant fn withdraw(amount: int): bool {
  require(amount > 0, "invalid amount")

  let bal = self.deposits[caller]
  require(bal >= amount, "insufficient deposit")

  self.deposits[caller] = bal - amount
  self.total_locked -= amount

  require(transfer(caller, amount), "transfer failed")
  return true
}
```

## better fix, where unsigned integers are supported

use unsigned types for deposits and withdraw amounts:

```aml
state {
  deposits: map[address]u128
  total_locked: u128
}

nonreentrant fn withdraw(amount: u128): bool {
  require(amount > 0, "invalid amount")

  let bal = self.deposits[caller]
  require(bal >= amount, "insufficient deposit")

  self.deposits[caller] = bal - amount
  self.total_locked -= amount

  require(transfer(caller, amount), "transfer failed")
  return true
}
```

## audit check

flag any withdraw-like function where:

```text
amount is int
and user_balance >= amount is used
and user_balance - amount is used
and there is no require(amount > 0)
```

also check that:

```text
state is updated before external transfer
total_locked cannot become negative
withdraw amount cannot be zero unless explicitly intended
```

## severity

critical if negative withdraw can create credit, drain funds, or corrupt vault solvency

high if it corrupts internal accounting without direct fund loss

# opbr-004: unchecked constructor parameters

## summary

constructor parameters define the initial state of a program.

if constructor inputs are not checked, a program can start in an invalid state before any user action happens.

## where to look

check:

- `constructor`
- `init`
- deploy-time parameters
- owner/admin setup
- initial token supply
- initial reserves
- initial fees
- multisig threshold
- oracle address
- treasury address

look for parameters like:

```aml
initial_supply: int
reserve_a: int
reserve_b: int
fee_bps: int
threshold: int
owner: address
oracle: address
treasury: address
```

## why it is dangerous

a program can be broken from deployment if it accepts invalid values like:

```text
initial_supply = -1000
reserve_a = 0
reserve_b = -50
fee_bps = 50000
threshold = 0
owner = invalid address
oracle = invalid address
```

this can cause:

- negative supply
- broken reserves
- division by zero
- impossible governance
- ownerless admin functions
- unsafe oracle reads
- fees larger than 100%

## bad example

```aml
state {
  owner: address
  total_supply: int
  fee_bps: int
}

constructor(initial_supply: int, fee_bps: int) {
  self.owner = origin
  self.total_supply = initial_supply
  self.balances[origin] = initial_supply
  self.fee_bps = fee_bps
}
```

## fixed example

```aml
state {
  owner: address
  total_supply: int
  fee_bps: int
  balances: map[address]int
}

constructor(initial_supply: int, fee_bps: int) {
  require(initial_supply > 0, "invalid supply")
  require(fee_bps >= 0, "invalid fee")
  require(fee_bps <= 10000, "fee too high")
  assert_address(origin)

  self.owner = origin
  self.total_supply = initial_supply
  self.balances[origin] = initial_supply
  self.fee_bps = fee_bps
}
```

## bad example: multisig

```aml
state {
  threshold: int
  signer_count: int
}

constructor(threshold: int, signer_count: int) {
  self.threshold = threshold
  self.signer_count = signer_count
}
```

## fixed example: multisig

```aml
state {
  threshold: int
  signer_count: int
}

constructor(threshold: int, signer_count: int) {
  require(signer_count > 0, "no signers")
  require(threshold > 0, "invalid threshold")
  require(threshold <= signer_count, "threshold too high")

  self.threshold = threshold
  self.signer_count = signer_count
}
```

## bad example: amm

```aml
state {
  reserve_a: int
  reserve_b: int
}

constructor(reserve_a: int, reserve_b: int) {
  self.reserve_a = reserve_a
  self.reserve_b = reserve_b
}
```

## fixed example: amm

```aml
state {
  reserve_a: int
  reserve_b: int
}

constructor(reserve_a: int, reserve_b: int) {
  require(reserve_a > 0, "invalid reserve a")
  require(reserve_b > 0, "invalid reserve b")

  self.reserve_a = reserve_a
  self.reserve_b = reserve_b
}
```

## audit check

flag any constructor or init function where:

```text
numeric inputs are not range-checked
addresses are not validated
thresholds can be zero or larger than signer count
fees can be negative or above max
reserves can be zero or negative
initial supply can be zero or negative
admin/owner can be invalid
```

## severity

critical if bad constructor parameters can create minting, draining, or permanent admin failure.

high if they can break accounting, governance, or market math.

medium if they only create bad metadata or bad configuration

# opbr-005: broken supply invariant

## summary

token supply must match token balances.

for a normal token, the total supply should equal the sum of all user balances.

if `total_supply` changes incorrectly, or balances change without matching supply logic, the token accounting becomes unreliable.

## core invariant

```text
total_supply == sum(all balances)
```

for most token programs, this should always be true.

## where to look

check functions like:

- `constructor`
- `transfer`
- `pull`
- `mint`
- `burn`
- `airdrop`
- `bridge_mint`
- `bridge_burn`

look for state like:

```aml
total_supply: int
balances: map[address]int
```

or, where unsigned integers are supported:

```aml
total_supply: u128
balances: map[address]u128
```

## why it is dangerous

if token supply and balances drift apart, the program may allow:

- hidden minting
- broken burns
- negative balances
- wrong explorer balances
- wrong bridge accounting
- wrong market cap / tvl display
- impossible reconciliation after an exploit

## bad example: transfer changes supply

```aml
fn transfer(to: address, amount: int): bool {
  require(amount > 0, "invalid amount")

  let bal = self.balances[caller]
  require(bal >= amount, "insufficient balance")

  self.balances[caller] = bal - amount
  self.balances[to] += amount
  self.total_supply += amount

  return true
}
```

why bad:

```text
transfer should only move tokens.
it should not increase total_supply.
```

## fixed example: transfer does not change supply

```aml
fn transfer(to: address, amount: int): bool {
  require(amount > 0, "invalid amount")

  let bal = self.balances[caller]
  require(bal >= amount, "insufficient balance")

  self.balances[caller] = bal - amount
  self.balances[to] += amount

  return true
}
```

## bad example: mint updates balance but not supply

```aml
fn mint(to: address, amount: int): bool {
  require(amount > 0, "invalid amount")
  require(caller == self.owner, "not owner")

  self.balances[to] += amount

  return true
}
```

why bad:

```text
new tokens were created, but total_supply was not increased.
```

## fixed example: mint updates balance and supply

```aml
fn mint(to: address, amount: int): bool {
  require(amount > 0, "invalid amount")
  require(caller == self.owner, "not owner")
  assert_address(to)

  self.total_supply += amount
  self.balances[to] += amount

  return true
}
```

## bad example: burn updates supply but not balance

```aml
fn burn(amount: int): bool {
  require(amount > 0, "invalid amount")

  self.total_supply -= amount

  return true
}
```

why bad:

```text
supply decreased, but the caller still has the same balance.
```

## fixed example: burn updates both balance and supply

```aml
fn burn(amount: int): bool {
  require(amount > 0, "invalid amount")

  let bal = self.balances[caller]
  require(bal >= amount, "insufficient balance")

  self.balances[caller] = bal - amount
  self.total_supply -= amount

  return true
}
```

## audit check

flag token logic where:

```text
transfer changes total_supply
mint changes balances but not total_supply
burn changes total_supply but not balances
balances can become negative
total_supply can become negative
bridge mint/burn does not have strict authorization
```

also check that every token operation preserves this rule:

```text
transfer: total_supply unchanged
mint: total_supply increases by amount
burn: total_supply decreases by amount
```

## severity

critical if supply can be inflated, hidden minting is possible, or bridge accounting can break.

high if accounting becomes inconsistent without direct minting.

medium if only displays/indexers become wrong.

# opbr-006: broken vault accounting

## summary

vault accounting must match the real funds controlled by the program.

if deposits, shares, or `total_locked` do not match the actual program balance, users may withdraw too much, get stuck, or receive the wrong share of funds.

## core invariants

```text
total_locked == sum(all user deposits)
```

and:

```text
total_locked <= actual program balance
```

for share-based vaults:

```text
total_shares == sum(all user shares)
```

and:

```text
user_assets = total_assets * user_shares / total_shares
```

## where to look

check functions like:

- `deposit`
- `withdraw`
- `stake`
- `unstake`
- `claim`
- `redeem`
- `harvest`
- `donate`
- `emergency_withdraw`

look for state like:

```aml
deposits: map[address]int
shares: map[address]int
total_locked: int
total_shares: int
rewards: map[address]int
```

## why it is dangerous

if vault accounting is wrong, users may be able to:

- withdraw more than they deposited
- create fake deposits
- inflate shares
- drain rewards
- leave the vault insolvent
- lock funds permanently

## bad example: deposit does not update total

```aml
state {
  deposits: map[address]int
  total_locked: int
}

payable fn deposit(): bool {
  require(value > 0, "invalid deposit")

  self.deposits[caller] += value

  return true
}
```

why bad:

```text
user deposit increased, but total_locked did not.
```

## fixed example

```aml
state {
  deposits: map[address]int
  total_locked: int
}

payable fn deposit(): bool {
  require(value > 0, "invalid deposit")

  self.deposits[caller] += value
  self.total_locked += value

  return true
}
```

## bad example: withdraw does not update deposit

```aml
nonreentrant fn withdraw(amount: int): bool {
  require(amount > 0, "invalid amount")
  require(self.deposits[caller] >= amount, "insufficient deposit")

  self.total_locked -= amount
  transfer(caller, amount)

  return true
}
```

why bad:

```text
funds are sent out, but the user's deposit stays the same.
the user may withdraw again.
```

## fixed example

```aml
nonreentrant fn withdraw(amount: int): bool {
  require(amount > 0, "invalid amount")

  let bal = self.deposits[caller]
  require(bal >= amount, "insufficient deposit")

  self.deposits[caller] = bal - amount
  self.total_locked -= amount

  require(transfer(caller, amount), "transfer failed")
  return true
}
```

## bad example: share minting ignores existing assets

```aml
state {
  shares: map[address]int
  total_shares: int
  total_assets: int
}

payable fn deposit(): int {
  require(value > 0, "invalid deposit")

  let minted = value

  self.shares[caller] += minted
  self.total_shares += minted
  self.total_assets += value

  return minted
}
```

why bad:

```text
if the vault already has yield or donated funds, minting 1 share per 1 asset can misprice new deposits.
```

## safer share minting example

```aml
state {
  shares: map[address]int
  total_shares: int
  total_assets: int
}

payable fn deposit(): int {
  require(value > 0, "invalid deposit")

  let minted = value

  if self.total_shares > 0 {
    require(self.total_assets > 0, "invalid vault state")
    minted = value * self.total_shares / self.total_assets
  }

  require(minted > 0, "deposit too small")

  self.shares[caller] += minted
  self.total_shares += minted
  self.total_assets += value

  return minted
}
```

## audit check

flag vault logic where:

```text
deposit updates user balance but not total_locked
withdraw updates total_locked but not user balance
shares are minted without considering total_assets / total_shares
total_locked can become negative
total_shares can become negative
rewards can be claimed without reducing claimable rewards
external transfer happens before state update
```

also check these invariants:

```text
sum(deposits) == total_locked
total_locked <= actual program balance
sum(shares) == total_shares
withdraw cannot be called twice for the same accounting credit
```

## severity

critical if users can withdraw more than they own or drain the vault

high if accounting can become insolvent or permanently inconsistent

medium if only displayed balances are wrong

# opbr-007: missing or wrong auth

## summary

some functions should not be public.

if anyone can call admin logic, the program is instantly cooked.

this is one of the most common smart program bugs.

## where to look

check functions like:

- `mint`
- `burn_from`
- `set_owner`
- `set_oracle`
- `set_fee`
- `pause`
- `unpause`
- `upgrade`
- `admin_withdraw`
- `execute`
- `set_reserve`

## why it is dangerous

if auth is missing, any user can do admin actions.

examples:

```text
anyone can mint tokens
anyone can change owner
anyone can change oracle
anyone can pause the program
anyone can withdraw admin funds
anyone can execute multisig actions
```

## bad example

```aml
fn mint(to: address, amount: int): bool {
  require(amount > 0, "invalid amount")

  self.total_supply += amount
  self.balances[to] += amount

  return true
}
```

why bad:

```text
there is no owner check.
anyone can mint.
```

## fixed example

```aml
fn mint(to: address, amount: int): bool {
  require(caller == self.owner, "not owner")
  require(amount > 0, "invalid amount")
  assert_address(to)

  self.total_supply += amount
  self.balances[to] += amount

  return true
}
```

## common variant: origin auth

using `origin` for auth is usually dangerous.

bad:

```aml
fn set_fee(fee_bps: int): bool {
  require(origin == self.owner, "not owner")

  self.fee_bps = fee_bps
  return true
}
```

fixed:

```aml
fn set_fee(fee_bps: int): bool {
  require(caller == self.owner, "not owner")
  require(fee_bps >= 0, "invalid fee")
  require(fee_bps <= 10000, "fee too high")

  self.fee_bps = fee_bps
  return true
}
```

## audit check

flag any function that changes important state and has no auth.

important state means:

```text
supply
owner
admin
oracle
fee
reserve
pause state
upgrade logic
treasury
bridge state
multisig execution
```

also flag:

```text
origin used instead of caller
admin functions callable by anyone
owner can be set to invalid address
auth check happens after state changes
```

## severity

critical if anyone can mint, withdraw, upgrade, or execute admin calls.

high if anyone can change oracle, fee, owner, or pause state.

medium if the function only changes metadata.

# opbr-008: unsafe oracle / price feed usage

## summary

if a program uses a price feed, the price must be checked.

bad oracle logic can break stablecoins, lending, amms, vaults, liquidations, and minting.

even if the oracle is trusted, the value can still be stale, zero, negative, too large, or wrong decimals.

## where to look

check functions like:

- `set_price`
- `update_price`
- `mint`
- `redeem`
- `borrow`
- `liquidate`
- `swap`
- `deposit_collateral`

look for state like:

```aml
oracle: address
price: int
last_price_update: int
decimals: int
```

## why it is dangerous

if price is wrong, the whole program accounting is wrong.

examples:

```text
price = 0
price = -1
price is stale
price uses wrong decimals
price can be updated by wrong address
price jumps 100x in one update
```

this can lead to:

```text
free mint
bad redeem
wrong liquidation
bad collateral ratio
reserve drain
stablecoin depeg
```

## bad example

```aml
state {
  oracle: address
  price: int
  balances: map[address]int
}

fn set_price(new_price: int): bool {
  self.price = new_price
  return true
}

payable fn mint(): bool {
  let minted = value * self.price

  self.balances[caller] += minted
  return true
}
```

why bad:

```text
anyone can set price.
price can be zero or negative.
price can use wrong decimals.
mint trusts price blindly.
```

## fixed example

```aml
state {
  oracle: address
  price: int
  balances: map[address]int
}

fn set_price(new_price: int): bool {
  require(caller == self.oracle, "not oracle")
  require(new_price > 0, "invalid price")

  self.price = new_price
  return true
}

payable fn mint(): bool {
  require(value > 0, "invalid deposit")
  require(self.price > 0, "invalid price")

  let minted = value * self.price

  self.balances[caller] += minted
  return true
}
```

## better example

```aml
state {
  oracle: address
  price: int
  last_price_update: int
  max_price_age: int
  balances: map[address]int
}

fn set_price(new_price: int, updated_at: int): bool {
  require(caller == self.oracle, "not oracle")
  require(new_price > 0, "invalid price")
  require(updated_at > 0, "invalid timestamp")

  self.price = new_price
  self.last_price_update = updated_at

  return true
}

payable fn mint(now: int): bool {
  require(value > 0, "invalid deposit")
  require(self.price > 0, "invalid price")
  require(now - self.last_price_update <= self.max_price_age, "stale price")

  let minted = value * self.price

  self.balances[caller] += minted
  return true
}
```

## audit check

flag oracle logic where:

```text
price can be zero
price can be negative
price can be stale
price has unclear decimals
price can be updated by anyone
price is used without validation
large price jumps are not checked
mint/redeem/borrow uses price blindly
```

## severity

critical if bad price can mint value, drain reserves, or bypass collateral rules.

high if bad price can cause wrong liquidations or bad redemptions.

medium if bad price only affects display or analytics.

# opbr-009: checks-effects-interactions violation

## summary

checks-effects-interactions, or cei, is a common security pattern.

the safe order is:

```text
checks -> effects -> interactions
```

which means:

```text
1. check inputs, permissions, balances, limits
2. update program state
3. call or transfer to external addresses
```

if a program does external calls before updating state, the checked state may still look valid during the external action.

this can lead to double claim, double execute, stale accounting, or reentrancy-style bugs.

## where to look

check functions like:

- `withdraw`
- `claim`
- `redeem`
- `execute`
- `swap`
- `liquidate`
- `settle`
- `finalize`
- `distribute`

look for external actions like:

```aml
transfer(to, amount)
call(target, method, args...)
deploy(...)
```

## why it is dangerous

security checks only help if the state is updated before the risky external action.

bad order:

```text
check state
external transfer/call
update state
```

safe order:

```text
check state
update state
external transfer/call
```

bad order can allow:

```text
double withdraw
double claim
double execute
stale reserves
stale shares
stale accounting
```

## bad example: claim flag updated after transfer

```aml
fn claim(id: int): bool {
  // check: user has something to claim
  require(self.claimable[id][caller] > 0, "nothing to claim")

  // read claim amount
  let amount = self.claimable[id][caller]

  // interaction: external transfer happens first
  // bad: claim is still active here
  transfer(caller, amount)

  // effect: state is updated too late
  self.claimable[id][caller] = 0

  return true
}
```

why bad:

```text
the program sends funds before clearing the claim.
the claim is still active during the external transfer.
```

## fixed example

```aml
fn claim(id: int): bool {
  // check: user has something to claim
  require(self.claimable[id][caller] > 0, "nothing to claim")

  // read claim amount
  let amount = self.claimable[id][caller]

  // effect: clear claim before sending funds
  self.claimable[id][caller] = 0

  // interaction: external transfer happens last
  require(transfer(caller, amount), "transfer failed")

  return true
}
```

## bad example: proposal marked executed after call

```aml
fn execute(id: u128): bool {
  // check: enough approvals
  require(self.approvals[id] >= self.threshold, "not enough approvals")

  // check: proposal was not executed yet
  require(!self.executed[id], "already executed")

  // interaction: external call happens first
  // bad: proposal is still marked as not executed here
  call(self.target[id], self.method[id], self.args[id])

  // effect: executed flag is updated too late
  self.executed[id] = true

  return true
}
```

why bad:

```text
the external call happens while the proposal is still marked as not executed.
```

## fixed example

```aml
fn execute(id: u128): bool {
  // check: enough approvals
  require(self.approvals[id] >= self.threshold, "not enough approvals")

  // check: proposal was not executed yet
  require(!self.executed[id], "already executed")

  // effect: mark executed before external call
  self.executed[id] = true

  // interaction: external call happens last
  require(
    call(self.target[id], self.method[id], self.args[id]),
    "proposal call failed"
  )

  return true
}
```

## bad example: reserves updated after payout

```aml
fn swap(amount_in: int): int {
  // check: input must be positive
  require(amount_in > 0, "invalid amount")

  // check: reserve must be usable
  require(self.reserve_x > 0, "bad reserve x")

  // calculate output using current reserves
  let out = amount_in * self.reserve_y / self.reserve_x

  // interaction: payout happens first
  // bad: reserves are still old here
  transfer(caller, out)

  // effect: reserves are updated too late
  self.reserve_x += amount_in
  self.reserve_y -= out

  return out
}
```

why bad:

```text
the program pays out before updating reserves.
during the external action, reserve state is stale.
```

## fixed example

```aml
fn swap(amount_in: int): int {
  // check: input must be positive
  require(amount_in > 0, "invalid amount")

  // check: reserves must be usable
  require(self.reserve_x > 0, "bad reserve x")
  require(self.reserve_y > 0, "bad reserve y")

  // calculate output using current reserves
  let out = amount_in * self.reserve_y / self.reserve_x

  // check: output must be valid
  require(out > 0, "zero output")
  require(self.reserve_y >= out, "insufficient reserve")

  // effect: update reserves before payout
  self.reserve_x += amount_in
  self.reserve_y -= out

  // interaction: payout happens last
  require(transfer(caller, out), "transfer failed")

  return out
}
```

## audit check

flag functions where:

```text
state is checked
then transfer/call/deploy happens
then checked state is updated
```

also flag:

```text
claim is cleared after transfer
withdraw balance is reduced after transfer
executed is set after call
reserves are updated after payout
shares are burned after redeem transfer
rewards are marked claimed after reward transfer
```

## rule of thumb

```text
checks first
effects second
interactions last
```

## severity

critical if wrong order can cause double withdraw, double claim, double execute, or reserve drain

high if wrong order can corrupt reserves, shares, rewards, or accounting

medium if it only causes inconsistent state or bad events

# opbr-010: replay / double execution

## summary

an action that should happen once can happen more than once.

this is common in:

```text
claims
bridge withdrawals
airdrops
multisig proposals
orders
redemptions
liquidations
```

## where to look

check functions like:

- `execute`
- `claim`
- `redeem`
- `withdraw_message`
- `finalize_bridge`
- `fill_order`
- `settle`
- `liquidate`
- `use_signature`

look for state like:

```aml
executed: map[u128]bool
claimed: map[u128]map[address]bool
used_nonce: map[address]map[u128]bool
processed_message: map[u128]bool
```

## why it is dangerous

if the program does not remember that an action was already used, the same proof/order/message/signature can be submitted again.

that can lead to:

```text
double claim
double bridge withdrawal
double airdrop
double vote
double order fill
double multisig execute
```

## bad example: bridge message can be reused

```aml
fn finalize_bridge(message_id: u128, to: address, amount: int): bool {
  require(amount > 0, "invalid amount")
  assert_address(to)

  require(verify_message(message_id, to, amount), "invalid message")

  self.balances[to] += amount
  self.total_supply += amount

  return true
}
```

why bad:

```text
the message is valid, but it is never marked as used.
the same message can be submitted again.
```

## fixed example

```aml
state {
  processed_messages: map[u128]bool
  balances: map[address]int
  total_supply: int
}

fn finalize_bridge(message_id: u128, to: address, amount: int): bool {
  require(amount > 0, "invalid amount")
  assert_address(to)

  require(!self.processed_messages[message_id], "message already processed")
  require(verify_message(message_id, to, amount), "invalid message")

  self.processed_messages[message_id] = true

  self.balances[to] += amount
  self.total_supply += amount

  return true
}
```

## bad example: airdrop claim can be repeated

```aml
fn claim_airdrop(drop_id: u128, amount: int, proof: string): bool {
  require(amount > 0, "invalid amount")
  require(verify_proof(drop_id, caller, amount, proof), "invalid proof")

  self.balances[caller] += amount
  self.total_supply += amount

  return true
}
```

## fixed example

```aml
state {
  claimed: map[u128]map[address]bool
  balances: map[address]int
  total_supply: int
}

fn claim_airdrop(drop_id: u128, amount: int, proof: string): bool {
  require(amount > 0, "invalid amount")
  require(!self.claimed[drop_id][caller], "already claimed")
  require(verify_proof(drop_id, caller, amount, proof), "invalid proof")

  self.claimed[drop_id][caller] = true

  self.balances[caller] += amount
  self.total_supply += amount

  return true
}
```

## bad example: signature nonce not used

```aml
fn use_signature(owner: address, amount: int, nonce: u128, sig: string): bool {
  require(amount > 0, "invalid amount")
  require(verify_sig(owner, amount, nonce, sig), "bad signature")

  self.balances[owner] -= amount
  self.balances[caller] += amount

  return true
}
```

## fixed example

```aml
state {
  used_nonce: map[address]map[u128]bool
  balances: map[address]int
}

fn use_signature(owner: address, amount: int, nonce: u128, sig: string): bool {
  require(amount > 0, "invalid amount")
  assert_address(owner)

  require(!self.used_nonce[owner][nonce], "nonce already used")
  require(verify_sig(owner, amount, nonce, sig), "bad signature")

  self.used_nonce[owner][nonce] = true

  let bal = self.balances[owner]
  require(bal >= amount, "insufficient balance")

  self.balances[owner] = bal - amount
  self.balances[caller] += amount

  return true
}
```

## audit check

flag any one-time action where:

```text
there is no used/claimed/executed flag
the flag is set after external call
nonce is not stored
message id is not stored
signature can be reused
proof can be reused
proposal can be executed twice
```

## rule of thumb

```text
if an action should happen once, store that it happened before doing the action.
```

## severity

critical if replay can mint, withdraw, bridge, or execute funds twice

high if replay can double vote, double claim rewards, or fill orders twice

medium if replay only causes duplicate events or wrong accounting

# opbr-011: unchecked call or transfer result

## summary

aml programs can call other programs with `call(...)` and can send native OCT with `transfer(...)`.

`call(...)` returns whether the external program call succeeded. official examples also use `transfer(...)` in checked forms such as `require(transfer(...), "transfer failed")`.

if the result is ignored, the program may continue as if the external action worked, even when it failed.

this is similar to the ethereum bug class:

```text
unchecked call return value
```

## where to look

check functions like:

- `execute`
- `admin_call`
- `forward`
- `swap`
- `redeem`
- `bridge_call`
- `finalize_message`
- `claim_and_call`
- `batch_call`
- `withdraw`
- `release`
- `refund`

look for code like:

```aml
call(target, method, args...)
transfer(to, amount)
```

## why it is dangerous

the program may update its own state, call another program or transfer OCT, ignore failure, and return success.

this can create broken state.

examples:

```text
proposal marked executed, but target call failed
bridge message marked processed, but target call failed
swap state updated, but token transfer failed
redeem marked complete, but payout transfer failed
```

## bad example

```aml
fn execute(id: u128): bool {
  // check: proposal was not executed
  require(!self.executed[id], "already executed")

  // effect: mark proposal as executed
  self.executed[id] = true

  // bad: external call result is ignored
  call(self.target[id], self.method[id], self.arg[id])

  return true
}
```

why bad:

```text
if call(...) fails, executed[id] may still be true unless the program reverts.
the proposal cannot be retried, but the action did not happen.
```

## fixed example

```aml
fn execute(id: u128): bool {
  // check: proposal was not executed
  require(!self.executed[id], "already executed")

  // effect: mark proposal as executed
  self.executed[id] = true

  // good: require external call success
  require(
    call(self.target[id], self.method[id], self.arg[id]),
    "external call failed"
  )

  return true
}
```

## bad example: token swap

```aml
fn swap(amount_x: int, amount_y: int, recipient: address): bool {
  require(amount_x > 0, "invalid amount x")
  require(amount_y > 0, "invalid amount y")
  assert_address(recipient)

  // bad: pull may fail, but result is ignored
  call(self.token_x, "pull", caller, self_addr, amount_x)

  // bad: transfer may fail, but result is ignored
  call(self.token_y, "transfer", recipient, amount_y)

  return true
}
```

## fixed example: token swap

```aml
fn swap(amount_x: int, amount_y: int, recipient: address): bool {
  require(amount_x > 0, "invalid amount x")
  require(amount_y > 0, "invalid amount y")
  assert_address(recipient)

  // good: stop if token_x pull fails
  require(
    call(self.token_x, "pull", caller, self_addr, amount_x),
    "pull token x failed"
  )

  // good: stop if token_y transfer fails
  require(
    call(self.token_y, "transfer", recipient, amount_y),
    "push token y failed"
  )

  return true
}
```

## audit check

flag every external action where:

```text
call(...) is not inside require(...)
call(...) result is not assigned and checked
transfer(...) result is not checked when payout success matters
state is updated as if the external action succeeded
message/proposal/order is consumed before unchecked call or transfer
swap/redeem/bridge logic ignores external action failure
```

good patterns:

```aml
require(call(target, method, args...), "external call failed")
require(transfer(to, amount), "transfer failed")
```

or:

```aml
let ok = call(target, method, args...)
require(ok, "external call failed")
```

## rule of thumb

```text
never ignore call(...).
if another program must do something, require that it actually succeeded.
```

## severity

critical if unchecked call failure can fake execution, lock funds, consume bridge messages, or break swaps.

high if it can corrupt accounting, proposals, orders, or redemptions.

medium if it only causes bad events or wrong UI state

# opbr-012: reentrancy / missing nonreentrant guard

## summary

reentrancy happens when a program starts an external action, such as `transfer(...)` or `call(...)`, and the external side can enter the program again before the first execution has finished.

if the first execution has not fully updated or locked the state, the second execution can pass the same checks again.

in AML, the practical guard is simple:

```aml
nonreentrant fn withdraw(amount: int): bool {
  ...
}
```

so yes: for reentrancy-sensitive entrypoints, you often just write `nonreentrant fn`.

this matches the official vault template style, where withdraw-like logic is declared as `nonreentrant fn withdraw(...)`.

that guard is not a replacement for checks-effects-interactions. use both:

```text
nonreentrant entrypoint
checks
effects
checked interaction
```

## where to look

check functions like:

- `withdraw`
- `claim`
- `redeem`
- `unstake`
- `release`
- `refund`
- `execute`
- `swap`
- `bridge_finalize`

look for functions that:

```text
read a user balance, claim, proposal, or escrow state
then call transfer(...) or call(...)
and are declared as fn instead of nonreentrant fn
```

## why it is dangerous

if a withdraw-like function can be entered again before the first call finishes, the same balance or claim may be used more than once.

this can lead to:

```text
double withdraw
double claim
double redeem
double proposal execution
stale reserve usage
vault drain
```

## bad example

```aml
state {
  deposits: map[address]int
  total_locked: int
}

fn withdraw(amount: int): bool {
  require(amount > 0, "invalid amount")

  let bal = self.deposits[caller]
  require(bal >= amount, "insufficient deposit")

  // bad: external payout happens while the old balance is still stored
  require(transfer(caller, amount), "transfer failed")

  self.deposits[caller] = bal - amount
  self.total_locked -= amount

  return true
}
```

## fixed example

```aml
state {
  deposits: map[address]int
  total_locked: int
}

nonreentrant fn withdraw(amount: int): bool {
  require(amount > 0, "invalid amount")

  let bal = self.deposits[caller]
  require(bal >= amount, "insufficient deposit")

  self.deposits[caller] = bal - amount
  self.total_locked -= amount

  require(transfer(caller, amount), "transfer failed")
  return true
}
```

## better fix, where unsigned integers are supported

use `u128` for deposits and amounts where supported, keep the `nonreentrant` guard, and still check zero amounts and transfer success:

```aml
state {
  deposits: map[address]u128
  total_locked: u128
}

nonreentrant fn withdraw(amount: u128): bool {
  require(amount > 0, "invalid amount")

  let bal = self.deposits[caller]
  require(bal >= amount, "insufficient deposit")

  self.deposits[caller] = bal - amount
  self.total_locked -= amount

  require(transfer(caller, amount), "transfer failed")
  return true
}
```

## audit check

flag externally callable functions where:

```text
function sends OCT with transfer(...)
or function calls another program with call(...)
and function reads or updates user/proposal/claim/vault state
and function is declared as fn instead of nonreentrant fn
```

also flag:

```text
external action happens before state update
transfer(...) or call(...) result is ignored
nonreentrant is used but amount can be negative
nonreentrant is used but replay/claimed/executed flags are missing
```

## rule of thumb

```text
if an entrypoint pays out or calls out after reading sensitive state, write nonreentrant fn.
then still update state before the external action and require the external action to succeed.
```

## severity

critical if reentrancy can drain funds, withdraw twice, claim twice, or execute a proposal twice.

high if it can corrupt accounting, reserves, proposal state, or escrow state.

medium if it only causes duplicate events or temporary inconsistent state.

# opbr-014: missing exists flag / default map state

## summary

maps can return default values for keys that were never created.

if a function only checks a status map like `executed[id]`, `claimed[id]`, or `filled[id]`, an unknown id can look like a valid unexecuted object.

example:

```text
executed[999] defaults to false
!executed[999] is true
execute(999) passes
```

the fix is to store and check an explicit existence flag.

## where to look

check functions like:

- `execute`
- `claim`
- `redeem`
- `cancel`
- `fill_order`
- `settle`
- `finalize_message`
- `vote`
- `update_position`

look for state like:

```aml
executed: map[int]bool
claimed: map[int]bool
filled: map[int]bool
targets: map[int]address
amounts: map[int]int
```

without:

```aml
exists: map[int]bool
```

## why it is dangerous

unknown ids can pass checks that were meant for real objects.

this can lead to:

```text
fake proposal execution
fake claim execution
fake order fill
misleading events
junk state writes
default target or amount usage
admin or bridge logic running on an object that was never created
```

## bad example

```aml
state {
  next_id: int
  targets: map[int]address
  amounts: map[int]int
  executed: map[int]bool
}

fn propose(target: address, amount: int): int {
  assert_address(target)
  require(amount > 0, "invalid amount")

  let id = self.next_id
  self.targets[id] = target
  self.amounts[id] = amount
  self.executed[id] = false
  self.next_id = id + 1

  return id
}

fn execute(id: int): bool {
  require(!self.executed[id], "already executed")

  self.executed[id] = true
  require(transfer(self.targets[id], self.amounts[id]), "transfer failed")

  return true
}
```

why bad:

```text
execute(999) can pass if executed[999] defaults to false.
the program marks an object as executed even though it was never proposed.
```

## fixed example

```aml
state {
  next_id: int
  exists: map[int]bool
  targets: map[int]address
  amounts: map[int]int
  executed: map[int]bool
}

fn propose(target: address, amount: int): int {
  assert_address(target)
  require(amount > 0, "invalid amount")

  let id = self.next_id
  self.exists[id] = true
  self.targets[id] = target
  self.amounts[id] = amount
  self.executed[id] = false
  self.next_id = id + 1

  return id
}

fn execute(id: int): bool {
  require(id >= 0, "invalid id")
  require(self.exists[id], "proposal not found")
  require(!self.executed[id], "already executed")

  self.executed[id] = true
  require(transfer(self.targets[id], self.amounts[id]), "transfer failed")

  return true
}
```

## audit check

flag id-based logic where:

```text
function accepts id/order_id/claim_id/message_id/position_id
map[id] is read before an existence check
only executed[id], claimed[id], filled[id], or voted[id] is checked
there is no exists[id] / created[id] / owner[id] validation
unknown id can write state or emit events
```

also check:

```text
id lower bound is missing
id upper bound is missing
default target/address/amount can be used
exists flag is set after external action
delete/close logic clears exists consistently
```

## rule of thumb

```text
for every id-based object, store exists[id] = true when it is created.
before operating on id, require exists[id].
then check status flags like executed[id] or claimed[id].
```

## severity

critical if an unknown id can execute transfers, admin calls, bridge messages, or claims.

high if it can corrupt proposal, order, claim, position, or message state.

medium if it only creates misleading events or junk storage.

# opbr-015: broken accounting / economic invariant

## summary

an invariant is a rule that must stay true after every action.

this bug happens when all local checks pass, but the main accounting or economic rule becomes false.

common examples:

```text
total_supply == sum(all balances)
total_locked == sum(all deposits)
total_shares == sum(all shares)
reserve_a * reserve_b should not decrease in a constant-product AMM
wrapped_supply <= locked_source_assets
threshold <= number of unique signers
```

this is not always a missing `require(...)` on one line. it is often a mismatch between what the program checks and what the protocol actually needs to keep true.

## invariant examples

token:

```text
total_supply == sum(balances)
transfer: total_supply is unchanged
mint: total_supply increases by amount and receiver balance increases by amount
burn: total_supply decreases by amount and holder balance decreases by amount
allowance[from][spender] never increases during pull/transfer_from
```

simple OCT vault:

```text
total_locked == sum(deposits)
total_locked <= actual program OCT balance
deposit: deposits[user] and total_locked increase by the same value
withdraw: deposits[user] and total_locked decrease by the same amount
admin fees are separate from user deposits
```

share vault:

```text
total_shares == sum(shares)
total_assets is enough to satisfy all user shares
deposit mints shares > 0
redeem burns shares > 0
assets_out <= user claim on total_assets
price per share changes only according to documented yield/fee/donation rules
```

AMM:

```text
reserve_a >= 0 and reserve_b >= 0
reserves match actual balances after swaps and liquidity changes
constant-product AMM: k = reserve_a * reserve_b does not decrease unexpectedly
swap output > 0
swap output < reserve_out
LP total supply == sum(LP balances)
add/remove liquidity keeps LP shares proportional to reserves
```

bridge:

```text
wrapped_supply <= locked_source_assets
message/proof can be processed once
minted amount equals verified message amount
burn/release decreases wrapped or locked accounting exactly once
proof binds recipient, amount, source chain/domain, and message id
```

governance / multisig:

```text
threshold > 0
threshold <= unique_signer_count
owners/signers are unique
proposal exists before vote or execute
each signer votes at most once
vote_count == number of unique yes votes
executed proposal cannot execute again
executed proposal cannot be modified
```

fee accounting:

```text
fee_bps >= 0 and fee_bps <= 10000
fee_amount <= amount
protocol_fees <= actual balance not owed to users
overpayment is refunded or explicitly accounted as payment
treasury/fee recipient is valid
```

id lifecycle:

```text
exists[id] is true before operating on id
id >= 0 and id < next_id
status transition is valid
closed/executed/cancelled object cannot be reused
delete/close clears or freezes related state consistently
```

## where to look

check:

- token `transfer`, `mint`, `burn`, `pull`
- vault `deposit`, `withdraw`, `redeem`, `harvest`
- share vault `deposit`, `mint`, `redeem`
- AMM `swap`, `add_liquidity`, `remove_liquidity`
- bridge `lock`, `mint`, `burn`, `release`
- multisig/governance `propose`, `vote`, `execute`
- admin/emergency accounting

## why it is dangerous

broken invariants can make a program insolvent or economically wrong even when each function appears to run normally.

examples:

```text
token balances no longer match total_supply
vault user deposits no longer match total_locked
shares are minted at the wrong price
reserves drift away from actual balances
wrapped assets are minted without backing
governance threshold becomes impossible or too easy
```

## bad example: vault total not updated

```aml
state {
  deposits: map[address]int
  total_locked: int
}

payable fn deposit(): bool {
  require(value > 0, "invalid deposit")

  self.deposits[caller] += value

  return true
}
```

why bad:

```text
deposits[caller] increased, but total_locked did not.
the invariant total_locked == sum(deposits) is broken.
```

## fixed example

```aml
state {
  deposits: map[address]int
  total_locked: int
}

payable fn deposit(): bool {
  require(value > 0, "invalid deposit")

  self.deposits[caller] += value
  self.total_locked += value

  return true
}
```

## bad example: token supply drift

```aml
state {
  balances: map[address]int
  total_supply: int
  owner: address
}

fn mint(to: address, amount: int): bool {
  require(caller == self.owner, "not owner")
  require(amount > 0, "invalid amount")
  assert_address(to)

  self.balances[to] += amount

  return true
}
```

why bad:

```text
new balance was created, but total_supply was not increased.
the invariant total_supply == sum(balances) is broken.
```

## fixed example

```aml
state {
  balances: map[address]int
  total_supply: int
  owner: address
}

fn mint(to: address, amount: int): bool {
  require(caller == self.owner, "not owner")
  require(amount > 0, "invalid amount")
  assert_address(to)

  self.total_supply += amount
  self.balances[to] += amount

  return true
}
```

## audit check

write down the invariant first, then check every function that can affect it.

flag logic where:

```text
function updates user balance but not total
function updates total but not user balance
shares are minted without checking total_assets / total_shares
reserves are updated without matching actual received/sent amounts
bridge mints without a matching lock/proof/process flag
threshold or owner count can make governance impossible
emergency/admin path bypasses normal accounting
```

invariant examples to test:

```text
token: total_supply == sum(balances)
vault: total_locked == sum(deposits)
share vault: total_shares == sum(shares)
amm: reserves match actual balances and k does not decrease unexpectedly
bridge: wrapped_supply <= locked_assets
governance: threshold <= unique_signer_count
id lifecycle: exists[id] and status agree with allowed actions
```

reference links for more examples:

- [EIP-20 ERC-20 token standard](https://eips.ethereum.org/EIPS/eip-20)
- [EIP-4626 tokenized vault standard](https://eips.ethereum.org/EIPS/eip-4626)
- [OpenZeppelin ERC-4626 docs](https://docs.openzeppelin.com/contracts/5.x/erc4626)
- [Uniswap docs: how Uniswap works](https://developers.uniswap.org/docs/get-started/concepts/how-uniswap-works)
- [Uniswap glossary: constant product formula](https://developers.uniswap.org/docs/get-started/concepts/glossary)

## rule of thumb

```text
if a variable is a total, every function that changes an item must change the total too.
if a function creates, burns, locks, unlocks, mints, or pays value, write the invariant it must preserve.
```

## severity

critical if the broken invariant can mint value, drain funds, make a vault insolvent, or create unbacked bridge assets.

high if it can corrupt accounting, shares, reserves, or governance execution.

medium if it only affects reporting, events, or non-critical counters.

# opbr-016: rounding / zero shares

## summary

AML integer math does not keep fractions.

when a vault or AMM divides integers, the fractional part is lost.

example:

```text
1 * 100 / 1000 = 0
```

in a share vault, this can mean a deposit is accepted, but the user receives `0` shares.

## where to look

check functions like:

- `deposit`
- `mint`
- `redeem`
- `withdraw`
- `swap`
- `add_liquidity`
- `remove_liquidity`

look for formulas like:

```aml
minted = amount * total_shares / total_assets
assets_out = shares * total_assets / total_shares
out = amount_in * reserve_out / reserve_in
fee = amount * fee_bps / 10000
```

## why it is dangerous

if the result rounds to zero and the program still updates state, users can lose value or accounting can drift.

examples:

```text
deposit accepted but minted shares == 0
redeem burns shares but assets_out == 0
swap accepts input but output == 0
fee rounds to zero when it should not
```

## bad example

```aml
state {
  total_assets: int
  total_shares: int
  shares: map[address]int
}

fn deposit(amount: int): int {
  require(amount > 0, "invalid amount")

  let minted = amount * self.total_shares / self.total_assets

  self.total_assets += amount
  self.shares[caller] += minted
  self.total_shares += minted

  return minted
}
```

why bad:

```text
if total_assets = 1000, total_shares = 100, and amount = 1:
minted = 1 * 100 / 1000 = 0.
the vault receives the deposit, but the user receives 0 shares.
```

## fixed example: reject zero shares

```aml
state {
  total_assets: int
  total_shares: int
  shares: map[address]int
}

fn deposit(amount: int): int {
  require(amount > 0, "invalid amount")

  let minted = amount

  if self.total_shares > 0 {
    require(self.total_assets > 0, "bad vault state")
    minted = amount * self.total_shares / self.total_assets
  }

  require(minted > 0, "deposit too small")

  self.total_assets += amount
  self.shares[caller] += minted
  self.total_shares += minted

  return minted
}
```

## better pattern: scaled internal share units

because there are no floats, store fractional shares as larger integer units.

```text
1 real share = 1_000_000 internal share units
```

```aml
const SHARE_SCALE: int = 1000000

state {
  total_assets: int
  total_shares: int
  shares: map[address]int
}

fn deposit(amount: int): int {
  require(amount > 0, "invalid amount")

  let minted = amount * SHARE_SCALE

  if self.total_shares > 0 {
    require(self.total_assets > 0, "bad vault state")
    minted = amount * self.total_shares / self.total_assets
  }

  require(minted > 0, "deposit too small")

  self.total_assets += amount
  self.shares[caller] += minted
  self.total_shares += minted

  return minted
}
```

with scaled shares:

```text
total_assets = 1000
total_shares = 100 * 1_000_000
amount = 1
minted = 1 * 100_000_000 / 1000 = 100_000
```

the user receives `0.1` real share represented as `100000` internal share units.

## ethereum-style mitigations

ERC-4626 vault implementations usually combine several defenses:

```text
reject zero shares
use higher precision share units
use virtual shares and virtual assets
seed the vault / dead shares for the initial rate
allow the user to pass min_shares as slippage protection
```

virtual shares/assets make the empty or low-liquidity vault rate harder to manipulate:

```aml
const VIRTUAL_ASSETS: int = 1
const VIRTUAL_SHARES: int = 1000000

let minted =
  amount * (self.total_shares + VIRTUAL_SHARES) /
  (self.total_assets + VIRTUAL_ASSETS)

require(minted > 0, "deposit too small")
```

`min_shares` protects the user from receiving fewer shares than expected:

```aml
fn deposit(amount: int, min_shares: int): int {
  require(amount > 0, "invalid amount")
  require(min_shares > 0, "invalid min shares")

  let minted = amount

  if self.total_shares > 0 {
    require(self.total_assets > 0, "bad vault state")
    minted = amount * self.total_shares / self.total_assets
  }

  require(minted >= min_shares, "slippage")

  self.total_assets += amount
  self.shares[caller] += minted
  self.total_shares += minted

  return minted
}
```

reference links:

- [OpenZeppelin ERC-4626 docs](https://docs.openzeppelin.com/contracts/5.x/erc4626)
- [OpenZeppelin: ERC4626 inflation attack defense](https://www.openzeppelin.com/news/a-novel-defense-against-erc4626-inflation-attacks)
- [EIP-4626 tokenized vault standard](https://eips.ethereum.org/EIPS/eip-4626)

## audit check

flag share or output math where:

```text
division result can become zero
deposit updates assets before checking minted > 0
redeem burns shares before checking assets_out > 0
swap accepts input before checking out > 0
share accounting uses low precision units
rounding direction is not documented
no min_shares / min_out protection exists for user-facing deposits or swaps
```

also check:

```text
first deposit path is handled separately
total_assets > 0 before dividing
total_shares > 0 before proportional share math
scaled internal units are used when small deposits should be supported
virtual shares/assets or initial seed/dead shares are considered for empty vaults
```

## rule of thumb

```text
after any division that creates shares, assets_out, swap out, or fees, check the result is non-zero when zero is not meaningful.
if fractional ownership matters, store it as scaled integer units.
for user-facing deposits or swaps, let the user set min_shares or min_out.
```

## severity

high if users can lose deposits, shares, redemptions, or swap inputs due to rounding.

medium if it only rejects small users or causes small accounting drift.

low if zero output is explicitly intended and documented.

# opbr-017: bad id bounds / negative id

## summary

many AML programs use `int` ids for proposals, claims, orders, models, and positions.

if a function checks only the upper bound, a negative id can pass.

bad pattern:

```aml
require(id < self.next_id, "invalid id")
```

missing:

```aml
require(id >= 0, "invalid id")
```

## where to look

check functions like:

- `execute(id)`
- `claim(id)`
- `vote(id)`
- `cancel(id)`
- `fill_order(id)`
- `get_model(id)`
- `update_position(id)`
- `finalize_message(id)`

look for:

```aml
require(id < self.next_id, "invalid id")
require(id < self.count, "invalid id")
require(id <= self.max_id, "invalid id")
```

without a matching lower-bound check.

## why it is dangerous

negative ids can read or write unexpected map slots, bypass existence logic, or operate on default state.

this can lead to:

```text
fake proposal execution
invalid claim or order state
negative model/position access
junk state writes
default map values being treated as real objects
```

## bad example

```aml
state {
  next_id: int
  exists: map[int]bool
  executed: map[int]bool
}

fn execute(id: int): bool {
  require(id < self.next_id, "invalid id")
  require(self.exists[id], "not found")
  require(!self.executed[id], "already executed")

  self.executed[id] = true
  return true
}
```

## fixed example

```aml
state {
  next_id: int
  exists: map[int]bool
  executed: map[int]bool
}

fn execute(id: int): bool {
  require(id >= 0, "invalid id")
  require(id < self.next_id, "invalid id")
  require(self.exists[id], "not found")
  require(!self.executed[id], "already executed")

  self.executed[id] = true
  return true
}
```

## audit check

flag id-based functions where:

```text
id is int
id is user-controlled
id is checked with id < count or id < next_id
there is no require(id >= 0)
map[id] is read or written
```

also check:

```text
id lower bound is checked before map reads
id upper bound matches the creation range
exists[id] is checked for object ids
closed/deleted ids cannot be reused accidentally
```

## rule of thumb

```text
for int ids, check both sides: id >= 0 and id < next_id.
then check exists[id].
```

## severity

high if negative ids can execute, claim, vote, withdraw, or mutate important state.

medium if they only create junk storage or bad reads.

low if all negative ids are harmless view-only reads.

# opbr-018: duplicate signer / bad threshold

## summary

multisigs, committees, bridge validators, and governance systems need a valid quorum.

the signer set must be unique, and the threshold must be possible.

bad states:

```text
threshold == 0
threshold > unique_signer_count
same signer appears twice
same signer can vote twice
```

## where to look

check:

- constructors
- owner/signer setup
- `add_owner`
- `remove_owner`
- `set_threshold`
- `vote`
- `execute`
- bridge guardian or validator setup

look for state like:

```aml
owners: list[address]
threshold: int
votes: map[int]map[address]bool
vote_counts: map[int]int
```

## why it is dangerous

duplicate signers can make a quorum easier than intended.

bad thresholds can make governance impossible or let anyone execute.

examples:

```text
same address is added twice and counts as two owners
threshold is set to 0 and execution requires no real approval
threshold is larger than owner count and execution is permanently stuck
removed signer can still vote
vote_count does not match unique yes votes
```

## bad example

```aml
state {
  owners: list[address]
  threshold: int
}

constructor(a: address, b: address, threshold_val: int) {
  assert_address(a)
  assert_address(b)

  self.owners.push(a)
  self.owners.push(b)
  self.threshold = threshold_val
}
```

why bad:

```text
a and b can be the same address.
threshold can be 0.
threshold can be greater than the number of unique owners.
```

## fixed example

```aml
state {
  owners: list[address]
  is_owner: map[address]bool
  threshold: int
  owner_count: int
}

constructor(a: address, b: address, threshold_val: int) {
  assert_address(a)
  assert_address(b)
  require(a != b, "duplicate owner")
  require(threshold_val > 0, "invalid threshold")
  require(threshold_val <= 2, "threshold too high")

  self.owners.push(a)
  self.owners.push(b)
  self.is_owner[a] = true
  self.is_owner[b] = true
  self.owner_count = 2
  self.threshold = threshold_val
}
```

## audit check

flag multisig/governance logic where:

```text
owners/signers can contain duplicates
threshold can be zero
threshold can exceed unique signer count
vote_count increments without checking signer is unique
removed signer can still vote
owner_count and owner list can drift apart
```

also check:

```text
set_threshold validates the new threshold
add_owner rejects duplicates
remove_owner updates owner_count and threshold constraints
execute checks vote_count >= threshold
proposal cannot be executed twice
```

## rule of thumb

```text
quorum math must use unique signers, not list length if duplicates are possible.
always enforce 0 < threshold <= unique_signer_count.
```

## severity

critical if duplicate signers or bad thresholds allow unauthorized execution, bridge minting, or treasury movement.

high if governance can become permanently stuck or too easy to pass.

medium if only metadata or non-critical settings are affected.

# opbr-019: pause / emergency bypass

## summary

pause and emergency controls must cover every path that can move funds or change critical state.

this bug happens when one function checks `paused`, but another equivalent path does not.

examples:

```text
transfer is paused, but pull still works
withdraw is paused, but claim still pays out
swap is paused, but remove_liquidity still drains reserves
emergency_withdraw can take user deposits instead of only protocol fees
```

## where to look

check functions like:

- `transfer`
- `pull`
- `withdraw`
- `claim`
- `redeem`
- `swap`
- `remove_liquidity`
- `admin_withdraw`
- `emergency_withdraw`
- `pause`
- `unpause`

look for state like:

```aml
paused: bool
owner: address
treasury: int
total_locked: int
deposits: map[address]int
```

## why it is dangerous

a pause that does not pause all value-moving paths gives a false sense of safety.

an emergency function that ignores user accounting can drain funds that belong to users.

## bad example

```aml
state {
  paused: bool
  balances: map[address]int
  grants: map[address]map[address]int
}

fn transfer(to: address, amount: int): bool {
  require(!self.paused, "paused")
  require(amount > 0, "invalid amount")

  let bal = self.balances[caller]
  require(bal >= amount, "insufficient balance")

  self.balances[caller] = bal - amount
  self.balances[to] += amount
  return true
}

fn pull(from: address, to: address, amount: int): bool {
  require(amount > 0, "invalid amount")

  let allowed = self.grants[from][caller]
  require(allowed >= amount, "not allowed")

  let bal = self.balances[from]
  require(bal >= amount, "insufficient balance")

  self.grants[from][caller] = allowed - amount
  self.balances[from] = bal - amount
  self.balances[to] += amount
  return true
}
```

## fixed example

```aml
state {
  paused: bool
  balances: map[address]int
  grants: map[address]map[address]int
}

fn transfer(to: address, amount: int): bool {
  require(!self.paused, "paused")
  require(amount > 0, "invalid amount")

  let bal = self.balances[caller]
  require(bal >= amount, "insufficient balance")

  self.balances[caller] = bal - amount
  self.balances[to] += amount
  return true
}

fn pull(from: address, to: address, amount: int): bool {
  require(!self.paused, "paused")
  require(amount > 0, "invalid amount")

  let allowed = self.grants[from][caller]
  require(allowed >= amount, "not allowed")

  let bal = self.balances[from]
  require(bal >= amount, "insufficient balance")

  self.grants[from][caller] = allowed - amount
  self.balances[from] = bal - amount
  self.balances[to] += amount
  return true
}
```

## emergency withdraw check

bad:

```aml
fn emergency_withdraw(amount: int): bool {
  require(caller == self.owner, "not owner")
  require(amount > 0, "invalid amount")

  require(transfer(caller, amount), "transfer failed")
  return true
}
```

better:

```aml
fn emergency_withdraw_fees(amount: int): bool {
  require(caller == self.owner, "not owner")
  require(amount > 0, "invalid amount")
  require(self.protocol_fees >= amount, "insufficient fees")

  self.protocol_fees -= amount
  require(transfer(caller, amount), "transfer failed")
  return true
}
```

## audit check

flag pause/emergency logic where:

```text
only one transfer path checks paused
pull/transfer_from bypasses pause
withdraw is paused but claim/redeem is not
admin or emergency withdraw can touch user deposits
pause/unpause lacks auth
pause state is checked after state changes
```

also check:

```text
every value-moving function has a clear pause policy
read-only views remain available if intended
emergency withdraw only moves protocol-owned funds
events are emitted for pause/unpause/emergency actions
```

## rule of thumb

```text
list every function that can move value or change critical accounting.
decide whether pause should block it.
then verify the check exists on every path.
```

## severity

critical if pause bypass or emergency withdraw can drain user funds.

high if a system cannot be safely stopped during an incident.

medium if only non-critical operations bypass pause.

# opbr-020: unbounded loops / resource DoS

## summary

loops over user-controlled or ever-growing data can make a function too expensive or impossible to execute.

in Ethereum this is gas DoS. in AML programs, treat it as resource or execution DoS.

bad patterns:

```aml
for i in 0..n {
  ...
}
```

where `n` is controlled by a user or grows without a cap.

## where to look

check functions like:

- `batch_transfer`
- `batch_vote`
- `batch_claim`
- `distribute_rewards`
- `settle_all`
- `cleanup`
- `remove_owner`
- `execute_all`
- `harvest`

look for state like:

```aml
owners: list[address]
users: list[address]
stakers: list[address]
proposal_count: int
reward_count: int
```

## why it is dangerous

an attacker can make a function process too many items.

this can lead to:

```text
withdrawals become impossible
reward distribution gets stuck
governance execution cannot finish
cleanup can never complete
one user's data makes everyone else's function fail
```

## bad example

```aml
state {
  users: list[address]
  rewards: map[address]int
}

fn distribute_rewards(amount_each: int): bool {
  require(amount_each > 0, "invalid amount")

  for i in 0..len(self.users) {
    let user = self.users[i]
    self.rewards[user] += amount_each
  }

  return true
}
```

why bad:

```text
self.users can grow forever.
distribute_rewards eventually becomes too expensive or impossible.
```

## fixed example: bounded batch

```aml
state {
  users: list[address]
  rewards: map[address]int
}

fn distribute_rewards(start: int, limit: int, amount_each: int): int {
  require(start >= 0, "invalid start")
  require(limit > 0, "invalid limit")
  require(limit <= 100, "limit too high")
  require(amount_each > 0, "invalid amount")

  let end = start + limit
  if end > len(self.users) {
    end = len(self.users)
  }

  for i in start..end {
    let user = self.users[i]
    self.rewards[user] += amount_each
  }

  return end
}
```

## better pattern: pull over push

instead of looping over every user, let each user claim their own amount.

```aml
state {
  reward_per_user: int
  claimed: map[address]bool
}

fn claim_reward(): bool {
  require(!self.claimed[caller], "already claimed")
  require(self.reward_per_user > 0, "no reward")

  self.claimed[caller] = true
  require(transfer(caller, self.reward_per_user), "transfer failed")
  return true
}
```

## audit check

flag loops where:

```text
loop bound is user-controlled
loop bound is len(list) and list can grow forever
batch size has no max
one call tries to process all users/proposals/rewards
loop performs transfer(...) or call(...)
cleanup requires iterating over all historical ids
```

also check:

```text
limit parameter has a hard cap
progress can resume from a cursor/index
users can claim individually instead of being pushed to
loop cannot be forced to include invalid/stuck entries
```

## rule of thumb

```text
never require one transaction/call to process an unbounded list.
use bounded batches, cursors, or pull-based claims.
```

## severity

high if unbounded loops can freeze withdrawals, claims, governance, or reward distribution.

medium if only admin maintenance or cleanup can get stuck.

low if the loop is over a small fixed-size list.
