# Guest Book Smart Contract Tutorial

---

## Overview

This tutorial introduces a **Guest Book smart contract** built using Soroban on the Stellar network.

It is designed as the first step after a basic "Hello World" contract, focusing on:
- Authentication
- Structured data storage
- Working with addresses

### What You'll Build

A contract that allows users to:
- Post messages on-chain
- Associate messages with their address
- Retrieve messages by ID
- View total message count

---

## Learning Objectives

After completing this tutorial, you will be able to:

1. Understand how authentication works in Soroban
2. Create custom structs using `#[contracttype]`
3. Use typed storage keys with enums
4. Store and retrieve structured data from storage
5. Use `Address` to represent users
6. Implement and test contract functionality

---

## Contract Design

Each message is stored with an **incrementing numeric ID**, following Soroban best practices by using a typed `DataKey` enum instead of raw string keys.
```
Message 1 -> DataKey::Message(1) -> { author, text }
Message 2 -> DataKey::Message(2) -> { author, text }
...
Total Count -> DataKey::Count -> u32
```

---

## Full Contract Code
```rust
#![no_std]
use soroban_sdk::{contract, contractimpl, contracttype, log, Address, Env, String};

/// Structured storage keys for the contract.
/// This is better than using raw strings like "COUNT"
/// because it is type-safe and avoids typo-related bugs.
#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub enum DataKey {
    Count,
    Message(u32),
}

/// A guest book message
#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct Message {
    pub author: Address,
    pub text: String,
}

#[contract]
pub struct GuestBook;

#[contractimpl]
impl GuestBook {
    /// Post a message to the guest book.
    /// The author must authorize this action.
    pub fn post_message(env: Env, author: Address, text: String) {
        author.require_auth();

        // Read current count from storage, defaulting to 0 if not set
        let mut count: u32 = env.storage().instance().get(&DataKey::Count).unwrap_or(0);

        // Increment count to create the next message ID
        count += 1;

        let message = Message {
            author: author.clone(),
            text,
        };

        // Store the message under a typed key
        env.storage()
            .instance()
            .set(&DataKey::Message(count), &message);

        // Update the message count
        env.storage().instance().set(&DataKey::Count, &count);

        log!(&env, "Message {} posted by: {}", count, author);
    }

    /// Get a specific message by its ID
    pub fn get_message(env: Env, message_id: u32) -> Message {
        env.storage()
            .instance()
            .get(&DataKey::Message(message_id))
            .unwrap_or_else(|| panic!("Message not found"))
    }

    /// Get the total number of messages
    pub fn get_message_count(env: Env) -> u32 {
        env.storage().instance().get(&DataKey::Count).unwrap_or(0)
    }
}

#[cfg(test)]
mod test {
    use super::*;
    use soroban_sdk::{testutils::Address as _, Address, Env};

    #[test]
    fn test_post_and_get_message() {
        let env = Env::default();
        env.mock_all_auths();

        let contract_id = env.register(GuestBook, ());
        let client = GuestBookClient::new(&env, &contract_id);

        let user = Address::generate(&env);
        let message_text = String::from_str(&env, "Hello, Soroban!");

        client.post_message(&user, &message_text);

        let msg = client.get_message(&1);
        assert_eq!(msg.author, user);
        assert_eq!(msg.text, message_text);

        let count = client.get_message_count();
        assert_eq!(count, 1);
    }

    #[test]
    fn test_multiple_messages() {
        let env = Env::default();
        env.mock_all_auths();

        let contract_id = env.register(GuestBook, ());
        let client = GuestBookClient::new(&env, &contract_id);

        let user1 = Address::generate(&env);
        let user2 = Address::generate(&env);

        let first = String::from_str(&env, "First!");
        let second = String::from_str(&env, "Second!");

        client.post_message(&user1, &first);
        client.post_message(&user2, &second);

        assert_eq!(client.get_message_count(), 2);

        let msg1 = client.get_message(&1);
        assert_eq!(msg1.author, user1);
        assert_eq!(msg1.text, first);

        let msg2 = client.get_message(&2);
        assert_eq!(msg2.author, user2);
        assert_eq!(msg2.text, second);
    }

    #[test]
    #[should_panic(expected = "Message not found")]
    fn test_get_nonexistent_message() {
        let env = Env::default();

        let contract_id = env.register(GuestBook, ());
        let client = GuestBookClient::new(&env, &contract_id);

        client.get_message(&999);
    }
}
```

---

## Core Concepts

### 1. `#![no_std]` - No Standard Library

Soroban contracts are compiled to **WebAssembly (WASM)** and do not use Rust's standard library.

> **Note:** This is required for on-chain execution, where contracts run in a restricted environment.

---

### 2. Typed Storage Keys with `DataKey`
```rust
#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub enum DataKey {
    Count,
    Message(u32),
}
```

#### Why This Is Better

| Raw Strings | Typed Enum |
|-------------|------------|
| `"COUNT"` prone to typos | Type-safe |
| No compile-time checks | Catches errors early |
| Hard to understand | Self-documenting |

**Storage Structure:**
- `DataKey::Count` -> Total number of messages
- `DataKey::Message(1)` -> First message
- `DataKey::Message(2)` -> Second message

---

### 3. Custom Data Structures with `#[contracttype]`
```rust
#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct Message {
    pub author: Address,
    pub text: String,
}
```

The `Message` struct represents structured data stored on-chain.

**Derived Traits:**
- `Clone` -> Allows copying values when needed
- `Debug` -> Helps with inspection and debugging
- `Eq` + `PartialEq` -> Allow equality checks in tests

---

### 4. Authentication with `require_auth()`
```rust
author.require_auth();
```

#### Security Benefits

- Ensures the author actually approved the transaction
- Prevents impersonation attacks
- No one can post messages pretending to be another address

> **Important:** This is one of the most important security checks in smart contract development.

---

### 5. Instance Storage
```rust
env.storage().instance()
```

Soroban provides contract storage for:
- Total message count
- Individual messages under typed keys

Instance storage is ideal for contract-specific state like counters and records.

---

### 6. Message Indexing Flow
```mermaid
graph LR
    A[Read Count] --> B[Increment]
    B --> C[Create Message]
    C --> D[Store Message]
    D --> E[Update Count]
```

**Step-by-step:**
1. Read the current message count
2. Increment it to create the next message ID
3. Build a `Message` struct
4. Store the message under `DataKey::Message(count)`
5. Store the new count under `DataKey::Count`

---

### 7. Retrieving Data
```rust
pub fn get_message(env: Env, message_id: u32) -> Message
```

Retrieves a stored message by ID. If the message doesn't exist:
```
panic!("Message not found")
```

This makes missing data explicit and easy to detect during testing.

---

### 8. Logging
```rust
log!(&env, "Message {} posted by: {}", count, author);
```

**Use cases:**
- Debugging contract behavior
- Tracing execution
- Observing values (message ID, author, etc.)

---

## Testing

Soroban provides a **local testing environment** to verify contract behavior before deployment.

### Key Testing Setup
```rust
let env = Env::default();
env.mock_all_auths();
```

- `Env::default()` -> Creates a simulated blockchain environment
- `mock_all_auths()` -> Simulates authorization (bypasses `require_auth()` checks)

### Generating Test Addresses
```rust
let user = Address::generate(&env);
```

Creates a test address that acts like a user account.

---

### Test Scenarios

#### Test 1: Posting and Retrieving a Message

Verifies:
- User can post a message
- Message is stored correctly
- Message can be retrieved by ID
- Total count becomes 1

#### Test 2: Multiple Messages

Verifies:
- Multiple users can post messages
- Message count increases correctly
- Each message is stored under the correct ID
- Retrieved messages match author and text

#### Test 3: Non-Existent Message

Verifies:
- Requesting a missing message causes a panic
- Error message is correct: `"Message not found"`

---

## Best Practices

### Why This Version Is Better

| Aspect | Basic Version | This Version |
|--------|---------------|--------------|
| Storage Keys | Raw strings `"COUNT"` | Typed enum `DataKey::Count` |
| Type Safety | Runtime errors | Compile-time checks |
| Maintainability | Prone to typos | Self-documenting |
| Scalability | Harder to extend | Easy to add new keys |

### How This Prepares You for Advanced Contracts

This Guest Book introduces:
- Authentication with `require_auth`
- Structured storage with `#[contracttype]`
- Typed storage keys using enums
- Persistent counters
- Message retrieval patterns
- Local contract testing

---

## Assignment

Extend this contract by adding:

### Task 1: Timestamps
Add a `timestamp` field to each message

### Task 2: Delete Messages
Add a function to delete a message (only by its author)

### Task 3: Validation
Prevent empty messages from being posted

**Hint:** These extensions will deepen your understanding of authorization, validation, and state control.

---

## Next Steps

### Task Manager Contract

After completing this tutorial, proceed to learn:
- Enums for task state
- Ownership checks
- Controlled state transitions
- Controlled updates

### Escrow Contract (Advanced)

Eventually build an Escrow contract that combines:
- Multiple roles
- Token transfers
- Stricter state management
- Advanced authorization rules

---

## Resources

- [Soroban Documentation](https://soroban.stellar.org/)
- [Stellar Developer Discord](https://discord.gg/stellar)
- [Rust Programming Language](https://www.rust-lang.org/)



---

**Happy Building!**
