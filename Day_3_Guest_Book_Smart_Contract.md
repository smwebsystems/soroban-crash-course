# Guest Book Smart Contract Tutorial

## Prerequisites

Before starting this tutorial, you should have completed:

1. **Hello World Contract** - Understanding basic contract structure and deployment
2. **Increment Contract** - Working with storage and state management

### Required Knowledge

- Basic Rust syntax (variables, functions, structs, enums)
- Understanding of blockchain accounts and addresses
- Familiarity with Soroban contract structure (`#[contract]`, `#[contractimpl]`)
- Basic storage operations (`get`, `set`)

### Development Environment

Ensure you have installed:
- Rust toolchain (1.70 or later)
- Soroban CLI
- Code editor with Rust support (VS Code recommended)

---

## Learning Objectives

By the end of this lesson, you will be able to:

**Knowledge**
- Explain the purpose and importance of authentication in smart contracts
- Describe how typed storage keys improve code quality and maintainability
- Understand the relationship between addresses and contract authorization

**Skills**
- Implement authentication using `require_auth()` to secure contract functions
- Create and use custom data structures with `#[contracttype]`
- Design and implement typed storage key systems using enums
- Store and retrieve structured data from contract storage
- Write comprehensive unit tests for contract functionality

**Application**
- Build a complete guest book application with multiple user interactions
- Apply authentication patterns to prevent unauthorized actions
- Design scalable storage schemas for real-world contracts

---

## Introduction

### What Is a Guest Book Contract?

A **Guest Book** is a simple interactive application where users can:
- Sign in by posting a message
- View messages from other users
- See how many people have visited

Think of it like a digital guestbook at a museum or event, but stored permanently on the blockchain!

### Why Build This?

This contract introduces three critical concepts you'll use in every smart contract:

1. **Authentication** - Ensuring users are who they claim to be
2. **Structured Data** - Organizing complex information efficiently
3. **Storage Patterns** - Best practices for saving data on-chain

### Real-World Applications

The patterns you learn here apply to:
- Social media posts
- Comment systems
- Review platforms
- User registration systems
- Activity logging

---

## Conceptual Overview

### The Problem We're Solving

In the **Increment Contract**, you learned how to store a simple number. But what if we need to store more complex information?

**Questions to consider:**
- How do we store who posted a message?
- How do we prevent someone from posting as another user?
- How do we organize multiple messages efficiently?

### Our Solution

We'll build a system that:
1. Uses **typed storage keys** instead of strings (avoiding typos and bugs)
2. Implements **authentication** to verify user identity
3. Stores **structured data** (combining author and message text)
4. Maintains a **counter** to track message IDs

### Comparison with Previous Lessons

| Feature | Hello World | Increment | Guest Book |
|---------|-------------|-----------|------------|
| Storage | None | Single number | Multiple structured records |
| Authentication | No | No | **Yes** |
| Data Structure | Text | Primitive (u32) | **Custom struct** |
| Storage Keys | N/A | String/Symbol | **Typed enum** |
| User Interaction | View only | Single user | **Multi-user** |

---

## Contract Architecture

### Data Model

Our contract uses two main data structures:

#### 1. Storage Keys (DataKey Enum)
```rust
pub enum DataKey {
    Count,           // Stores total message count
    Message(u32),    // Stores individual messages by ID
}
```

**Think of it like a filing system:**
- `Count` is the drawer label for "Total Messages"
- `Message(1)`, `Message(2)`, etc. are labeled folders for each message

#### 2. Message Structure
```rust
pub struct Message {
    pub author: Address,  // Who wrote it
    pub text: String,     // What they wrote
}
```

### Storage Layout Diagram
```
Contract Storage
├── DataKey::Count → 3
├── DataKey::Message(1) → { author: 0xABC..., text: "Hello!" }
├── DataKey::Message(2) → { author: 0xDEF..., text: "Welcome!" }
└── DataKey::Message(3) → { author: 0x123..., text: "Great project!" }
```

### Contract Functions

| Function | Purpose | Authentication |
|----------|---------|----------------|
| `post_message(author, text)` | Add a new message | Required |
| `get_message(message_id)` | Retrieve a specific message | Not required |
| `get_message_count()` | Get total number of messages | Not required |

---

## Step-by-Step Implementation

### Step 1: Set Up the Contract Structure
```rust
#![no_std]
use soroban_sdk::{contract, contractimpl, contracttype, log, Address, Env, String};
```

**What's happening here?**
- `#![no_std]` - We don't use Rust's standard library (required for blockchain)
- We import only what we need from `soroban_sdk`

**Discussion Question:** Why can't we use the standard library in blockchain contracts?

<details>
<summary>Click to see answer</summary>

Blockchain contracts run in a restricted WebAssembly environment without access to system resources like file I/O, networking, or operating system features. The standard library depends on these features, so we use `no_std` mode.
</details>

---

### Step 2: Define Storage Keys
```rust
#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub enum DataKey {
    Count,
    Message(u32),
}
```

**Key Concepts:**

- `#[contracttype]` - Tells Soroban this type can be stored on-chain
- `enum` - Allows us to create different kinds of keys
- `Clone, Debug, Eq, PartialEq` - Traits needed for testing and comparison

**Why use an enum instead of strings?**

| Approach | Example | Problem |
|----------|---------|---------|
| String keys | `"COUNT"`, `"MESAGE_1"` | Typos cause runtime errors |
| Typed enum | `DataKey::Count`, `DataKey::Message(1)` | Typos cause compile errors |

**Interactive Exercise:** What happens if you type `DataKey::Cont` instead of `DataKey::Count`?

<details>
<summary>Click to see answer</summary>

The Rust compiler will immediately show an error: "no variant named `Cont` found for enum `DataKey`". This catches the mistake before you even run the code!
</details>

---

### Step 3: Define the Message Structure
```rust
#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct Message {
    pub author: Address,
    pub text: String,
}
```

**Understanding the Components:**

- `pub struct Message` - Creates a custom data type
- `author: Address` - The blockchain address of who posted the message
- `text: String` - The message content

**Thought Exercise:** Why do we need to store the author's address? What could go wrong if we only stored the text?

---

### Step 4: Create the Contract and Implementation
```rust
#[contract]
pub struct GuestBook;

#[contractimpl]
impl GuestBook {
    // Functions go here
}
```

**Contract Skeleton Explained:**
- `#[contract]` - Marks this as a Soroban smart contract
- `pub struct GuestBook;` - Empty struct (we store data in contract storage, not here)
- `#[contractimpl]` - Contains all the contract's functions

---

### Step 5: Implement `post_message`

Let's build this function piece by piece:

#### 5a. Function Signature and Authentication
```rust
pub fn post_message(env: Env, author: Address, text: String) {
    author.require_auth();
```

**Breaking it down:**
- `env: Env` - Provides access to contract storage and blockchain state
- `author: Address` - The user posting the message
- `text: String` - The message content
- `author.require_auth()` - **Critical security check!**

**Security Spotlight:**
```rust
author.require_auth();
```

This single line prevents:
- User A from posting as User B
- Unauthorized messages
- Impersonation attacks

**Real-world analogy:** Like requiring a signature on a contract - you can't sign someone else's name!

#### 5b. Read and Update Counter
```rust
let mut count: u32 = env.storage().instance().get(&DataKey::Count).unwrap_or(0);
count += 1;
```

**What's happening:**
1. Try to read the current count from storage
2. If nothing is stored yet, use `0` as the default (`unwrap_or(0)`)
3. Increment the count to create the next message ID

**Interactive Question:** What will `count` be for the very first message?

<details>
<summary>Click to see answer</summary>

For the first message, storage is empty, so `get(&DataKey::Count)` returns `None`. The `unwrap_or(0)` provides `0` as the default, then we increment to `1`. So the first message has ID `1`.
</details>

#### 5c. Create and Store the Message
```rust
let message = Message {
    author: author.clone(),
    text,
};

env.storage()
    .instance()
    .set(&DataKey::Message(count), &message);
```

**Step-by-step:**
1. Create a `Message` struct with the author and text
2. Store it using a typed key `DataKey::Message(count)`

**Why `.clone()`?** We need a copy of `author` for logging later, but we're moving it into the `Message` struct.

#### 5d. Update Counter and Log
```rust
env.storage().instance().set(&DataKey::Count, &count);

log!(&env, "Message {} posted by: {}", count, author);
```

**Final steps:**
1. Save the new count back to storage
2. Log the event for debugging and transparency

---

### Step 6: Implement `get_message`
```rust
pub fn get_message(env: Env, message_id: u32) -> Message {
    env.storage()
        .instance()
        .get(&DataKey::Message(message_id))
        .unwrap_or_else(|| panic!("Message not found"))
}
```

**Understanding the Flow:**
1. Look up the message by its ID
2. If found, return it
3. If not found, panic with an error message

**Discussion:** Why panic instead of returning an Option or Result?

<details>
<summary>Click to see answer</summary>

In Soroban contracts, panicking is the standard way to handle errors. When a contract panics, the entire transaction is reverted, ensuring no partial state changes occur. This is safer than returning optional values that might be ignored.
</details>

---

### Step 7: Implement `get_message_count`
```rust
pub fn get_message_count(env: Env) -> u32 {
    env.storage().instance().get(&DataKey::Count).unwrap_or(0)
}
```

**Simple and straightforward:**
- Read the count from storage
- Return `0` if no messages exist yet

---

## Complete Contract Code

Now let's see how all the pieces fit together:
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
        // SECURITY: Verify the author authorized this transaction
        author.require_auth();

        // Read current count from storage, defaulting to 0 if not set
        let mut count: u32 = env.storage().instance().get(&DataKey::Count).unwrap_or(0);

        // Increment count to create the next message ID
        count += 1;

        // Create the message with author and text
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

        // Log the event for transparency
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
```

---

## Testing Your Contract

Testing is **critical** in smart contract development. Once deployed, contracts cannot be easily changed!

### Understanding the Test Environment

Soroban provides a local testing framework that simulates the blockchain:
```rust
#[cfg(test)]
mod test {
    use super::*;
    use soroban_sdk::{testutils::Address as _, Address, Env};
```

**Test Setup Components:**
- `#[cfg(test)]` - Code only compiles when testing
- `use super::*` - Import everything from the parent module
- Test utilities for creating addresses

---

### Test 1: Basic Message Posting and Retrieval
```rust
#[test]
fn test_post_and_get_message() {
    // 1. Create a test environment
    let env = Env::default();
    env.mock_all_auths();

    // 2. Register the contract
    let contract_id = env.register(GuestBook, ());
    let client = GuestBookClient::new(&env, &contract_id);

    // 3. Create test data
    let user = Address::generate(&env);
    let message_text = String::from_str(&env, "Hello, Soroban!");

    // 4. Post a message
    client.post_message(&user, &message_text);

    // 5. Verify the message was stored correctly
    let msg = client.get_message(&1);
    assert_eq!(msg.author, user);
    assert_eq!(msg.text, message_text);

    // 6. Verify the count updated
    let count = client.get_message_count();
    assert_eq!(count, 1);
}
```

**Step-by-Step Explanation:**

**Step 1: Create Environment**
```rust
let env = Env::default();
env.mock_all_auths();
```
- Creates a simulated blockchain
- `mock_all_auths()` bypasses authentication (allows testing without real signatures)

**Step 2: Register Contract**
```rust
let contract_id = env.register(GuestBook, ());
let client = GuestBookClient::new(&env, &contract_id);
```
- Deploys the contract to the test environment
- Creates a client to interact with it

**Step 3: Create Test Data**
```rust
let user = Address::generate(&env);
let message_text = String::from_str(&env, "Hello, Soroban!");
```
- Generates a random test address
- Creates a test message string

**Step 4-6: Test the Functionality**
- Call the contract function
- Verify the results match expectations

---

### Test 2: Multiple Users Posting Messages
```rust
#[test]
fn test_multiple_messages() {
    let env = Env::default();
    env.mock_all_auths();

    let contract_id = env.register(GuestBook, ());
    let client = GuestBookClient::new(&env, &contract_id);

    // Create two different users
    let user1 = Address::generate(&env);
    let user2 = Address::generate(&env);

    let first = String::from_str(&env, "First!");
    let second = String::from_str(&env, "Second!");

    // Both users post messages
    client.post_message(&user1, &first);
    client.post_message(&user2, &second);

    // Verify count is correct
    assert_eq!(client.get_message_count(), 2);

    // Verify each message is stored correctly
    let msg1 = client.get_message(&1);
    assert_eq!(msg1.author, user1);
    assert_eq!(msg1.text, first);

    let msg2 = client.get_message(&2);
    assert_eq!(msg2.author, user2);
    assert_eq!(msg2.text, second);
}
```

**What This Tests:**
- Multiple users can post messages
- Each message gets a unique ID
- Messages are stored independently
- Counter increments correctly

---

### Test 3: Error Handling
```rust
#[test]
#[should_panic(expected = "Message not found")]
fn test_get_nonexistent_message() {
    let env = Env::default();

    let contract_id = env.register(GuestBook, ());
    let client = GuestBookClient::new(&env, &contract_id);

    // Try to get a message that doesn't exist
    client.get_message(&999);
}
```

**Understanding `#[should_panic]`:**
- This test expects the code to panic
- If it doesn't panic, the test fails
- Verifies our error handling works correctly

**Why Test Error Cases?**
- Ensures the contract fails safely
- Prevents unexpected behavior
- Validates error messages are clear

---

### Running Your Tests
```bash
# Run all tests
cargo test

# Run tests with output
cargo test -- --nocapture

# Run a specific test
cargo test test_post_and_get_message
```

**Expected Output:**
```
running 3 tests
test test::test_post_and_get_message ... ok
test test::test_multiple_messages ... ok
test test::test_get_nonexistent_message ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

---

## Common Pitfalls

### Pitfall 1: Forgetting `require_auth()`

**Incorrect:**
```rust
pub fn post_message(env: Env, author: Address, text: String) {
    // Missing: author.require_auth();
    
    let message = Message { author, text };
    // ... rest of code
}
```

**Problem:** Anyone can post as anyone else!

**Correct:**
```rust
pub fn post_message(env: Env, author: Address, text: String) {
    author.require_auth(); // Always check authentication first!
    
    let message = Message { author, text };
    // ... rest of code
}
```

---

### Pitfall 2: Using String Keys Instead of Typed Enums

**Incorrect:**
```rust
env.storage().instance().get("COUNT") // Typo-prone!
env.storage().instance().get("MESAGE_1") // Oops, wrong spelling
```

**Correct:**
```rust
env.storage().instance().get(&DataKey::Count) // Type-safe
env.storage().instance().get(&DataKey::Message(1)) // Compiler-checked
```

---

### Pitfall 3: Not Handling Missing Data

**Incorrect:**
```rust
pub fn get_message_count(env: Env) -> u32 {
    env.storage().instance().get(&DataKey::Count).unwrap()
    // Panics if count doesn't exist!
}
```

**Correct:**
```rust
pub fn get_message_count(env: Env) -> u32 {
    env.storage().instance().get(&DataKey::Count).unwrap_or(0)
    // Returns 0 if no messages yet
}
```

---

### Pitfall 4: Forgetting to Update the Counter

**Incorrect:**
```rust
let count: u32 = env.storage().instance().get(&DataKey::Count).unwrap_or(0);
count += 1;
env.storage().instance().set(&DataKey::Message(count), &message);
// Missing: env.storage().instance().set(&DataKey::Count, &count);
```

**Problem:** Counter never increases, all messages overwrite ID 1!

---

## Practice Exercises

### Exercise 1: Add Message Timestamps (Beginner)

**Goal:** Add a timestamp field to each message

**Requirements:**
1. Modify the `Message` struct to include a `timestamp: u64` field
2. Update `post_message` to record the current ledger timestamp
3. Modify tests to verify timestamps are stored

**Hints:**
- Use `env.ledger().timestamp()` to get current time
- Remember to update all struct instantiations
- Test that each message has a different timestamp

**Solution Template:**
```rust
#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct Message {
    pub author: Address,
    pub text: String,
    pub timestamp: u64, // Add this field
}

pub fn post_message(env: Env, author: Address, text: String) {
    author.require_auth();
    
    // ... existing code ...
    
    let message = Message {
        author: author.clone(),
        text,
        timestamp: env.ledger().timestamp(), // Add this
    };
    
    // ... rest of code ...
}
```

---

### Exercise 2: Prevent Empty Messages (Intermediate)

**Goal:** Add validation to reject empty messages

**Requirements:**
1. Check if the message text is empty
2. Panic with a clear error message if it is
3. Write a test that verifies empty messages are rejected

**Hints:**
- Use `text.len()` to check length
- Add the check right after `require_auth()`
- Use `#[should_panic]` in your test

**Test Template:**
```rust
#[test]
#[should_panic(expected = "Message cannot be empty")]
fn test_empty_message_rejected() {
    let env = Env::default();
    env.mock_all_auths();
    
    let contract_id = env.register(GuestBook, ());
    let client = GuestBookClient::new(&env, &contract_id);
    
    let user = Address::generate(&env);
    let empty = String::from_str(&env, "");
    
    client.post_message(&user, &empty);
}
```

---

### Exercise 3: Delete Messages (Advanced)

**Goal:** Allow users to delete their own messages

**Requirements:**
1. Create a `delete_message` function
2. Only allow the original author to delete their message
3. Handle attempts to delete non-existent messages
4. Write comprehensive tests

**Challenges to Consider:**
- How do you verify the caller is the original author?
- What happens to the message count?
- Should IDs be reused?

**Function Signature:**
```rust
pub fn delete_message(env: Env, caller: Address, message_id: u32) {
    // Your implementation here
}
```

---

### Exercise 4: Get Recent Messages (Advanced)

**Goal:** Create a function that returns the last N messages

**Requirements:**
1. Accept a parameter for how many messages to return
2. Return messages in reverse chronological order (newest first)
3. Handle cases where fewer messages exist than requested

**Function Signature:**
```rust
pub fn get_recent_messages(env: Env, count: u32) -> Vec<Message> {
    // Your implementation here
}
```

**Hints:**
- You'll need to import `Vec` from `soroban_sdk`
- Loop backwards from the current count
- Handle the case where `count` > total messages

---

## Assessment

### Knowledge Check Questions

**Question 1:** What does `require_auth()` do, and why is it important?

<details>
<summary>Click to see answer</summary>

`require_auth()` verifies that the transaction was authorized by the specified address. It's critical for security because it prevents impersonation - without it, anyone could perform actions on behalf of other users.
</details>

---

**Question 2:** Why do we use typed enums for storage keys instead of strings?

<details>
<summary>Click to see answer</summary>

Typed enums provide:
- Type safety (compiler catches mistakes)
- No typo-related bugs (strings can be mistyped)
- Better IDE support (autocomplete)
- Self-documenting code (clear structure)
- Compile-time verification instead of runtime errors
</details>

---

**Question 3:** What happens if you try to `get_message` with an ID that doesn't exist?

<details>
<summary>Click to see answer</summary>

The contract panics with the error "Message not found" and the entire transaction is reverted. This is the correct behavior in Soroban - panicking ensures no partial state changes occur.
</details>

---

### Code Review Challenge

**Find the bugs in this code:**
```rust
pub fn post_message(env: Env, author: Address, text: String) {
    let count: u32 = env.storage().instance().get("COUNT").unwrap();
    count += 1;
    
    let message = Message { author, text };
    
    env.storage().instance().set("MESSAGE", &message);
}
```

<details>
<summary>Click to see answers</summary>

**Bugs found:**

1. **Missing authentication** - No `author.require_auth()` call
2. **Using string keys** - Should use `DataKey::Count` instead of `"COUNT"`
3. **Will panic on first message** - Should use `.unwrap_or(0)` instead of `.unwrap()`
4. **Wrong storage key** - Should use `DataKey::Message(count)` not `"MESSAGE"`
5. **Counter not updated** - Missing `set(&DataKey::Count, &count)`
6. **No error handling** - Multiple `.unwrap()` calls without defaults
</details>

---

## Additional Resources

### Official Documentation
- [Soroban Documentation](https://soroban.stellar.org/)
- [Soroban by Example](https://soroban.stellar.org/docs/examples)
- [Stellar Developer Discord](https://discord.gg/stellar)

### Related Tutorials
- **Previous:** Increment Contract - Simple Storage Patterns
- **Next:** Task Manager Contract - State Machines and Enums
- **Advanced:** Escrow Contract - Multi-Party Authorization

### Video Tutorials
- [Soroban Basics Playlist](https://soroban.stellar.org/)
- [Understanding Authentication in Smart Contracts](https://soroban.stellar.org/)

### Further Reading
- [Smart Contract Security Best Practices](https://soroban.stellar.org/)
- [Rust Ownership and Borrowing](https://doc.rust-lang.org/book/)
- [WebAssembly in Blockchain](https://webassembly.org/)

---

## Summary

### What You've Learned

In this lesson, you've mastered:

**Core Concepts:**
- Authentication with `require_auth()`
- Typed storage keys using enums
- Custom data structures with `#[contracttype]`
- Multi-user contract interactions

**Practical Skills:**
- Building contracts with structured data
- Implementing security checks
- Writing comprehensive tests
- Handling errors appropriately

**Best Practices:**
- Always authenticate user actions
- Use typed keys instead of strings
- Provide default values for storage reads
- Test both success and failure cases

### Key Takeaways

1. **Security First** - Always use `require_auth()` for user-specific actions
2. **Type Safety** - Typed enums prevent runtime errors
3. **Test Thoroughly** - Test success, failure, and edge cases
4. **Think About Scale** - Design patterns that work for 1 or 1,000,000 users

### Next Steps

You're now ready to move on to the **Task Manager Contract**, where you'll learn:
- State machines with enums
- More complex authorization patterns
- Status transitions and validation
- Building multi-step workflows

---

## Homework Assignment

### Part 1: Extend the Guest Book

Implement **all three** practice exercises:
1. Add timestamps to messages
2. Prevent empty messages
3. Add a delete function (author-only)

### Part 2: Written Reflection

Answer these questions (2-3 sentences each):

1. Why is `require_auth()` critical for blockchain applications?
2. How do typed storage keys improve code quality?
3. What are the advantages of panicking vs returning error types in Soroban?
4. Describe a real-world application where this guest book pattern could be used



