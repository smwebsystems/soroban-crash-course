# Mini Escrow Smart Contract Tutorial

> **Course Module:** Soroban Crash Course
> **Follow-up to:** Task Manager Smart Contract Tutorial

---

## Prerequisites

Before starting this tutorial, you should have completed:

1. **Hello World Contract** - Basic contract structure
2. **Increment Contract** - Simple storage operations
3. **Guest Book Contract** - Authentication and structured data with typed storage keys
4. **Task Manager Contract** - State machines, ownership checks, and update operations

### Required Knowledge

* Authentication with `require_auth()`
* Custom data structures using `#[contracttype]`
* Typed storage keys using enums
* Storage operations (`get`, `set`)
* State transitions with enums
* Writing and running tests
* Understanding addresses and ownership

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

* Understand what an escrow contract is
* Explain how state transitions work in a multi-party contract
* Describe role-based authorization in a smart contract
* Understand how one contract can enforce different permissions for different users

**Skills**

* Build a simple escrow contract in Soroban
* Model escrow lifecycle states using enums
* Restrict actions to specific actors like client and freelancer
* Validate state before allowing updates
* Write tests for happy paths and failure paths

**Application**

* Design contracts for freelance work agreements
* Apply role-based state transitions to real-world workflows
* Build contracts where different users have different powers
* Extend escrow logic into more advanced marketplace or payment flows

---

## Introduction

### What Is a Mini Escrow Contract?

A **Mini Escrow** contract is a simple agreement contract between two parties.

In this contract:

* a **client** creates the escrow
* a **freelancer** is assigned to it
* the escrow starts in a `Created` state
* the **client** can release it
* the **freelancer** can refund it

This contract does **not** move tokens yet.
Instead, it simulates the escrow workflow by storing the agreement and updating its state.

That makes it a great learning contract because it helps you focus on:

* contract structure
* state transitions
* multi-user authorization
* storage patterns

### Why Build This?

This tutorial builds naturally on the **Task Manager** contract.

In Task Manager, you learned:

* one resource
* one owner
* one main state transition

In Mini Escrow, you will learn:

* one resource shared by **two roles**
* different permissions for different actors
* branching state transitions

That is much closer to how real smart contracts work.

### Real-World Applications

The patterns in this lesson apply to:

* freelance milestone agreements
* marketplace order flows
* delivery confirmation logic
* approval systems
* refund workflows
* dispute-resolution systems

---

## Conceptual Overview

### Building on Task Manager

Task Manager introduced the idea of a state machine:

* tasks start as `Pending`
* tasks can become `Completed`
* only the owner can update the task

Mini Escrow expands that idea.

Instead of a single owner updating a task, we now have:

* a **client**
* a **freelancer**

Both are part of the same escrow, but they do **not** have the same permissions.

### The New Challenges

This contract answers questions like:

* How do we store an agreement between two users?
* How do we track the escrow lifecycle?
* How do we allow only the client to release?
* How do we allow only the freelancer to refund?
* How do we block updates once the escrow is finalized?

### Our Solution

We will build a contract that:

1. Creates escrows with unique IDs
2. Stores the client, freelancer, amount, and status
3. Uses an enum to track escrow state
4. Uses typed storage keys for safety
5. Enforces role-based authorization
6. Prevents invalid repeat transitions

### Comparison with Task Manager

| Feature             | Task Manager        | Mini Escrow                   |
| ------------------- | ------------------- | ----------------------------- |
| Main resource       | Task                | Escrow                        |
| Roles               | One owner           | Client + Freelancer           |
| State enum          | Pending / Completed | Created / Released / Refunded |
| Authorization       | Owner-based         | Role-based                    |
| Updates             | Complete task       | Release or refund escrow      |
| Storage key pattern | Typed enum          | Typed enum                    |
| Complexity          | Single actor update | Multi-actor update            |

---

## Contract Architecture

### Data Model

This contract uses three main data structures.

---

### 1. Status Enum

```rust
pub enum Status {
    Created,
    Released,
    Refunded,
}
```

This enum represents the escrow’s current state.

* `Created` - the escrow has been created and is still active
* `Released` - the client approved release
* `Refunded` - the freelancer triggered a refund

This gives us a simple but useful state machine.

---

### 2. MiniEscrow Struct

```rust
pub struct MiniEscrow {
    pub id: u32,
    pub client: Address,
    pub freelancer: Address,
    pub amount: u32,
    pub status: Status,
}
```

This struct stores the full escrow record.

**Fields:**

* `id` - unique escrow ID
* `client` - the person who created the escrow
* `freelancer` - the second party
* `amount` - the escrow amount
* `status` - current escrow state

---

### 3. DataKey Enum

```rust
pub enum DataKey {
    Escrow(u32),
    Count,
}
```

This is the typed storage key pattern you already used in Guest Book and Task Manager.

**Variants:**

* `Count` stores the total number of escrows
* `Escrow(u32)` stores one escrow by ID

---

### State Machine Diagram

```text
             create
               │
               ▼
          ┌─────────┐
          │ Created │
          └─────────┘
           │       │
   release │       │ refund
   client  │       │ freelancer
           ▼       ▼
   ┌──────────┐ ┌──────────┐
   │ Released │ │ Refunded │
   └──────────┘ └──────────┘
```

### State Transition Rules

* every new escrow starts as `Created`
* only the **client** can change `Created` to `Released`
* only the **freelancer** can change `Created` to `Refunded`
* once released or refunded, the escrow cannot change again

---

### Contract Functions

| Function                             | Purpose           | Authorization                      | Returns           |
| ------------------------------------ | ----------------- | ---------------------------------- | ----------------- |
| `create(client, freelancer, amount)` | Create new escrow | Client auth                        | Escrow ID (`u32`) |
| `release(client, id)`                | Release escrow    | Client auth + client match         | `bool`            |
| `refund(freelancer, id)`             | Refund escrow     | Freelancer auth + freelancer match | `bool`            |
| `get(id)`                            | Fetch escrow      | Public                             | `MiniEscrow`      |

---

## Step-by-Step Build Section

---

### Step 1: Imports and Setup

```rust
#![no_std]
use soroban_sdk::{contract, contractimpl, contracttype, log, Address, Env};
```

### What this means

* `#![no_std]` tells Rust this contract runs in a constrained environment
* `contract` marks the contract type
* `contractimpl` marks the implementation block
* `contracttype` allows structs and enums to be stored in Soroban storage
* `log` lets us log messages
* `Address` represents account or contract addresses
* `Env` gives access to contract storage and runtime features

This is the standard Soroban contract setup.

---

### Step 2: Define the Status Enum

```rust
#[contracttype]
#[derive(Clone, PartialEq, Debug)]
pub enum Status {
    Created,
    Released,
    Refunded,
}
```

### Why use an enum?

Because the escrow can only be in one valid state at a time.

This is much safer than using strings.

**Bad example:**

```rust
status: "created"
```

Problems:

* typo risk
* no compile-time protection
* inconsistent values possible

**Good example:**

```rust
status: Status::Created
```

The compiler helps enforce valid states.

### Why derive these traits?

* `Clone` lets values be copied when needed
* `PartialEq` lets us compare:

  ```rust
  e.status != Status::Created
  ```
* `Debug` helps with testing and debugging

---

### Step 3: Define the Escrow Struct

```rust
#[contracttype]
#[derive(Clone, PartialEq, Debug)]
pub struct MiniEscrow {
    pub id: u32,
    pub client: Address,
    pub freelancer: Address,
    pub amount: u32,
    pub status: Status,
}
```

### Understanding each field

**`id: u32`**
The escrow ID. It uniquely identifies each escrow.

**`client: Address`**
The person who creates the escrow.

**`freelancer: Address`**
The other participant in the agreement.

**`amount: u32`**
The amount associated with the escrow.

**`status: Status`**
The current state of the escrow.

### Why use `u32` here?

We use `u32` for both:

* `id`
* `amount`

This keeps the tutorial consistent and simple for learning. It is enough for many educational examples and makes the contract easier to follow.

---

### Step 4: Define Typed Storage Keys

```rust
#[contracttype]
#[derive(Clone)]
pub enum DataKey {
    Escrow(u32),
    Count,
}
```

This is the same typed storage key pattern you learned earlier.

### What each variant means

**`Count`**

* stores the number of escrows created so far

**`Escrow(u32)`**

* stores an individual escrow by its ID

### Example

```rust
DataKey::Count
DataKey::Escrow(1)
DataKey::Escrow(2)
```

This approach is better than raw strings because it is:

* type-safe
* consistent
* easier to refactor
* easier to read

---

### Step 5: Create the Contract Shell

```rust
#[contract]
pub struct MiniEscrowContract;

#[contractimpl]
impl MiniEscrowContract {
    // functions go here
}
```

This follows the normal Soroban structure:

* define the contract type
* implement the contract functions inside the impl block

---

### Step 6: Implement `create`

```rust
pub fn create(env: Env, client: Address, freelancer: Address, amount: u32) -> u32 {
    client.require_auth();

    let mut count: u32 = env.storage().instance().get(&DataKey::Count).unwrap_or(0);
    count += 1;

    let e = MiniEscrow {
        id: count,
        client: client.clone(),
        freelancer,
        amount,
        status: Status::Created,
    };

    env.storage().instance().set(&DataKey::Escrow(count), &e);
    env.storage().instance().set(&DataKey::Count, &count);

    log!(&env, "created {}", count);
    count
}
```

Let’s break it down.

---

#### Part A: Authentication

```rust
client.require_auth();
```

The client must authorize the call.

This prevents someone from creating an escrow while pretending to be another client.

---

#### Part B: Load and Increment the Counter

```rust
let mut count: u32 = env.storage().instance().get(&DataKey::Count).unwrap_or(0);
count += 1;
```

This is a common pattern.

* if no count exists yet, use `0`
* increment it to get the next escrow ID

So:

* first escrow gets ID `1`
* second gets `2`
* third gets `3`

---

#### Part C: Build the Escrow Object

```rust
let e = MiniEscrow {
    id: count,
    client: client.clone(),
    freelancer,
    amount,
    status: Status::Created,
};
```

Every new escrow starts in the `Created` state.

This is enforced by the contract.

The caller cannot choose the initial state, which makes the logic more predictable and secure.

---

#### Part D: Save to Storage

```rust
env.storage().instance().set(&DataKey::Escrow(count), &e);
env.storage().instance().set(&DataKey::Count, &count);
```

Two things are stored:

1. the full escrow record
2. the updated total count

Example storage after the first create:

```text
DataKey::Count -> 1
DataKey::Escrow(1) -> MiniEscrow {
    id: 1,
    client: Alice,
    freelancer: Bob,
    amount: 1000,
    status: Created
}
```

---

#### Part E: Log and Return

```rust
log!(&env, "created {}", count);
count
```

* log the creation
* return the new escrow ID

Returning the ID is useful because the caller can immediately use it in later calls.

---

### Step 7: Implement `release`

```rust
pub fn release(env: Env, client: Address, id: u32) -> bool {
    client.require_auth();

    let mut e: MiniEscrow = env
        .storage()
        .instance()
        .get(&DataKey::Escrow(id))
        .expect("not found");

    if e.client != client || e.status != Status::Created {
        return false;
    }

    e.status = Status::Released;
    env.storage().instance().set(&DataKey::Escrow(id), &e);
    true
}
```

This function allows the client to mark the escrow as released.

---

#### Part A: Authenticate

```rust
client.require_auth();
```

The address passed in as `client` must authorize the call.

---

#### Part B: Load the Escrow

```rust
let mut e: MiniEscrow = env
    .storage()
    .instance()
    .get(&DataKey::Escrow(id))
    .expect("not found");
```

This fetches the escrow from storage using the typed key.

If the escrow does not exist, the function panics with `"not found"`.

---

#### Part C: Check Role and State

```rust
if e.client != client || e.status != Status::Created {
    return false;
}
```

This line performs two validations.

**Role check**

```rust
e.client != client
```

Only the stored client is allowed to release.

**State check**

```rust
e.status != Status::Created
```

Release only works while the escrow is still active.

If either check fails, return `false`.

### Why use `||`?

Because either invalid role or invalid state should block the action.

---

#### Part D: Update the State

```rust
e.status = Status::Released;
env.storage().instance().set(&DataKey::Escrow(id), &e);
true
```

If validation passes:

* change status to `Released`
* save the updated escrow
* return `true`

---

### Step 8: Implement `refund`

```rust
pub fn refund(env: Env, freelancer: Address, id: u32) -> bool {
    freelancer.require_auth();

    let mut e: MiniEscrow = env
        .storage()
        .instance()
        .get(&DataKey::Escrow(id))
        .expect("not found");

    if e.freelancer != freelancer || e.status != Status::Created {
        return false;
    }

    e.status = Status::Refunded;
    env.storage().instance().set(&DataKey::Escrow(id), &e);
    true
}
```

This function is similar to `release`, but for the freelancer.

### What is different?

* the authenticated role is the `freelancer`
* the role check uses `e.freelancer`
* the final status becomes `Refunded`

This shows an important contract design pattern:

**different users can be allowed to trigger different transitions on the same resource.**

---

### Step 9: Implement `get`

```rust
pub fn get(env: Env, id: u32) -> MiniEscrow {
    env.storage()
        .instance()
        .get(&DataKey::Escrow(id))
        .expect("not found")
}
```

This is a helper function for reading stored data.

It:

* fetches the escrow by ID
* returns the full `MiniEscrow`
* panics if the escrow does not exist

This is a read-only public accessor.

---

## Complete Contract Code

```rust
#![no_std]
use soroban_sdk::{contract, contractimpl, contracttype, log, Address, Env};

#[contracttype]
#[derive(Clone, PartialEq, Debug)]
pub enum Status {
    Created,
    Released,
    Refunded,
}

#[contracttype]
#[derive(Clone, PartialEq, Debug)]
pub struct MiniEscrow {
    pub id: u32,
    pub client: Address,
    pub freelancer: Address,
    pub amount: u32,
    pub status: Status,
}

#[contracttype]
#[derive(Clone)]
pub enum DataKey {
    Escrow(u32),
    Count,
}

#[contract]
pub struct MiniEscrowContract;

#[contractimpl]
impl MiniEscrowContract {
    pub fn create(env: Env, client: Address, freelancer: Address, amount: u32) -> u32 {
        client.require_auth();

        let mut count: u32 = env.storage().instance().get(&DataKey::Count).unwrap_or(0);
        count += 1;

        let e = MiniEscrow {
            id: count,
            client: client.clone(),
            freelancer,
            amount,
            status: Status::Created,
        };

        env.storage().instance().set(&DataKey::Escrow(count), &e);
        env.storage().instance().set(&DataKey::Count, &count);

        log!(&env, "created {}", count);
        count
    }

    pub fn release(env: Env, client: Address, id: u32) -> bool {
        client.require_auth();

        let mut e: MiniEscrow = env
            .storage()
            .instance()
            .get(&DataKey::Escrow(id))
            .expect("not found");

        if e.client != client || e.status != Status::Created {
            return false;
        }

        e.status = Status::Released;
        env.storage().instance().set(&DataKey::Escrow(id), &e);
        true
    }

    pub fn refund(env: Env, freelancer: Address, id: u32) -> bool {
        freelancer.require_auth();

        let mut e: MiniEscrow = env
            .storage()
            .instance()
            .get(&DataKey::Escrow(id))
            .expect("not found");

        if e.freelancer != freelancer || e.status != Status::Created {
            return false;
        }

        e.status = Status::Refunded;
        env.storage().instance().set(&DataKey::Escrow(id), &e);
        true
    }

    pub fn get(env: Env, id: u32) -> MiniEscrow {
        env.storage()
            .instance()
            .get(&DataKey::Escrow(id))
            .expect("not found")
    }
}
```

---

## Testing Section

Now let’s walk through the tests.

---

### Test Module Setup

```rust
mod test;
#![cfg(test)]
use super::*;

use soroban_sdk::{
    testutils::{Address as _, Logs},
    Address, Env,
};
```

### What this setup does

* `mod test;` means tests are stored in a separate test module
* `#![cfg(test)]` means this code only compiles during testing
* `use super::*;` imports the contract code
* `Address::generate(&env)` creates test addresses
* `env.mock_all_auths()` allows test calls to pass authorization without manual signing
* `Logs` allows inspection of emitted logs

---

### Test 1: `create_stores_data_and_increments_id`

```rust
#[test]
fn create_stores_data_and_increments_id() {
    let env = Env::default();
    env.mock_all_auths();

    let contract_id = env.register(MiniEscrowContract, ());
    let client = MiniEscrowContractClient::new(&env, &contract_id);

    let c1 = Address::generate(&env);
    let f1 = Address::generate(&env);

    let id1 = client.create(&c1, &f1, &1_000u32);
    assert_eq!(id1, 1);

    let e1 = client.get(&id1);
    assert_eq!(e1.id, 1);
    assert_eq!(e1.client, c1);
    assert_eq!(e1.freelancer, f1);
    assert_eq!(e1.amount, 1_000);
    assert_eq!(e1.status, Status::Created);

    let c2 = Address::generate(&env);
    let id2 = client.create(&c2, &f1, &500u32);
    assert_eq!(id2, 2);

    let logs = env.logs().all();
    assert!(logs.iter().any(|l| l.contains("created")));
}
```

### What this test verifies

* the first escrow gets ID `1`
* data is stored correctly
* the second escrow gets ID `2`
* the counter increments properly
* logs contain the word `"created"`

This test checks both storage correctness and ID sequencing.

---

### Test 2: `release_happy_path`

```rust
#[test]
fn release_happy_path() {
    let env = Env::default();
    env.mock_all_auths();

    let contract_id = env.register(MiniEscrowContract, ());
    let client = MiniEscrowContractClient::new(&env, &contract_id);

    let c = Address::generate(&env);
    let f = Address::generate(&env);

    let id = client.create(&c, &f, &1000u32);

    let ok = client.release(&c, &id);
    assert!(ok);

    let e = client.get(&id);
    assert_eq!(e.status, Status::Released);
}
```

### What this tests

* the correct client can call `release`
* the function returns `true`
* the escrow status changes from `Created` to `Released`

---

### Test 3: `release_fails_if_wrong_client`

```rust
#[test]
fn release_fails_if_wrong_client() {
    let env = Env::default();
    env.mock_all_auths();

    let contract_id = env.register(MiniEscrowContract, ());
    let client = MiniEscrowContractClient::new(&env, &contract_id);

    let real_client = Address::generate(&env);
    let wrong_client = Address::generate(&env);
    let freelancer = Address::generate(&env);

    let id = client.create(&real_client, &freelancer, &1000u32);

    let ok = client.release(&wrong_client, &id);
    assert!(!ok);

    let e = client.get(&id);
    assert_eq!(e.status, Status::Created);
}
```

### What this verifies

* the wrong client cannot release the escrow
* the function returns `false`
* the state remains unchanged

This is testing role-based authorization.

---

### Test 4: `release_fails_if_already_released`

```rust
#[test]
fn release_fails_if_already_released() {
    let env = Env::default();
    env.mock_all_auths();

    let contract_id = env.register(MiniEscrowContract, ());
    let client = MiniEscrowContractClient::new(&env, &contract_id);

    let c = Address::generate(&env);
    let f = Address::generate(&env);

    let id = client.create(&c, &f, &1000u32);

    assert!(client.release(&c, &id));
    assert!(!client.release(&c, &id));
}
```

### What this verifies

* the first release succeeds
* the second release fails
* an escrow cannot be released twice

This is state transition validation.

---

### Test 5: `refund_happy_path`

```rust
#[test]
fn refund_happy_path() {
    let env = Env::default();
    env.mock_all_auths();

    let contract_id = env.register(MiniEscrowContract, ());
    let client = MiniEscrowContractClient::new(&env, &contract_id);

    let c = Address::generate(&env);
    let f = Address::generate(&env);

    let id = client.create(&c, &f, &1000u32);

    let ok = client.refund(&f, &id);
    assert!(ok);

    let e = client.get(&id);
    assert_eq!(e.status, Status::Refunded);
}
```

### What this tests

* the correct freelancer can call `refund`
* the function returns `true`
* the status becomes `Refunded`

---

### Test 6: `refund_fails_if_wrong_freelancer`

```rust
#[test]
fn refund_fails_if_wrong_freelancer() {
    let env = Env::default();
    env.mock_all_auths();

    let contract_id = env.register(MiniEscrowContract, ());
    let client = MiniEscrowContractClient::new(&env, &contract_id);

    let c = Address::generate(&env);
    let real_freelancer = Address::generate(&env);
    let wrong_freelancer = Address::generate(&env);

    let id = client.create(&c, &real_freelancer, &1000u32);

    let ok = client.refund(&wrong_freelancer, &id);
    assert!(!ok);

    let e = client.get(&id);
    assert_eq!(e.status, Status::Created);
}
```

### What this verifies

* the wrong freelancer cannot refund
* the function returns `false`
* the escrow stays in `Created`

---

### Test 7: `get_panics_for_missing_id`

```rust
#[test]
#[should_panic(expected = "not found")]
fn get_panics_for_missing_id() {
    let env = Env::default();

    let contract_id = env.register(MiniEscrowContract, ());
    let client = MiniEscrowContractClient::new(&env, &contract_id);

    client.get(&999u32);
}
```

### What this tests

* calling `get` with a missing ID should panic
* the panic message should contain `"not found"`

---

## Test-by-Test Summary

| Test                                   | Purpose                             |
| -------------------------------------- | ----------------------------------- |
| `create_stores_data_and_increments_id` | Verify creation, storage, IDs, logs |
| `release_happy_path`                   | Verify successful release           |
| `release_fails_if_wrong_client`        | Verify only client can release      |
| `release_fails_if_already_released`    | Verify duplicate release is blocked |
| `refund_happy_path`                    | Verify successful refund            |
| `refund_fails_if_wrong_freelancer`     | Verify only freelancer can refund   |
| `get_panics_for_missing_id`            | Verify missing escrow panics        |

---

## Common Pitfalls

### Pitfall 1: Forgetting role checks

**Incorrect**

```rust
pub fn release(env: Env, client: Address, id: u32) -> bool {
    client.require_auth();

    let mut e: MiniEscrow = env
        .storage()
        .instance()
        .get(&DataKey::Escrow(id))
        .expect("not found");

    e.status = Status::Released;
    env.storage().instance().set(&DataKey::Escrow(id), &e);
    true
}
```

### Problem

This checks authentication, but it does not verify that the caller is the stored client.

### Fix

```rust
if e.client != client || e.status != Status::Created {
    return false;
}
```

---

### Pitfall 2: Forgetting state validation

**Incorrect**

```rust
if e.client != client {
    return false;
}

e.status = Status::Released;
```

### Problem

This allows invalid transitions, like releasing an escrow that has already been released or refunded.

### Fix

Always check the current status before changing it.

---

### Pitfall 3: Forgetting to save after update

**Incorrect**

```rust
e.status = Status::Refunded;
true
```

### Problem

You only changed the local variable. The updated state was never written back to storage.

### Fix

```rust
env.storage().instance().set(&DataKey::Escrow(id), &e);
```

---

### Pitfall 4: Using raw storage keys

**Incorrect**

```rust
env.storage().instance().set(&id, &e);
```

### Problem

This breaks the typed-key pattern and makes the code less safe and less consistent.

### Fix

```rust
env.storage().instance().set(&DataKey::Escrow(id), &e);
```

---

### Pitfall 5: Forgetting that this contract is only a workflow simulation

This contract models escrow logic, but it does not yet transfer any real asset.

That is okay for learning.

You are focusing on:

* storage
* state
* permissions
* transitions

Later, you can extend this pattern to include token transfers.

---

## Practice Exercises

### Exercise 1: Add a `Cancelled` Status

Add a new state:

```rust
Cancelled
```

Then implement:

```rust
pub fn cancel(env: Env, client: Address, id: u32) -> bool
```

Rules:

* only the client can cancel
* only allowed from `Created`

---

### Exercise 2: Add `get_count`

The contract stores a count, but it does not expose it.

Create:

```rust
pub fn get_count(env: Env) -> u32
```

It should return the total number of escrows created.

---

### Exercise 3: Prevent zero-amount escrows

Add validation in `create` so that `amount > 0`.

Possible approaches:

* panic when `amount == 0`
* later redesign the function to return a result type

---

### Exercise 4: Add helper getters

Create:

```rust
pub fn get_client(env: Env, id: u32) -> Address
pub fn get_freelancer(env: Env, id: u32) -> Address
```

This helps practice reading a stored struct and returning individual fields.

---

### Exercise 5: Add a finalized checker

Create:

```rust
pub fn is_finalized(env: Env, id: u32) -> bool
```

It should return:

* `true` for `Released` or `Refunded`
* `false` for `Created`

---

## Assessment

### Question 1: Why is `Status` modeled as an enum instead of a string or boolean?

<details>
<summary>Click to see answer</summary>

An enum is better because the escrow has a small set of known valid states. A boolean is too limited because this contract has three states, not two. A string is less safe because typos can create invalid values, while an enum gives compile-time safety.

</details>

---

### Question 2: What is the difference between `client.require_auth()` and `e.client != client`?

<details>
<summary>Click to see answer</summary>

`client.require_auth()` checks authentication. It proves that the address passed in actually authorized the call. `e.client != client` checks authorization against the contract’s stored business rules. It confirms that the authenticated caller is also the client assigned to that escrow.

</details>

---

### Question 3: Why do `release` and `refund` return `bool` instead of always panicking?

<details>
<summary>Click to see answer</summary>

They return `bool` because some failures are expected business outcomes. For example, the wrong actor trying to update the escrow, or trying to update after the escrow is no longer in `Created`, are normal failure cases. Panics are better reserved for unexpected problems like missing data.

</details>

---

### Question 4: How does this tutorial build on the Task Manager contract?

<details>
<summary>Click to see answer</summary>

Task Manager taught state tracking, ownership checks, and update logic. Mini Escrow builds on those same ideas but introduces multiple roles with different permissions. It also adds branching state transitions, where one user can release and another can refund.

</details>

---

### Question 5: Why is `DataKey::Count` still useful even though the contract mainly exposes `get(id)`?

<details>
<summary>Click to see answer</summary>

`DataKey::Count` is what allows the contract to generate unique sequential escrow IDs. Even without a public `get_count()` function, the contract still needs it internally for creation. It also makes the contract easier to extend later with counting or iteration features.

</details>

---

## Summary and Next Steps

### What You’ve Learned

In this tutorial, you learned how to build a small multi-party escrow contract in Soroban.

You now understand:

* how to model workflow states with enums
* how to store escrow records with typed storage keys
* how to assign different permissions to different actors
* how to validate state before transitions
* how to test both success and failure cases

### Key Takeaways

1. **Enums are powerful for workflow states**
   `Created`, `Released`, and `Refunded` clearly model the escrow lifecycle.

2. **Authorization can depend on role**
   The correct signer for one action may not be the correct signer for another.

3. **State validation is essential**
   Even the right actor should not be able to perform invalid transitions.

4. **Typed storage keys remain a best practice**
   The same safe storage-key pattern continues to scale into more advanced contracts.

---

## Homework Assignment

### Part 1: Coding

Implement at least two of these:

* `get_count`
* `cancel`
* zero-amount validation
* `is_finalized`

Write tests for each new feature.

### Part 2: Reflection

Answer these in 3–4 sentences each:

1. What makes Mini Escrow different from Task Manager?
2. Why do `release` and `refund` belong to different roles?
3. Why is `Created` the only state from which transitions are allowed?
4. How do typed storage keys improve this contract?

### Part 3: Design Challenge

Design a more advanced escrow contract with:

* token transfers
* an arbiter
* dispute handling
* expiration logic
* funded deposits

For your design, write:

* the new `Status` enum
* the main data structures
* the function list
* the authorization rules
* the testing plan

---

## Final Thoughts

This is a strong follow-up to Task Manager because it takes the same foundational ideas and applies them in a more realistic multi-user workflow.

Task Manager taught you:

* state
* ownership
* updates

Mini Escrow adds:

* two roles
* role-specific permissions
* branching transitions

That makes it a great bridge toward more advanced smart contracts like real escrow systems, marketplaces, and payment workflows.
