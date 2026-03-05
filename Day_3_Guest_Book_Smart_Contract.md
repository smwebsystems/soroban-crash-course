# Guest Book Smart Contract

## Overview

This tutorial introduces a Guest Book smart contract built using Soroban on the Stellar network.

It is designed as the first step after a basic "Hello World" contract. The focus is on understanding authentication, structured data storage, and working with addresses.

By completing this tutorial, participants will learn how to:

- Require transaction authorization using `require_auth()`
- Define and store custom data structures
- Work with `Address` types
- Persist structured data in contract storage
- Write and run Soroban tests

This contract builds the foundation required for more advanced contracts such as Task Managers and Escrow systems.

---

## Learning Objectives

After this tutorial, students should be able to:

1. Understand how authentication works in Soroban
2. Create custom structs using `#[contracttype]`
3. Store and retrieve structured data from storage
4. Use `Address` to represent users
5. Implement and test contract functionality

---

## Contract Design

The Guest Book contract allows users to:

- Post a message
- Associate the message with their address
- Store messages on-chain
- Retrieve messages by ID
- View total message count

Each message is stored with an incrementing numeric ID.

---

## Full Working Contract Code (Includes Tests)

```rust
#![no_std]
use soroban_sdk::{contract, contractimpl, contracttype, Address, Env, String, log};

/// A guest book contract to learn:
/// - Authentication (require_auth)
/// - Custom data structures
/// - Working with addresses
/// - Storing structured data

#[contracttype]
#[derive(Clone, Debug)]
pub struct Message {
    pub author: Address,
    pub text: String,
}

#[contract]
pub struct GuestBook;

#[contractimpl]
impl GuestBook {
    /// Post a message to the guest book
    /// The author must authorize this action
    pub fn post_message(env: Env, author: Address, text: String) {
        // Require authentication from the author
        author.require_auth();

        let message = Message {
            author: author.clone(),
            text: text.clone(),
        };

        // Get current message count
        let mut count: u32 = env.storage().instance().get(&"COUNT").unwrap_or(0);
        count += 1;

        // Store the message with its index
        env.storage().instance().set(&count, &message);
        env.storage().instance().set(&"COUNT", &count);

        log!(&env, "Message posted by: {}", author);
    }

    /// Get a specific message by its number
    pub fn get_message(env: Env, message_id: u32) -> Message {
        env.storage()
            .instance()
            .get(&message_id)
            .expect("Message not found")
    }

    /// Get the total number of messages
    pub fn get_message_count(env: Env) -> u32 {
        env.storage().instance().get(&"COUNT").unwrap_or(0)
    }
}

#[cfg(test)]
mod test {
    use super::*;
    use soroban_sdk::{testutils::Address as _, Address, Env};

    #[test]
    fn test_post_and_get_message() {
        let env = Env::default();
        env.mock_all_auths(); // Mock auth for testing

        let contract_id = env.register(GuestBook, ());
        let client = GuestBookClient::new(&env, &contract_id);

        let user = Address::generate(&env);
        let message_text = String::from_str(&env, "Hello, Soroban!");

        // Post a message
        client.post_message(&user, &message_text);

        // Verify the message
        let msg = client.get_message(&1);
        assert_eq!(msg.author, user);
        assert_eq!(msg.text, message_text);

        // Check count
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

        client.post_message(&user1, &String::from_str(&env, "First!"));
        client.post_message(&user2, &String::from_str(&env, "Second!"));

        assert_eq!(client.get_message_count(), 2);

        let msg1 = client.get_message(&1);
        assert_eq!(msg1.author, user1);

        let msg2 = client.get_message(&2);
        assert_eq!(msg2.author, user2);
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

## Core Concepts Explained

### 1. `#![no_std]`

- Soroban contracts are compiled to WebAssembly (WASM) and do not use Rust's standard library.
- This is required for on-chain execution.

### 2. Custom Data Structures (`#[contracttype]`)

```rust
#[contracttype]
#[derive(Clone, Debug)]
pub struct Message {
    pub author: Address,
    pub text: String,
}
```

The `Message` struct represents structured data stored on-chain.

The `#[contracttype]` attribute makes the struct serializable so it can be stored in contract storage.

Each message contains:
- The author's address
- The message text

### 3. Authentication with `require_auth()`

```rust
author.require_auth();
```

This ensures that:
- The caller signed the transaction
- No one can post a message pretending to be someone else

Without authentication, any address could be impersonated.

**Authentication is one of the most important security concepts in smart contract development.**

### 4. Persistent Storage + Message Indexing

Soroban provides contract storage using:

```rust
env.storage().instance()
```

We use storage to:
- Keep track of message count (`"COUNT"`)
- Store each message using its numeric ID (1, 2, 3, ...)

**How posting works:**

1. Read `"COUNT"` from storage (defaults to 0)
2. Add 1 to get the next message ID
3. Store the message under that ID
4. Update `"COUNT"`

This gives every message a predictable ID.

### 5. Retrieving Data

```rust
pub fn get_message(env: Env, message_id: u32) -> Message
```

This function retrieves a stored message by ID.

If the message does not exist, the contract panics with:
```
Message not found
```

This ensures invalid reads are handled explicitly.

---

## Testing the Contract

Soroban provides a powerful local testing environment.

### Key Testing Setup

```rust
let env = Env::default();
env.mock_all_auths();
```

- `Env::default()` creates a simulated blockchain environment (local test instance).
- `mock_all_auths()` simulates required signatures so `require_auth()` won't fail in tests.

### Generating Users

```rust
let user = Address::generate(&env);
```

This creates a new test address (like a user account) inside the test environment.

### Test Scenarios Included

This working version includes three test scenarios:

#### 1. Posting and retrieving a message
- Post one message
- Retrieve it by ID 1
- Verify author + text
- Verify message count is 1

#### 2. Posting multiple messages
- Post from user1 and user2
- Check message count is 2
- Verify message 1 belongs to user1
- Verify message 2 belongs to user2

#### 3. Retrieving a non-existent message
- Calls `get_message(999)`
- Test expects a panic with "Message not found"

---

## How This Prepares Students for More Advanced Contracts

This Guest Book contract introduces:

- Authentication (`require_auth`)
- Structured storage (`#[contracttype]` + structs)
- Persistent counters (`"COUNT"`)
- Data retrieval patterns
- Proper local testing workflow (including mocked auth)

The next tutorial (Task Manager) builds on this by introducing:

- Enums for state management
- Ownership checks
- Controlled state transitions

From there, students can move into building an Escrow contract, which combines:

- Multiple roles
- Token transfers
- Strict state transition rules

---

## Assignment

Extend this contract by:

1. Adding a timestamp field
2. Adding a function to delete a message (only by its author)
3. Preventing empty messages

This will deepen understanding of authorization and state control.

---

## Next Step

After completing this tutorial, proceed to the **Task Manager** contract to learn:

- Enums
- State transitions
- Ownership validation
- Controlled updates
