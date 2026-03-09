# Simple NFT Smart Contract Tutorial

> **Course Module:** Soroban Crash Course
> **Follow-up to:** Mini Escrow Smart Contract Tutorial

---

## Prerequisites

Before starting this tutorial, you should have completed:

1. **Hello World Contract** - Basic Soroban contract structure
2. **Increment Contract** - Storage basics and counters
3. **Guest Book Contract** - Authentication and typed storage keys
4. **Task Manager Contract** - State updates and ownership checks
5. **Mini Escrow Contract** - Role-based authorization and multi-user workflows

### Required Knowledge

* Authentication with `require_auth()`
* Custom data structures using `#[contracttype]`
* Typed storage keys with enums
* Instance storage vs persistent storage
* Storage operations like `get`, `set`, `remove`, and `has`
* Writing and running Soroban tests
* Understanding ownership and balances

### Development Environment

Ensure you have installed:

* Rust toolchain
* Soroban CLI
* A code editor with Rust support
* A working Soroban project setup

---

## Learning Objectives

By the end of this lesson, you will be able to:

**Knowledge**

* Understand the core ideas behind an NFT contract
* Explain how token ownership is stored and updated
* Describe the difference between instance storage and persistent storage
* Understand admin-only minting and owner-only transfers

**Skills**

* Build a simple NFT contract in Soroban
* Store token owners and metadata
* Track balances per address
* Implement mint, transfer, burn, and read functions
* Write tests for NFT lifecycle operations

**Application**

* Design simple collectible contracts
* Build token-based ownership systems
* Apply storage namespaces for multiple data types
* Extend a basic NFT contract into more advanced collections later

---

## Introduction

### What Is a Simple NFT Contract?

An **NFT** is a **non-fungible token**, which means each token is unique.

Unlike a normal token where every unit is interchangeable, an NFT has its own identity.
For example:

* Token #1 is different from Token #2
* each token can have its own metadata
* each token has a specific owner

In this contract, we are building a **simple NFT** with these features:

* one admin initializes the contract
* only the admin can mint new NFTs
* tokens have sequential IDs
* each token has an owner
* each token has a metadata URI
* owners can transfer or burn their tokens
* balances are tracked per address

### Why Build This?

This contract combines several important Soroban patterns:

1. **Initialization logic** - a contract setup step that can only happen once
2. **Admin-only actions** - privileged minting
3. **Ownership mapping** - who owns which token
4. **Metadata storage** - each token has a URI
5. **Balance tracking** - how many NFTs each user owns
6. **State removal** - burning deletes token data

This is a great next step because it combines ideas from all your earlier tutorials into one contract.

### Real-World Applications

The patterns in this contract apply to:

* digital collectibles
* certificates and credentials
* event tickets
* proof of ownership systems
* game assets
* membership NFTs
* on-chain identity badges

---

## Conceptual Overview

### Building on Previous Lessons

This tutorial combines earlier concepts:

* **Guest Book** taught typed storage keys
* **Task Manager** taught ownership checks and updates
* **Mini Escrow** taught role-based permissions
* **Simple NFT** adds multiple storage namespaces and token lifecycle management

### The New Challenges

This contract answers important questions:

* How do we initialize a contract only once?
* How do we make minting admin-only?
* How do we store ownership for many tokens?
* How do we keep a balance count for each address?
* How do we remove token data safely when burning?
* How do we separate long-lived token data from contract config?

### Our Solution

We will build a contract that:

1. Stores one admin in instance storage
2. Stores a `NextId` counter for minting
3. Stores total supply
4. Stores token owners in persistent storage
5. Stores token metadata URIs in persistent storage
6. Stores balances per address in persistent storage
7. Restricts minting to the admin
8. Restricts transfer and burn to the token owner

### Comparison with Earlier Contracts

| Feature            | Task Manager     | Mini Escrow         | Simple NFT             |
| ------------------ | ---------------- | ------------------- | ---------------------- |
| Main resource      | Task             | Escrow              | NFT token              |
| Roles              | Owner            | Client + Freelancer | Admin + Token Owners   |
| ID system          | Sequential       | Sequential          | Sequential token IDs   |
| State changes      | Status update    | Release / Refund    | Mint / Transfer / Burn |
| Metadata           | Description      | Amount + parties    | Token URI              |
| Balance tracking   | No               | No                  | Yes                    |
| Storage namespaces | One main pattern | One main pattern    | Multiple mappings      |

---

## Contract Architecture

### Data Model

This contract uses one main storage-key enum and several storage mappings.

---

### 1. DataKey Enum

```rust
pub enum DataKey {
    // Instance storage
    Admin,
    NextId,
    Supply,

    // Persistent storage namespaces
    Owner(u32),
    TokenUri(u32),
    Balance(Address),
}
```

This enum is the backbone of the contract.

It defines every piece of data we store.

#### Instance storage keys

```rust
Admin
NextId
Supply
```

These belong to the contract itself:

* `Admin` - who controls minting
* `NextId` - the next token ID to mint
* `Supply` - how many NFTs currently exist

#### Persistent storage keys

```rust
Owner(u32)
TokenUri(u32)
Balance(Address)
```

These are namespaced records:

* `Owner(token_id)` → owner of a token
* `TokenUri(token_id)` → metadata URI for a token
* `Balance(address)` → number of NFTs owned by that address

---

### Why use one enum for many namespaces?

Because typed keys keep the contract organized and safe.

Instead of using strings like:

```rust
"ADMIN"
"OWNER_1"
"TOKEN_URI_1"
"BALANCE_ALICE"
```

we use:

```rust
DataKey::Admin
DataKey::Owner(1)
DataKey::TokenUri(1)
DataKey::Balance(alice)
```

This is:

* safer
* easier to read
* easier to refactor
* better for autocomplete

---

### NFT Lifecycle

```text
init
 │
 ▼
Contract initialized with admin

mint
 │
 ▼
Token created with:
- token_id
- owner
- token_uri

transfer
 │
 ▼
Ownership moves to another address

burn
 │
 ▼
Token state removed
Supply decreases
Owner balance decreases
```

---

### Storage Layout Example

After minting two NFTs, storage might look like this:

```text
Instance Storage
├── DataKey::Admin -> AdminAddress
├── DataKey::NextId -> 3
└── DataKey::Supply -> 2

Persistent Storage
├── DataKey::Owner(1) -> Alice
├── DataKey::TokenUri(1) -> "ipfs://token-1"
├── DataKey::Owner(2) -> Bob
├── DataKey::TokenUri(2) -> "ipfs://token-2"
├── DataKey::Balance(Alice) -> 1
└── DataKey::Balance(Bob) -> 1
```

---

### Contract Functions

| Function                       | Purpose                       | Authorization      | Returns        |
| ------------------------------ | ----------------------------- | ------------------ | -------------- |
| `init(admin)`                  | Initialize contract once      | Admin auth         | Nothing        |
| `mint(admin, to, token_uri)`   | Mint new token                | Admin-only         | `u32` token ID |
| `transfer(from, to, token_id)` | Transfer token                | Current owner only | Nothing        |
| `burn(owner, token_id)`        | Burn token                    | Current owner only | Nothing        |
| `owner_of(token_id)`           | Get token owner               | Public             | `Address`      |
| `token_uri(token_id)`          | Get token metadata URI        | Public             | `String`       |
| `balance_of(owner)`            | Get NFT count for address     | Public             | `u32`          |
| `total_supply()`               | Get number of existing tokens | Public             | `u32`          |

---

## Step-by-Step Build Section

---

### Step 1: Imports and Setup

```rust
#![no_std]

use soroban_sdk::{contract, contractimpl, contracttype, Address, Env, String};
```

### What this means

* `#![no_std]` is required for Soroban smart contracts
* `contract` marks the contract type
* `contractimpl` marks the contract implementation block
* `contracttype` allows custom types to work with Soroban storage
* `Address` represents blockchain addresses
* `Env` gives access to storage and runtime context
* `String` is used for token metadata URIs

This is a normal Soroban contract setup, but now we are also working with string metadata.

---

### Step 2: Define the Storage Keys

```rust
#[contracttype]
#[derive(Clone, Debug, PartialEq)]
pub enum DataKey {
    // Instance storage
    Admin,
    NextId,
    Supply,

    // Persistent storage namespaces
    Owner(u32),
    TokenUri(u32),
    Balance(Address),
}
```

### Why this enum matters

This enum lets us model different kinds of stored data in one structured way.

### Key groups

#### Contract configuration

```rust
Admin
NextId
Supply
```

These are contract-level values.

#### Token and account records

```rust
Owner(u32)
TokenUri(u32)
Balance(Address)
```

These are per-token or per-user values.

### Why derive `Clone`, `Debug`, and `PartialEq`?

* `Clone` allows copying when needed
* `Debug` helps with debugging and tests
* `PartialEq` allows comparison if needed

---

### Step 3: Create the Contract Shell

```rust
#[contract]
pub struct SimpleNft;

#[contractimpl]
impl SimpleNft {
    // functions go here
}
```

This is the normal Soroban contract pattern.

The struct is empty because the state lives in storage, not inside Rust fields.

---

### Step 4: Implement `init`

```rust
pub fn init(env: Env, admin: Address) {
    let existing: Option<Address> = env.storage().instance().get(&DataKey::Admin);
    if existing.is_some() {
        panic!("Already initialized");
    }

    admin.require_auth();

    env.storage().instance().set(&DataKey::Admin, &admin);
    env.storage().instance().set(&DataKey::NextId, &1u32);
    env.storage().instance().set(&DataKey::Supply, &0u32);
}
```

This function initializes the NFT contract.

---

#### Part A: Prevent re-initialization

```rust
let existing: Option<Address> = env.storage().instance().get(&DataKey::Admin);
if existing.is_some() {
    panic!("Already initialized");
}
```

This checks if an admin is already stored.

If yes, initialization must fail.

### Why is this important?

Because initialization should only happen once.

Without this check, someone could reassign the admin later and take control of the contract.

---

#### Part B: Authenticate the admin

```rust
admin.require_auth();
```

The supplied admin must authorize the call.

This proves that the real admin address is the one being set.

---

#### Part C: Store initial contract state

```rust
env.storage().instance().set(&DataKey::Admin, &admin);
env.storage().instance().set(&DataKey::NextId, &1u32);
env.storage().instance().set(&DataKey::Supply, &0u32);
```

This sets:

* the admin
* the next token ID to `1`
* the initial supply to `0`

### Why start `NextId` at 1?

This is a design choice that keeps token IDs human-friendly.

The first minted token becomes token `1`.

---

### Step 5: Implement `mint`

```rust
pub fn mint(env: Env, admin: Address, to: Address, token_uri: String) -> u32 {
    Self::require_admin(&env, &admin);

    let mut next_id: u32 = env.storage().instance().get(&DataKey::NextId).unwrap_or(1);

    if env.storage().persistent().has(&DataKey::Owner(next_id)) {
        panic!("Token ID already exists");
    }

    env.storage()
        .persistent()
        .set(&DataKey::Owner(next_id), &to);
    env.storage()
        .persistent()
        .set(&DataKey::TokenUri(next_id), &token_uri);

    Self::inc_balance(&env, &to);

    let mut supply: u32 = env.storage().instance().get(&DataKey::Supply).unwrap_or(0);
    supply += 1;
    env.storage().instance().set(&DataKey::Supply, &supply);

    next_id += 1;
    env.storage().instance().set(&DataKey::NextId, &next_id);

    next_id - 1
}
```

This function creates a new NFT.

---

#### Part A: Admin-only check

```rust
Self::require_admin(&env, &admin);
```

This is an internal helper that:

1. requires admin authentication
2. checks that the supplied admin matches the stored admin

This is stronger than just calling `require_auth()` alone.

---

#### Part B: Load the next token ID

```rust
let mut next_id: u32 = env.storage().instance().get(&DataKey::NextId).unwrap_or(1);
```

This reads the next available token ID.

If missing, it defaults to `1`, though in normal flow it should already exist after `init`.

---

#### Part C: Defensive existence check

```rust
if env.storage().persistent().has(&DataKey::Owner(next_id)) {
    panic!("Token ID already exists");
}
```

This makes sure we do not overwrite an existing token.

### Why is this defensive?

Because in the normal flow, `NextId` should already prevent collisions.
But checking again makes the logic safer.

---

#### Part D: Store owner and metadata

```rust
env.storage()
    .persistent()
    .set(&DataKey::Owner(next_id), &to);
env.storage()
    .persistent()
    .set(&DataKey::TokenUri(next_id), &token_uri);
```

This creates the NFT’s main state:

* owner
* metadata URI

### Why use persistent storage here?

Because token ownership and token metadata should survive independently as token records.

---

#### Part E: Increase the recipient's balance

```rust
Self::inc_balance(&env, &to);
```

If Alice receives a new NFT, her balance should go up by 1.

---

#### Part F: Increase total supply

```rust
let mut supply: u32 = env.storage().instance().get(&DataKey::Supply).unwrap_or(0);
supply += 1;
env.storage().instance().set(&DataKey::Supply, &supply);
```

This tracks how many tokens currently exist.

Minting adds one token, so supply increases.

---

#### Part G: Increment `NextId`

```rust
next_id += 1;
env.storage().instance().set(&DataKey::NextId, &next_id);
```

This prepares the next token ID for future mints.

---

#### Part H: Return minted token ID

```rust
next_id - 1
```

Because `next_id` was incremented, the minted token’s actual ID is the previous value.

---

### Step 6: Implement `transfer`

```rust
pub fn transfer(env: Env, from: Address, to: Address, token_id: u32) {
    from.require_auth();

    let owner = Self::owner_of(env.clone(), token_id);
    if owner != from {
        panic!("Not token owner");
    }

    env.storage()
        .persistent()
        .set(&DataKey::Owner(token_id), &to);

    Self::dec_balance(&env, &from);
    Self::inc_balance(&env, &to);
}
```

This function transfers an NFT from one address to another.

---

#### Part A: Authenticate sender

```rust
from.require_auth();
```

The `from` address must authorize the transfer.

---

#### Part B: Check ownership

```rust
let owner = Self::owner_of(env.clone(), token_id);
if owner != from {
    panic!("Not token owner");
}
```

This ensures only the current owner can transfer the NFT.

This is an ownership rule similar to earlier tutorials, but now it applies to tokens.

---

#### Part C: Update owner mapping

```rust
env.storage()
    .persistent()
    .set(&DataKey::Owner(token_id), &to);
```

The NFT’s owner changes from `from` to `to`.

---

#### Part D: Update balances

```rust
Self::dec_balance(&env, &from);
Self::inc_balance(&env, &to);
```

A transfer should affect balances:

* old owner loses one
* new owner gains one

If you forget this, ownership and balance counts become inconsistent.

---

### Step 7: Implement `burn`

```rust
pub fn burn(env: Env, owner: Address, token_id: u32) {
    owner.require_auth();

    let current_owner = Self::owner_of(env.clone(), token_id);
    if current_owner != owner {
        panic!("Not token owner");
    }

    env.storage().persistent().remove(&DataKey::Owner(token_id));
    env.storage()
        .persistent()
        .remove(&DataKey::TokenUri(token_id));

    Self::dec_balance(&env, &owner);

    let mut supply: u32 = env.storage().instance().get(&DataKey::Supply).unwrap_or(0);
    if supply == 0 {
        panic!("Supply underflow");
    }
    supply -= 1;
    env.storage().instance().set(&DataKey::Supply, &supply);
}
```

This permanently removes an NFT.

---

#### Part A: Authenticate owner

```rust
owner.require_auth();
```

Only the owner can burn the token.

---

#### Part B: Check current ownership

```rust
let current_owner = Self::owner_of(env.clone(), token_id);
if current_owner != owner {
    panic!("Not token owner");
}
```

This prevents unauthorized burning.

---

#### Part C: Remove token data

```rust
env.storage().persistent().remove(&DataKey::Owner(token_id));
env.storage()
    .persistent()
    .remove(&DataKey::TokenUri(token_id));
```

Burning deletes the token’s state:

* no owner
* no metadata URI

After this, the token no longer exists.

---

#### Part D: Decrease balance

```rust
Self::dec_balance(&env, &owner);
```

The owner loses one NFT from their balance.

---

#### Part E: Decrease supply

```rust
let mut supply: u32 = env.storage().instance().get(&DataKey::Supply).unwrap_or(0);
if supply == 0 {
    panic!("Supply underflow");
}
supply -= 1;
env.storage().instance().set(&DataKey::Supply, &supply);
```

Total supply must go down when a token is burned.

The underflow check is defensive programming.

---

### Step 8: Implement Read Functions

#### `owner_of`

```rust
pub fn owner_of(env: Env, token_id: u32) -> Address {
    env.storage()
        .persistent()
        .get(&DataKey::Owner(token_id))
        .expect("Token not found")
}
```

Returns the owner of a token.

If the token does not exist, it panics.

---

#### `token_uri`

```rust
pub fn token_uri(env: Env, token_id: u32) -> String {
    env.storage()
        .persistent()
        .get(&DataKey::TokenUri(token_id))
        .expect("Token not found")
}
```

Returns the metadata URI for a token.

---

#### `balance_of`

```rust
pub fn balance_of(env: Env, owner: Address) -> u32 {
    env.storage()
        .persistent()
        .get(&DataKey::Balance(owner))
        .unwrap_or(0)
}
```

Returns how many NFTs an address owns.

If no balance is stored, return `0`.

---

#### `total_supply`

```rust
pub fn total_supply(env: Env) -> u32 {
    env.storage().instance().get(&DataKey::Supply).unwrap_or(0)
}
```

Returns how many NFTs currently exist.

---

### Step 9: Implement Internal Helpers

These helpers keep the contract cleaner and avoid repeating logic.

---

#### `require_admin`

```rust
fn require_admin(env: &Env, admin: &Address) {
    admin.require_auth();

    let stored: Address = env
        .storage()
        .instance()
        .get(&DataKey::Admin)
        .expect("Not initialized");

    if &stored != admin {
        panic!("Not admin");
    }
}
```

This helper does two things:

1. checks that the caller authorized the action
2. checks that the caller is the stored admin

This is used by `mint`.

---

#### `inc_balance`

```rust
fn inc_balance(env: &Env, owner: &Address) {
    let key = DataKey::Balance(owner.clone());
    let mut bal: u32 = env.storage().persistent().get(&key).unwrap_or(0);
    bal += 1;
    env.storage().persistent().set(&key, &bal);
}
```

This increases an address balance by 1.

Used when:

* minting
* receiving a transfer

---

#### `dec_balance`

```rust
fn dec_balance(env: &Env, owner: &Address) {
    let key = DataKey::Balance(owner.clone());
    let mut bal: u32 = env.storage().persistent().get(&key).unwrap_or(0);
    if bal == 0 {
        panic!("Balance underflow");
    }
    bal -= 1;
    env.storage().persistent().set(&key, &bal);
}
```

This decreases an address balance by 1.

Used when:

* sending a transfer
* burning a token

The underflow check prevents invalid negative balances.

---

## Complete Contract Code

```rust
#![no_std]

use soroban_sdk::{contract, contractimpl, contracttype, Address, Env, String};

/// A minimal educational NFT contract.
/// Features:
/// - Admin-only minting
/// - Sequential token IDs
/// - owner_of / transfer / burn
/// - token_uri metadata
/// - balance tracking
///
/// Notes:
/// - This is for learning patterns (storage, auth, state checks).
/// - For production NFTs, prefer audited implementations (e.g., OpenZeppelin).

#[contracttype]
#[derive(Clone, Debug, PartialEq)]
pub enum DataKey {
    // Instance storage
    Admin,
    NextId,
    Supply,

    // Persistent storage namespaces
    Owner(u32),       // token_id -> Address
    TokenUri(u32),    // token_id -> String
    Balance(Address), // owner -> u32
}

#[contract]
pub struct SimpleNft;

#[contractimpl]
impl SimpleNft {
    /// Initialize contract with an admin.
    /// Can only be called once.
    pub fn init(env: Env, admin: Address) {
        let existing: Option<Address> = env.storage().instance().get(&DataKey::Admin);
        if existing.is_some() {
            panic!("Already initialized");
        }

        admin.require_auth();

        env.storage().instance().set(&DataKey::Admin, &admin);
        env.storage().instance().set(&DataKey::NextId, &1u32);
        env.storage().instance().set(&DataKey::Supply, &0u32);
    }

    /// Admin-only mint.
    /// Returns the newly minted token_id.
    pub fn mint(env: Env, admin: Address, to: Address, token_uri: String) -> u32 {
        Self::require_admin(&env, &admin);

        let mut next_id: u32 = env.storage().instance().get(&DataKey::NextId).unwrap_or(1);

        if env.storage().persistent().has(&DataKey::Owner(next_id)) {
            panic!("Token ID already exists");
        }

        env.storage()
            .persistent()
            .set(&DataKey::Owner(next_id), &to);
        env.storage()
            .persistent()
            .set(&DataKey::TokenUri(next_id), &token_uri);

        Self::inc_balance(&env, &to);

        let mut supply: u32 = env.storage().instance().get(&DataKey::Supply).unwrap_or(0);
        supply += 1;
        env.storage().instance().set(&DataKey::Supply, &supply);

        next_id += 1;
        env.storage().instance().set(&DataKey::NextId, &next_id);

        next_id - 1
    }

    /// Transfer token to another address.
    /// Only current owner can transfer.
    pub fn transfer(env: Env, from: Address, to: Address, token_id: u32) {
        from.require_auth();

        let owner = Self::owner_of(env.clone(), token_id);
        if owner != from {
            panic!("Not token owner");
        }

        env.storage()
            .persistent()
            .set(&DataKey::Owner(token_id), &to);

        Self::dec_balance(&env, &from);
        Self::inc_balance(&env, &to);
    }

    /// Burn a token.
    /// Only current owner can burn.
    pub fn burn(env: Env, owner: Address, token_id: u32) {
        owner.require_auth();

        let current_owner = Self::owner_of(env.clone(), token_id);
        if current_owner != owner {
            panic!("Not token owner");
        }

        env.storage().persistent().remove(&DataKey::Owner(token_id));
        env.storage()
            .persistent()
            .remove(&DataKey::TokenUri(token_id));

        Self::dec_balance(&env, &owner);

        let mut supply: u32 = env.storage().instance().get(&DataKey::Supply).unwrap_or(0);
        if supply == 0 {
            panic!("Supply underflow");
        }
        supply -= 1;
        env.storage().instance().set(&DataKey::Supply, &supply);
    }

    /// Read token owner.
    pub fn owner_of(env: Env, token_id: u32) -> Address {
        env.storage()
            .persistent()
            .get(&DataKey::Owner(token_id))
            .expect("Token not found")
    }

    /// Read token URI.
    pub fn token_uri(env: Env, token_id: u32) -> String {
        env.storage()
            .persistent()
            .get(&DataKey::TokenUri(token_id))
            .expect("Token not found")
    }

    /// Read balance for an address.
    pub fn balance_of(env: Env, owner: Address) -> u32 {
        env.storage()
            .persistent()
            .get(&DataKey::Balance(owner))
            .unwrap_or(0)
    }

    /// Read total supply (number of existing tokens).
    pub fn total_supply(env: Env) -> u32 {
        env.storage().instance().get(&DataKey::Supply).unwrap_or(0)
    }

    fn require_admin(env: &Env, admin: &Address) {
        admin.require_auth();

        let stored: Address = env
            .storage()
            .instance()
            .get(&DataKey::Admin)
            .expect("Not initialized");

        if &stored != admin {
            panic!("Not admin");
        }
    }

    fn inc_balance(env: &Env, owner: &Address) {
        let key = DataKey::Balance(owner.clone());
        let mut bal: u32 = env.storage().persistent().get(&key).unwrap_or(0);
        bal += 1;
        env.storage().persistent().set(&key, &bal);
    }

    fn dec_balance(env: &Env, owner: &Address) {
        let key = DataKey::Balance(owner.clone());
        let mut bal: u32 = env.storage().persistent().get(&key).unwrap_or(0);
        if bal == 0 {
            panic!("Balance underflow");
        }
        bal -= 1;
        env.storage().persistent().set(&key, &bal);
    }
}
```

---

## Testing Section

Now let’s walk through the tests one by one.

---

### Test Module Setup

```rust
mod test;
#![cfg(test)]

use super::*;
use soroban_sdk::{testutils::Address as _, Address, Env, String};
```

### What this setup does

* `mod test;` means the tests live in a separate module
* `#![cfg(test)]` means the code only compiles during tests
* `use super::*;` imports the contract code
* `Address::generate(&env)` creates test addresses
* `env.mock_all_auths()` bypasses manual auth setup during tests

---

### Test 1: `init_and_mint_and_read_works`

```rust
#[test]
fn init_and_mint_and_read_works() {
    let env = Env::default();
    env.mock_all_auths();

    let contract_id = env.register(SimpleNft, ());
    let client = SimpleNftClient::new(&env, &contract_id);

    let admin = Address::generate(&env);
    let alice = Address::generate(&env);

    client.init(&admin);

    let uri = String::from_str(&env, "ipfs://token-1");
    let id = client.mint(&admin, &alice, &uri);
    assert_eq!(id, 1);

    assert_eq!(client.total_supply(), 1);
    assert_eq!(client.balance_of(&alice), 1);

    let owner = client.owner_of(&1);
    assert_eq!(owner, alice);

    let stored_uri = client.token_uri(&1);
    assert_eq!(stored_uri, uri);
}
```

### What this test verifies

* contract initializes correctly
* admin can mint
* first token gets ID `1`
* total supply becomes `1`
* Alice’s balance becomes `1`
* owner is stored correctly
* token URI is stored correctly

This is the main happy-path test for setup, minting, and reading.

---

### Test 2: `init_only_once`

```rust
#[test]
#[should_panic(expected = "Already initialized")]
fn init_only_once() {
    let env = Env::default();
    env.mock_all_auths();

    let contract_id = env.register(SimpleNft, ());
    let client = SimpleNftClient::new(&env, &contract_id);

    let admin = Address::generate(&env);

    client.init(&admin);
    client.init(&admin);
}
```

### What this tests

* initialization works once
* the second initialization attempt panics

This protects the contract from admin replacement after deployment.

---

### Test 3: `mint_requires_admin`

```rust
#[test]
#[should_panic(expected = "Not admin")]
fn mint_requires_admin() {
    let env = Env::default();
    env.mock_all_auths();

    let contract_id = env.register(SimpleNft, ());
    let client = SimpleNftClient::new(&env, &contract_id);

    let admin = Address::generate(&env);
    let not_admin = Address::generate(&env);
    let alice = Address::generate(&env);

    client.init(&admin);

    client.mint(&not_admin, &alice, &String::from_str(&env, "ipfs://x"));
}
```

### What this verifies

* only the stored admin can mint
* any other address causes a panic

This tests the `require_admin` helper.

---

### Test 4: `transfer_updates_owner_and_balances`

```rust
#[test]
fn transfer_updates_owner_and_balances() {
    let env = Env::default();
    env.mock_all_auths();

    let contract_id = env.register(SimpleNft, ());
    let client = SimpleNftClient::new(&env, &contract_id);

    let admin = Address::generate(&env);
    let alice = Address::generate(&env);
    let bob = Address::generate(&env);

    client.init(&admin);

    let id = client.mint(&admin, &alice, &String::from_str(&env, "ipfs://token-1"));
    assert_eq!(id, 1);

    assert_eq!(client.balance_of(&alice), 1);
    assert_eq!(client.balance_of(&bob), 0);

    client.transfer(&alice, &bob, &1);

    assert_eq!(client.owner_of(&1), bob);
    assert_eq!(client.balance_of(&alice), 0);
    assert_eq!(client.balance_of(&bob), 1);
}
```

### What this tests

* current owner can transfer the NFT
* ownership moves to Bob
* Alice’s balance decreases
* Bob’s balance increases

This is a very important consistency test.

---

### Test 5: `transfer_requires_owner`

```rust
#[test]
#[should_panic(expected = "Not token owner")]
fn transfer_requires_owner() {
    let env = Env::default();
    env.mock_all_auths();

    let contract_id = env.register(SimpleNft, ());
    let client = SimpleNftClient::new(&env, &contract_id);

    let admin = Address::generate(&env);
    let alice = Address::generate(&env);
    let bob = Address::generate(&env);

    client.init(&admin);
    client.mint(&admin, &alice, &String::from_str(&env, "ipfs://token-1"));

    client.transfer(&bob, &bob, &1);
}
```

### What this verifies

* non-owners cannot transfer a token they do not own
* the function panics with `"Not token owner"`

This tests owner-only transfer enforcement.

---

### Test 6: `burn_removes_token_and_updates_supply`

```rust
#[test]
fn burn_removes_token_and_updates_supply() {
    let env = Env::default();
    env.mock_all_auths();

    let contract_id = env.register(SimpleNft, ());
    let client = SimpleNftClient::new(&env, &contract_id);

    let admin = Address::generate(&env);
    let alice = Address::generate(&env);

    client.init(&admin);
    client.mint(&admin, &alice, &String::from_str(&env, "ipfs://token-1"));

    assert_eq!(client.total_supply(), 1);
    assert_eq!(client.balance_of(&alice), 1);

    client.burn(&alice, &1);

    assert_eq!(client.total_supply(), 0);
    assert_eq!(client.balance_of(&alice), 0);
}
```

### What this tests

* owner can burn the token
* total supply decreases
* owner balance decreases

This confirms that burning updates aggregate state correctly.

---

### Test 7: `owner_of_missing_token_panics`

```rust
#[test]
#[should_panic(expected = "Token not found")]
fn owner_of_missing_token_panics() {
    let env = Env::default();
    env.mock_all_auths();

    let contract_id = env.register(SimpleNft, ());
    let client = SimpleNftClient::new(&env, &contract_id);

    client.owner_of(&999);
}
```

### What this tests

* reading the owner of a missing token should panic
* panic message should contain `"Token not found"`

This verifies missing-token handling in the read path.

---

## Test-by-Test Summary

| Test                                    | Purpose                                                             |
| --------------------------------------- | ------------------------------------------------------------------- |
| `init_and_mint_and_read_works`          | Verify initialization, minting, ownership, URI, supply, and balance |
| `init_only_once`                        | Verify contract can only be initialized once                        |
| `mint_requires_admin`                   | Verify minting is admin-only                                        |
| `transfer_updates_owner_and_balances`   | Verify transfer updates ownership and balances                      |
| `transfer_requires_owner`               | Verify only current owner can transfer                              |
| `burn_removes_token_and_updates_supply` | Verify burning removes token effects from balances and supply       |
| `owner_of_missing_token_panics`         | Verify missing tokens panic on read                                 |

---

## Common Pitfalls

### Pitfall 1: Forgetting one-time initialization protection

**Incorrect**

```rust
pub fn init(env: Env, admin: Address) {
    admin.require_auth();
    env.storage().instance().set(&DataKey::Admin, &admin);
}
```

### Problem

This allows re-initialization, which could let someone replace the admin.

### Fix

```rust
let existing: Option<Address> = env.storage().instance().get(&DataKey::Admin);
if existing.is_some() {
    panic!("Already initialized");
}
```

---

### Pitfall 2: Using only `require_auth()` for minting

**Incorrect**

```rust
pub fn mint(env: Env, admin: Address, to: Address, token_uri: String) -> u32 {
    admin.require_auth();
    // mint logic
}
```

### Problem

Any authenticated address could mint if you do not compare it with the stored admin.

### Fix

Use a helper like:

```rust
Self::require_admin(&env, &admin);
```

That checks both authentication and role.

---

### Pitfall 3: Updating ownership but forgetting balances

**Incorrect**

```rust
env.storage()
    .persistent()
    .set(&DataKey::Owner(token_id), &to);
```

### Problem

The token owner changes, but balances become wrong.

### Fix

Always update balances too:

```rust
Self::dec_balance(&env, &from);
Self::inc_balance(&env, &to);
```

---

### Pitfall 4: Burning without removing metadata

**Incorrect**

```rust
env.storage().persistent().remove(&DataKey::Owner(token_id));
```

### Problem

The token owner disappears, but the metadata URI still exists in storage.

### Fix

Remove both:

```rust
env.storage().persistent().remove(&DataKey::Owner(token_id));
env.storage().persistent().remove(&DataKey::TokenUri(token_id));
```

---

### Pitfall 5: Forgetting underflow checks

**Incorrect**

```rust
bal -= 1;
```

### Problem

If `bal` is zero, this creates invalid logic.

### Fix

```rust
if bal == 0 {
    panic!("Balance underflow");
}
bal -= 1;
```

The same idea applies to supply.

---

### Pitfall 6: Confusing instance storage and persistent storage

A common mistake is putting everything in one storage type without thinking about its role.

In this contract:

**Instance storage** is used for:

* admin
* next token ID
* total supply

**Persistent storage** is used for:

* token ownership
* token URIs
* balances

This separation keeps the design organized.

---

## Practice Exercises

### Exercise 1: Add `admin()` Getter

Create a public function:

```rust
pub fn admin(env: Env) -> Address
```

It should return the stored admin.

---

### Exercise 2: Add `exists(token_id)`

Create:

```rust
pub fn exists(env: Env, token_id: u32) -> bool
```

It should return `true` if the token exists and `false` otherwise.

Hint: use `has()` on the owner mapping.

---

### Exercise 3: Add admin-only burn

Modify the contract so that either:

* the token owner can burn, or
* the admin can burn any token

You will need to adjust the authorization logic carefully.

---

### Exercise 4: Prevent empty metadata URI

In `mint`, validate that the token URI is not empty.

You can choose to:

* panic if empty
* add more advanced validation later

---

### Exercise 5: Add `tokens_minted_count`

Right now `Supply` represents current existing tokens, not lifetime minted tokens.

Design a new counter that tracks how many NFTs have ever been minted, even if some are later burned.

---

## Assessment

### Question 1: Why does this contract use both instance storage and persistent storage?

<details>
<summary>Click to see answer</summary>

The contract uses instance storage for contract-level configuration and counters such as the admin, next token ID, and total supply. It uses persistent storage for token records and balances because those are namespaced per token or per address. This separation makes the storage model cleaner and easier to reason about.

</details>

---

### Question 2: Why is `require_admin` better than only calling `admin.require_auth()` in `mint`?

<details>
<summary>Click to see answer</summary>

`admin.require_auth()` only proves that the provided address authorized the call. It does not prove that this address is the stored admin of the contract. `require_admin` combines authentication with a role check against storage, so only the real stored admin can mint.

</details>

---

### Question 3: Why must `transfer` update both the owner mapping and balances?

<details>
<summary>Click to see answer</summary>

The owner mapping tells us who owns a specific token, while balances tell us how many tokens each address owns. If only the owner mapping changes, the balance data becomes inconsistent with actual ownership. A correct NFT transfer must keep both views of state aligned.

</details>

---

### Question 4: Why does `burn` remove both `Owner(token_id)` and `TokenUri(token_id)`?

<details>
<summary>Click to see answer</summary>

Burning means the token no longer exists. If only ownership is removed but metadata remains, part of the token state is still present in storage. Removing both entries ensures the token is fully deleted from the contract’s state.

</details>

---

### Question 5: What is the difference between `total_supply` and `NextId`?

<details>
<summary>Click to see answer</summary>

`total_supply` tracks how many tokens currently exist. `NextId` tracks the next token ID that will be assigned during minting. These values can differ because tokens may be burned, which lowers supply but does not move `NextId` backward.

</details>

---

## Summary and Next Steps

### What You’ve Learned

In this tutorial, you learned how to build a minimal NFT contract in Soroban.

You now understand:

* how one-time initialization works
* how admin-only minting can be enforced
* how token ownership is stored
* how metadata URIs can be linked to token IDs
* how balances are tracked per address
* how transfer and burn update multiple pieces of state
* how to separate contract config from token data

### Key Takeaways

1. **Initialization matters**
   Contracts with privileged roles should usually protect initialization from being run more than once.

2. **Authentication is not enough by itself**
   Role checks must still compare against stored state.

3. **NFT state is multi-layered**
   You are not only tracking ownership, but also metadata, balances, supply, and next ID.

4. **Consistency across mappings is critical**
   Transfer and burn must keep ownership, balances, and supply in sync.

---

## Homework Assignment

### Part 1: Coding

Implement at least two of these:

* `admin()` getter
* `exists(token_id)`
* empty URI validation
* admin-assisted burn
* lifetime minted counter

Write tests for each new feature.

### Part 2: Reflection

Answer these in 3–4 sentences each:

1. Why does this contract need both `Owner(token_id)` and `Balance(address)`?
2. Why is `NextId` separate from `total_supply`?
3. Why is minting restricted to the admin?
4. What storage patterns in this contract feel similar to earlier tutorials?

### Part 3: Design Challenge

Design a more advanced NFT contract with:

* approval logic
* safe transfer rules
* collection name and symbol
* per-token attributes
* royalty information

For your design, write:

* the expanded `DataKey` enum
* the main function list
* the authorization rules
* the new read functions
* a testing plan

---

## Final Thoughts

This is an excellent next tutorial because it brings together many Soroban fundamentals in one contract.

Earlier tutorials taught you:

* storage basics
* typed keys
* authorization
* state updates
* multi-user logic

This NFT tutorial combines all of that and adds:

* initialization
* admin roles
* token ownership mappings
* metadata storage
* balance accounting
* state deletion through burning

That makes it a strong bridge toward more advanced token standards, NFT marketplaces, and production-style smart contract design.
