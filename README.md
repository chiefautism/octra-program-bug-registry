# octra program bug registry

octra program bug registry is a list of common bug patterns to check when auditing octra / aml programs.

this registry is about program logic, templates, and examples.

it is not a claim that aml itself is broken.

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

use unsigned types for asset quantities and ids

use signed types only for explicit deltas, changes, or values that are allowed to go below zero

if a program uses `int` for amounts, balances, reserves, deposits, supply, or allowances, the program must check that the value is non-negative or positive

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

## better fix

use an unsigned type for money values if available:

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

```rust
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

```rust
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

```rust
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

## better pattern

keep balances unsigned, and handle direction explicitly:

```rust
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

```rust
grants: map[address]map[address]int
allowances: map[address]map[address]int
```

and parameters like:

```rust
amount: int
amt: int
allowance: int
```

## why it is dangerous

a common allowance check is:

```rust
require(allowed >= amount, "insufficient allowance")
```

but if `amount` is negative, this check passes:

```text
0 >= -50
```

that is true.

then this line becomes dangerous:

```rust
self.grants[from][caller] = allowed - amount
```

because:

```text
0 - (-50) = 50
```

a negative pull amount can increase allowance instead of decreasing it.

## bad example

```rust
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

```rust
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

## better fix

use unsigned types for allowances and transfer amounts:

```rust
grants: map[address]map[address]u128
```

```rust
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

```rust
deposits: map[address]int
stakes: map[address]int
shares: map[address]int
total: int
total_locked: int
```

and parameters like:

```rust
amount: int
amt: int
shares: int
```

## why it is dangerous

a common withdraw check is:

```rust
require(user_balance >= amount, "insufficient balance")
```

but if `amount` is negative, this check passes:

```text
0 >= -100
```

that is true.

then this line becomes dangerous:

```rust
self.deposits[caller] = user_balance - amount
```

because:

```text
0 - (-100) = 100
```

the withdraw can increase the user's recorded deposit instead of reducing it.

## bad example

```rust
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

```rust
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

  transfer(caller, amount)
  return true
}
```

## better fix

use unsigned types for deposits and withdraw amounts:

```rust
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

  transfer(caller, amount)
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

```rust
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

```rust
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

```rust
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

```rust
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

```rust
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

```rust
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

```rust
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

```rust
total_supply: int
balances: map[address]int
```

or:

```rust
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

```rust
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

```rust
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

```rust
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

```rust
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

```rust
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

```rust
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

```rust
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

```rust
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

```rust
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

```rust
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

```rust
nonreentrant fn withdraw(amount: int): bool {
  require(amount > 0, "invalid amount")

  let bal = self.deposits[caller]
  require(bal >= amount, "insufficient deposit")

  self.deposits[caller] = bal - amount
  self.total_locked -= amount

  transfer(caller, amount)
  return true
}
```

## bad example: share minting ignores existing assets

```rust
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

```rust
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

```rust
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

```rust
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

```rust
fn set_fee(fee_bps: int): bool {
  require(origin == self.owner, "not owner")

  self.fee_bps = fee_bps
  return true
}
```

fixed:

```rust
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

```rust
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

```rust
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

```rust
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

```rust
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

```rust
transfer(to, amount)
call(target, value, data)
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

```rust
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

```rust
fn claim(id: int): bool {
  // check: user has something to claim
  require(self.claimable[id][caller] > 0, "nothing to claim")

  // read claim amount
  let amount = self.claimable[id][caller]

  // effect: clear claim before sending funds
  self.claimable[id][caller] = 0

  // interaction: external transfer happens last
  transfer(caller, amount)

  return true
}
```

## bad example: proposal marked executed after call

```rust
fn execute(id: int): bool {
  // check: enough approvals
  require(self.approvals[id] >= self.threshold, "not enough approvals")

  // check: proposal was not executed yet
  require(!self.executed[id], "already executed")

  // interaction: external call happens first
  // bad: proposal is still marked as not executed here
  call(self.target[id], self.value[id], self.data[id])

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

```rust
fn execute(id: int): bool {
  // check: enough approvals
  require(self.approvals[id] >= self.threshold, "not enough approvals")

  // check: proposal was not executed yet
  require(!self.executed[id], "already executed")

  // effect: mark executed before external call
  self.executed[id] = true

  // interaction: external call happens last
  call(self.target[id], self.value[id], self.data[id])

  return true
}
```

## bad example: reserves updated after payout

```rust
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

```rust
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
  transfer(caller, out)

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


# opbr-011: replay / double execution

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

```rust
executed: map[u128]bool
claimed: map[u128]map[address]bool
used_nonce: map[address]map[u128]bool
processed_message: map[bytes32]bool
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

```rust
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

```rust
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

```rust
fn claim_airdrop(drop_id: u128, amount: int, proof: bytes): bool {
  require(amount > 0, "invalid amount")
  require(verify_proof(drop_id, caller, amount, proof), "invalid proof")

  self.balances[caller] += amount
  self.total_supply += amount

  return true
}
```

## fixed example

```rust
state {
  claimed: map[u128]map[address]bool
  balances: map[address]int
  total_supply: int
}

fn claim_airdrop(drop_id: u128, amount: int, proof: bytes): bool {
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

```rust
fn use_signature(owner: address, amount: int, nonce: u128, sig: bytes): bool {
  require(amount > 0, "invalid amount")
  require(verify_sig(owner, amount, nonce, sig), "bad signature")

  self.balances[owner] -= amount
  self.balances[caller] += amount

  return true
}
```

## fixed example

```rust
state {
  used_nonce: map[address]map[u128]bool
  balances: map[address]int
}

fn use_signature(owner: address, amount: int, nonce: u128, sig: bytes): bool {
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
