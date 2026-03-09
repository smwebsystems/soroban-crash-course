# Task Manager Smart Contract Tutorial

> **Course Module:** Soroban Crash Course  
---

## Prerequisites

Before starting this tutorial, you should have completed:

1. **Hello World Contract** - Basic contract structure
2. **Increment Contract** - Simple storage operations
3. **Guest Book Contract** - Authentication and structured data with typed storage keys

### Required Knowledge

- Authentication with `require_auth()`
- Custom data structures using `#[contracttype]`
- Typed storage keys using enums
- Storage operations (`get`, `set`)
- Writing and running tests
- Understanding of addresses and ownership

### Development Environment

Ensure you have installed:
- Rust toolchain (1.70 or later)
- Soroban CLI
- Code editor with Rust support (VS Code recommended)

---

## Learning Objectives

By the end of this lesson, you will be able to:

**Knowledge**
- Understand how to use enums to represent different states
- Explain state transition rules and validation
- Describe the difference between simple and complex authorization
- Understand the concept of resource ownership in smart contracts

**Skills**
- Implement state machines using Rust enums
- Create authorization logic that checks ownership
- Update existing data in contract storage
- Validate state transitions before allowing changes
- Return success/failure indicators from functions
- Apply typed storage key patterns learned in Guest Book

**Application**
- Build task management systems with status tracking
- Implement ownership-based access control
- Design state machines for real-world workflows
- Handle edge cases like duplicate operations

---

## Introduction

### What Is a Task Manager Contract?

A **Task Manager** is an interactive application where users can:
- Create tasks with descriptions
- Track task status (Pending or Completed)
- Mark their own tasks as complete
- Prevent unauthorized users from modifying others' tasks

Think of it like a to-do list app, but on the blockchain with built-in security!

### Why Build This?

This contract introduces critical patterns you'll use in advanced smart contracts:

1. **State Machines** - Using enums to represent different states
2. **Ownership Validation** - Checking who owns a resource
3. **State Transitions** - Rules for changing from one state to another
4. **Update Operations** - Modifying existing data safely
5. **Typed Storage Keys** - Building on the pattern from Guest Book

### Real-World Applications

The patterns you learn here apply to:
- Project management systems
- Workflow automation
- Order processing (Placed → Shipped → Delivered)
- Document approval flows (Draft → Review → Approved)
- Gaming (character states, quest progress)
- NFT marketplaces (Listed → Sold)

---

## Conceptual Overview

### Building on Guest Book

In the **Guest Book**, you learned about authentication, storing data, and using typed storage keys. But what about:
- What if data needs to change over time?
- What if only certain users can make changes?
- What if changes follow specific rules?

### The New Challenges

**Questions to consider:**
- How do we track whether a task is done or not?
- How do we prevent Alice from completing Bob's tasks?
- How do we prevent marking a task complete twice?
- How do we ensure data updates follow business rules?

### Our Solution

We'll build a system that:
1. Uses **enums** to represent task status (Pending vs Completed)
2. Implements **ownership checks** to verify who can modify tasks
3. Enforces **state transition rules** (can't complete twice)
4. Returns **success/failure indicators** instead of always panicking
5. Uses **typed storage keys** for type safety (like Guest Book)

### Comparison with Previous Lessons

| Feature | Guest Book | Task Manager |
|---------|------------|--------------|
| Data mutability | Append-only | Can update |
| Authorization | Simple (is author) | Complex (is owner) |
| State | No state tracking | Status enum |
| State transitions | None | Pending → Completed |
| Error handling | Panic on error | Return success/failure |
| Storage keys | Typed enum (DataKey) | Typed enum (StorageKey) |
| Validation | Basic | Multiple checks |

---

## Contract Architecture

### Data Model

Our contract uses three main data structures:

#### 1. Storage Keys Enum
```rust
pub enum StorageKey {
    TaskCount,
    Task(u32),
}
```

**Building on Guest Book's pattern:**
- `TaskCount` - Stores total number of tasks
- `Task(u32)` - Stores individual tasks by ID

**Why this is better than strings:**
- Type-safe (compiler catches typos)
- Self-documenting code
- Easy to refactor
- Consistent with best practices

#### 2. Task Status Enum
```rust
pub enum TaskStatus {
    Pending,
    Completed,
}
```

**Why an enum?**
- Represents mutually exclusive states
- Type-safe (can't be "Pendng" or "Complete")
- Easy to add more states later (InProgress, Cancelled, etc.)

**Think of it like a traffic light:**
- Only one state at a time (can't be both Red and Green)
- Clear, distinct states
- Known transitions between states

#### 3. Task Structure
```rust
pub struct Task {
    pub id: u32,          // Unique identifier
    pub owner: Address,   // Who created it
    pub description: String,  // What needs to be done
    pub status: TaskStatus,   // Current state
}
```

### State Machine Diagram
```
Task Lifecycle:
                                    
    CREATE                         COMPLETE
      │                               │
      ▼                               ▼
┌──────────┐                   ┌──────────┐
│ Pending  │ ─────────────────>│Completed │
└──────────┘                   └──────────┘
      │                               │
      └───────── Can't go back ───────┘
```

**State Transition Rules:**
- New tasks start as `Pending`
- `Pending` can transition to `Completed`
- `Completed` cannot transition back to `Pending`
- Only the owner can trigger transitions

### Storage Layout
```
Contract Storage
├── StorageKey::TaskCount → 3
├── StorageKey::Task(1) → Task { id: 1, owner: Alice, description: "Write docs", status: Pending }
├── StorageKey::Task(2) → Task { id: 2, owner: Bob, description: "Review PR", status: Completed }
└── StorageKey::Task(3) → Task { id: 3, owner: Alice, description: "Deploy", status: Pending }
```

### Contract Functions

| Function | Purpose | Authorization | Returns |
|----------|---------|---------------|---------|
| `create_task(owner, desc)` | Create new task | Requires owner auth | Task ID (u32) |
| `complete_task(owner, id)` | Mark task complete | Requires owner auth + ownership check | Success (bool) |
| `get_task(id)` | Retrieve task | Public | Task struct |
| `get_task_count()` | Get total tasks | Public | Count (u32) |

---

## Building the Smart Contract

Let's build this contract step by step, introducing new concepts as we go.

---

### Step 1: Set Up the Foundation

Start with imports and basic structure:
```rust
#![no_std]
use soroban_sdk::{contract, contractimpl, contracttype, log, Address, Env, String};
```

**What's here:**

Everything is familiar from Guest Book! We're using the same imports:
- `contract`, `contractimpl`, `contracttype` - Contract macros
- `log` - For logging events
- `Address`, `Env`, `String` - Core types

**No surprises yet** - we're building on what you know.

---

### Step 2: Define the Storage Keys Enum

First, let's set up our type-safe storage keys (just like Guest Book):
```rust
/// Storage keys enum - safer than string literals
#[contracttype]
#[derive(Clone)]
pub enum StorageKey {
    TaskCount,
    Task(u32), // Task by ID
}
```

**Understanding the storage keys:**

**Why use an enum?**

This is the **same pattern** you learned in Guest Book! Remember:

| Approach | Example | Problems |
|----------|---------|----------|
| String keys | `"COUNT"`, `"TASK_1"` | Typos cause runtime errors |
| Typed enum | `StorageKey::TaskCount`, `StorageKey::Task(1)` | Typos cause compile errors |

**The Variants:**

**`TaskCount`**
- Stores the total number of tasks created
- Simple variant with no data (like `DataKey::Count` in Guest Book)

**`Task(u32)`**
- Stores individual tasks by their ID number
- Variant that holds a number (like `DataKey::Message(u32)` in Guest Book)
- `Task(1)`, `Task(2)`, `Task(3)`, etc.

**Comparison with Guest Book:**
```rust
// Guest Book:
pub enum DataKey {
    Count,           // Total messages
    Message(u32),    // Individual messages
}

// Task Manager:
pub enum StorageKey {
    TaskCount,       // Total tasks
    Task(u32),       // Individual tasks
}
```

**Same concept, different names!**

**Derived Traits:**
```rust
#[derive(Clone)]
```
- We only need `Clone` for storage operations
- Don't need `Debug`, `Eq`, `PartialEq` for this enum (unlike Task)

**Why "StorageKey" instead of "DataKey"?**

Just a naming choice - both are valid! The contract uses `StorageKey` to be more descriptive, but `DataKey` from Guest Book works just as well.

---

### Step 3: Define the Task Status Enum

Now for our first new concept - state representation:
```rust
/// Represents the current state of a task
#[contracttype]
#[derive(Clone, PartialEq, Debug)]
pub enum TaskStatus {
    Pending,
    Completed,
}
```

**Understanding enums for state:**

**`enum TaskStatus`**
- Defines a type that can be one of several variants
- Each variant represents a distinct state
- Can only be one variant at a time

**The Variants:**
- `Pending` - Task is not yet done
- `Completed` - Task is finished

**Derived Traits:**
- `Clone` - Can make copies
- `PartialEq` - Can compare for equality (needed for `if task.status == TaskStatus::Completed`)
- `Debug` - Can print for debugging

**Why use an enum instead of a boolean?**

| Approach | Example | Extensibility |
|----------|---------|---------------|
| Boolean | `is_completed: bool` | Hard to add states (what about "InProgress"?) |
| Enum | `status: TaskStatus` | Easy to add variants (InProgress, Cancelled, etc.) |

**Example usage:**
```rust
// Creating a status
let status = TaskStatus::Pending;

// Comparing statuses
if status == TaskStatus::Pending {
    // Do something
}

// Changing status
let new_status = TaskStatus::Completed;
```

**Real-world analogy:** Like a package tracking system:
- Order Placed
- Shipped
- Out for Delivery
- Delivered

Each is a distinct state, not just "delivered: true/false"

---

### Step 4: Define the Task Structure

Now let's create the data structure for a complete task:
```rust
/// A task with an owner, description, and status
#[contracttype]
#[derive(Clone, Debug)]
pub struct Task {
    pub id: u32,
    pub owner: Address,
    pub description: String,
    pub status: TaskStatus,
}
```

**Understanding the Fields:**

**`id: u32`**
- Unique identifier for each task
- Sequential numbering (1, 2, 3, ...)
- Similar to Guest Book message IDs

**`owner: Address`**
- The blockchain address of who created the task
- Used to verify who can modify it
- Cannot be changed after creation

**`description: String`**
- What the task is about
- Free-form text
- Could be empty (though you might validate this)

**`status: TaskStatus`**
- Current state of the task
- Uses our enum (Pending or Completed)
- Can change over the task's lifetime

**Why no `PartialEq` for Task?**

Notice we derive `Clone, Debug` but not `PartialEq`. This is because:
- We don't need to compare entire tasks
- We only compare specific fields (like status)
- This is a design choice (we could add it if needed)

**Comparison with Guest Book Message:**

| Guest Book Message | Task Manager Task |
|-------------------|-------------------|
| `author: Address` | `owner: Address` |
| `text: String` | `description: String` |
| No state field | `status: TaskStatus` |
| Read-only | Can be updated |

---

### Step 5: Create the Contract Shell

Set up the contract structure:
```rust
#[contract]
pub struct TaskManager;

#[contractimpl]
impl TaskManager {
    // Functions will go here
}
```

**This is identical to Guest Book:**
- Empty struct as a namespace
- All functions defined in the implementation block
- Data lives in contract storage, not in the struct

---

### Step 6: Implement `create_task`

Let's create our first function - adding new tasks:
```rust
/// Create a new task
pub fn create_task(env: Env, owner: Address, description: String) -> u32 {
    owner.require_auth();

    // Use enum for storage key
    let mut count: u32 = env
        .storage()
        .instance()
        .get(&StorageKey::TaskCount)
        .unwrap_or(0);

    count += 1;

    let task = Task {
        id: count,
        owner: owner.clone(),
        description,
        status: TaskStatus::Pending,
    };

    // Use enum variant for task storage
    env.storage()
        .instance()
        .set(&StorageKey::Task(count), &task);
    env.storage().instance().set(&StorageKey::TaskCount, &count);

    log!(&env, "Task created with ID: {}", count);
    count
}
```

**Breaking it down:**

---

#### Part A: Function Signature
```rust
pub fn create_task(env: Env, owner: Address, description: String) -> u32
```

**New concept: Return value!**
- `-> u32` means this function returns a number
- Returns the ID of the newly created task
- Caller can use this ID immediately

**Why return the ID?**
- Convenient for the caller
- They can immediately reference the task
- No need for a separate "get_last_task_id" function

**Comparison:**
```rust
// Guest Book (no return):
post_message(author, text);
// How do I know what ID it got?

// Task Manager (with return):
let task_id = create_task(owner, description);
// I know the ID immediately!
```

---

#### Part B: Authentication
```rust
owner.require_auth();
```

**Same pattern as Guest Book:**
- Verify the owner authorized this transaction
- Prevents creating tasks on behalf of others
- First line of security

---

#### Part C: Counter Logic with Typed Keys
```rust
// Use enum for storage key
let mut count: u32 = env
    .storage()
    .instance()
    .get(&StorageKey::TaskCount)
    .unwrap_or(0);

count += 1;
```

**Using typed storage keys:**
```rust
.get(&StorageKey::TaskCount)
```

**This is exactly like Guest Book!**
```rust
// Guest Book:
env.storage().instance().get(&DataKey::Count).unwrap_or(0)

// Task Manager:
env.storage().instance().get(&StorageKey::TaskCount).unwrap_or(0)
```

**Benefits of typed keys:**
- `StorageKey::TaskCount` is checked by the compiler
- Can't accidentally type `StorageKey::TasCount` (compile error!)
- Self-documenting code
- Easy to refactor

**What if we used strings?**
```rust
// Bad - string keys:
.get(&"COUNT")      // Could be "count", "Count", "CONT", etc.
.get(&"TASK_COUNT") // Different string = different key!
.get(&"TaskCount")  // Yet another different key!
```

---

#### Part D: Create the Task
```rust
let task = Task {
    id: count,
    owner: owner.clone(),
    description,
    status: TaskStatus::Pending,
};
```

**Building the task:**
- `id: count` - Use the incremented counter
- `owner: owner.clone()` - Copy the owner address
- `description` - Move the description in
- `status: TaskStatus::Pending` - **Always start as Pending**

**Critical design decision:**
```rust
status: TaskStatus::Pending,  // New tasks are always Pending
```

We don't let the caller specify the initial status. Why?
- Enforces consistent initialization
- Prevents creating "already completed" tasks
- Makes the state machine predictable

**Wrong approach:**
```rust
// Bad - caller could cheat:
pub fn create_task(env: Env, owner: Address, description: String, status: TaskStatus) {
    // Caller could create a task that's already Completed!
}
```

---

#### Part E: Store with Typed Keys
```rust
// Use enum variant for task storage
env.storage()
    .instance()
    .set(&StorageKey::Task(count), &task);
env.storage().instance().set(&StorageKey::TaskCount, &count);
```

**Using typed keys for storage:**
```rust
.set(&StorageKey::Task(count), &task)
```

**This is the same pattern as Guest Book!**
```rust
// Guest Book:
env.storage().instance().set(&DataKey::Message(count), &message);

// Task Manager:
env.storage().instance().set(&StorageKey::Task(count), &task);
```

**Type safety in action:**
```rust
// Correct:
StorageKey::Task(1)    // ✓
StorageKey::Task(2)    // ✓
StorageKey::TaskCount  // ✓

// Wrong (won't compile):
StorageKey::Tsk(1)     // ✗ No variant 'Tsk'
StorageKey::Task       // ✗ Missing parameter
"TASK_1"               // ✗ Wrong type
```

**Storage state after creating first task:**
```
Before:
Storage: {}

After:
Storage: {
    StorageKey::TaskCount → 1,
    StorageKey::Task(1) → Task { id: 1, owner: Alice, description: "Learn Soroban", status: Pending }
}
```

---

#### Part F: Log and Return
```rust
log!(&env, "Task created with ID: {}", count);
count
```

**Final steps:**
1. Log the event for transparency
2. Return the task ID to the caller

**The return statement:**
```rust
count  // No semicolon! This is the return value
```

In Rust, the last expression (without semicolon) is returned.

---

### Step 7: Implement `complete_task` - The Complex Function

This is where things get interesting. We need to:
1. Verify the caller is authenticated
2. Load the existing task (using typed key!)
3. Check if the caller is the owner
4. Check if already completed
5. Update the status
6. Save the changes (using typed key!)
```rust
/// Mark a task as completed
pub fn complete_task(env: Env, owner: Address, task_id: u32) -> bool {
    owner.require_auth();

    let mut task: Task = env
        .storage()
        .instance()
        .get(&StorageKey::Task(task_id))
        .expect("Task not found");

    if task.owner != owner {
        log!(&env, "Unauthorized: caller is not task owner");
        return false;
    }

    if task.status == TaskStatus::Completed {
        log!(&env, "Task already completed");
        return false;
    }

    task.status = TaskStatus::Completed;
    env.storage()
        .instance()
        .set(&StorageKey::Task(task_id), &task);

    log!(&env, "Task {} marked as completed", task_id);
    true
}
```

Let's break down each part:

---

#### Part A: Function Signature and Return Type
```rust
pub fn complete_task(env: Env, owner: Address, task_id: u32) -> bool
```

**Parameters:**
- `env: Env` - Contract environment
- `owner: Address` - Who is trying to complete the task
- `task_id: u32` - Which task to complete

**Return type: `bool`**
- `true` if successful
- `false` if failed (wrong owner, already completed, etc.)

**New pattern: Returning success/failure instead of panicking!**

**Comparison:**
```rust
// Guest Book approach - panic on error:
pub fn get_message(env: Env, id: u32) -> Message {
    env.storage().get(&id).expect("Message not found")
    // Panics if not found
}

// Task Manager approach - return bool:
pub fn complete_task(env: Env, owner: Address, id: u32) -> bool {
    // Returns false on failure
    // Returns true on success
}
```

**Why use bool instead of panicking?**

Some operations are **expected to fail** sometimes:
- Trying to complete someone else's task - not an error, just not allowed
- Trying to complete an already-completed task - not catastrophic, just redundant

**When to panic vs return bool:**
```rust
// Panic: Unexpected errors
get_task(999) // Task doesn't exist - unexpected, panic!

// Return bool: Expected failures
complete_task(wrong_owner, 1) // Wrong owner - expected scenario, return false
```

---

#### Part B: Authentication
```rust
owner.require_auth();
```

**Same as always** - verify the caller is who they claim to be.

---

#### Part C: Load the Task with Typed Key
```rust
let mut task: Task = env
    .storage()
    .instance()
    .get(&StorageKey::Task(task_id))
    .expect("Task not found");
```

**Using typed storage key:**
```rust
.get(&StorageKey::Task(task_id))
```

**Type safety prevents errors:**
```rust
// Correct:
StorageKey::Task(1)     // ✓ Gets task 1
StorageKey::Task(task_id)  // ✓ Uses variable

// Won't compile:
"TASK_1"                // ✗ Wrong type
StorageKey::Task        // ✗ Missing ID
1                       // ✗ Wrong type
```

**What's happening:**

**`let mut task: Task`**
- Declaring a **mutable** variable
- We need `mut` because we'll change the status
- Type annotation `: Task` helps readability

**Loading from storage:**
```rust
env.storage().instance().get(&StorageKey::Task(task_id))
```
- Look up the task by its ID using typed key
- Returns `Option<Task>`

**Error handling:**
```rust
.expect("Task not found")
```
- If task exists: unwrap and use it
- If task doesn't exist: **panic** with this message

**Why panic here but return bool later?**
```rust
// Panic if task doesn't exist - unexpected error
let task = get(&StorageKey::Task(task_id)).expect("Task not found");

// Return false if wrong owner - expected scenario
if task.owner != owner {
    return false;
}
```

**Logical distinction:**
- Task not existing = data integrity issue (should exist)
- Wrong owner = authorization issue (expected to happen)

---

#### Part D: Ownership Validation
```rust
// Check if the caller is the owner
if task.owner != owner {
    log!(&env, "Unauthorized: caller is not task owner");
    return false;
}
```

**This is the key authorization check!**

**Step by step:**

1. **Compare addresses:**
```rust
   task.owner != owner
```
   - `task.owner` - Address stored when task was created
   - `owner` - Address of the caller
   - `!=` - Not equal operator

2. **If they don't match:**
```rust
   log!(&env, "Unauthorized: caller is not task owner");
   return false;
```
   - Log the failed attempt
   - Return `false` to indicate failure
   - **Don't panic** - this is expected behavior

**Example scenario:**
```rust
// Alice creates a task
create_task(alice_address, "My task")  // Task 1, owned by Alice

// Bob tries to complete Alice's task
complete_task(bob_address, 1)
    → task.owner (Alice) != owner (Bob)
    → Returns false
    → Transaction succeeds, but function indicates failure

// Alice completes her own task
complete_task(alice_address, 1)
    → task.owner (Alice) == owner (Alice) ✓
    → Continues to next check
```

**Why this matters:**

Without this check:
```rust
// Without ownership check - BAD:
pub fn complete_task(env: Env, owner: Address, task_id: u32) -> bool {
    owner.require_auth();
    // Anyone who's authenticated can complete any task!
    
    let mut task = get_task(task_id);
    task.status = TaskStatus::Completed;  // No owner check!
    save_task(task);
}
```

**Two-level authorization:**
```rust
// Level 1: Authentication
owner.require_auth();  // Proves you are who you say you are

// Level 2: Ownership
if task.owner != owner {  // Proves you own this resource
    return false;
}
```

---

#### Part E: Prevent Double Completion
```rust
// Check if already completed
if task.status == TaskStatus::Completed {
    log!(&env, "Task already completed");
    return false;
}
```

**State validation check:**

**Why this matters:**
```rust
// Without this check:
complete_task(alice, 1)  // Success - Pending → Completed
complete_task(alice, 1)  // Success again? But already completed!
complete_task(alice, 1)  // Waste of gas, redundant operation
```

**With the check:**
```rust
complete_task(alice, 1)  // Success - Pending → Completed
complete_task(alice, 1)  // Returns false - already completed
complete_task(alice, 1)  // Returns false - already completed
```

**Using the enum for comparison:**
```rust
if task.status == TaskStatus::Completed
```
- Compares the current status to the Completed variant
- Type-safe comparison (can't compare to "Completed" string)
- Clear and readable

**State machine enforcement:**

This check enforces our state transition rule:
```
Pending ──────> Completed
    ▲               │
    │               │
    └── Can't ──────┘
       go back
```

---

#### Part F: Update and Save with Typed Key
```rust
task.status = TaskStatus::Completed;
env.storage()
    .instance()
    .set(&StorageKey::Task(task_id), &task);
```

**Updating the status:**
```rust
task.status = TaskStatus::Completed;
```
- Modify the in-memory task
- Change from `Pending` to `Completed`
- This is why we needed `let mut task`

**Saving back to storage with typed key:**
```rust
env.storage()
    .instance()
    .set(&StorageKey::Task(task_id), &task);
```

**Type safety in action:**
```rust
// Correct:
.set(&StorageKey::Task(task_id), &task)  // ✓

// Won't compile:
.set(&task_id, &task)                    // ✗ Wrong type
.set(&"TASK_1", &task)                   // ✗ String not allowed
.set(&StorageKey::TaskCount, &task)      // ✗ Wrong variant
```

**Important:** Write the modified task back to storage
- **Critical step** - without this, changes are lost!
- Overwrites the old task with the updated one

**Before and after:**
```rust
// Before complete_task:
Storage: {
    StorageKey::Task(1) → Task { id: 1, owner: Alice, desc: "Learn", status: Pending }
}

// After complete_task:
Storage: {
    StorageKey::Task(1) → Task { id: 1, owner: Alice, desc: "Learn", status: Completed }
}
```

**Common mistake:**
```rust
// Wrong - forgot to save:
task.status = TaskStatus::Completed;
// Missing: env.storage().instance().set(&StorageKey::Task(task_id), &task);
log!(&env, "Task {} marked as completed", task_id);
return true;

// Changes are lost! Storage still has Pending status
```

---

#### Part G: Log and Return Success
```rust
log!(&env, "Task {} marked as completed", task_id);
true
```

**Final steps:**
1. Log the successful completion
2. Return `true` to indicate success

**The complete flow:**
```
complete_task(owner, task_id)
        │
        ▼
    require_auth() ✓
        │
        ▼
    Load task from storage (typed key)
        │
        ▼
    Check ownership ✓
        │
        ▼
    Check not already completed ✓
        │
        ▼
    Update status: Pending → Completed
        │
        ▼
    Save to storage (typed key)
        │
        ▼
    Log event
        │
        ▼
    Return true
```

**All the ways this function can fail:**

| Failure Reason | What Happens |
|----------------|--------------|
| Task doesn't exist | Panic: "Task not found" |
| Wrong owner | Return false, log "Unauthorized" |
| Already completed | Return false, log "Task already completed" |
| Success | Return true, log "Task marked as completed" |

---

### Step 8: Implement Helper Functions with Typed Keys

Now let's add the simpler read-only functions:

#### Get Task Function
```rust
/// Get a specific task
pub fn get_task(env: Env, task_id: u32) -> Task {
    env.storage()
        .instance()
        .get(&StorageKey::Task(task_id))
        .expect("Task not found")
}
```

**Using typed storage key:**
```rust
.get(&StorageKey::Task(task_id))
```

**Same pattern as Guest Book's `get_message`:**
```rust
// Guest Book:
env.storage().instance().get(&DataKey::Message(message_id))

// Task Manager:
env.storage().instance().get(&StorageKey::Task(task_id))
```

**Benefits:**
- Type-safe
- Can't accidentally use wrong key
- Self-documenting

---

#### Get Task Count Function
```rust
/// Get total number of tasks
pub fn get_task_count(env: Env) -> u32 {
    env.storage()
        .instance()
        .get(&StorageKey::TaskCount)
        .unwrap_or(0)
}
```

**Using typed storage key:**
```rust
.get(&StorageKey::TaskCount)
```

**Same pattern as Guest Book's `get_message_count`:**
```rust
// Guest Book:
env.storage().instance().get(&DataKey::Count).unwrap_or(0)

// Task Manager:
env.storage().instance().get(&StorageKey::TaskCount).unwrap_or(0)
```

**Consistency across contracts!**

---

### Complete Contract Code

Now let's see the entire contract together:
```rust
#![no_std]
use soroban_sdk::{contract, contractimpl, contracttype, log, Address, Env, String};

/// Storage keys enum - safer than string literals
#[contracttype]
#[derive(Clone)]
pub enum StorageKey {
    TaskCount,
    Task(u32), // Task by ID
}

#[contracttype]
#[derive(Clone, PartialEq, Debug)]
pub enum TaskStatus {
    Pending,
    Completed,
}

#[contracttype]
#[derive(Clone, Debug)]
pub struct Task {
    pub id: u32,
    pub owner: Address,
    pub description: String,
    pub status: TaskStatus,
}

#[contract]
pub struct TaskManager;

#[contractimpl]
impl TaskManager {
    /// Create a new task
    pub fn create_task(env: Env, owner: Address, description: String) -> u32 {
        owner.require_auth();

        // Use enum for storage key
        let mut count: u32 = env
            .storage()
            .instance()
            .get(&StorageKey::TaskCount)
            .unwrap_or(0);

        count += 1;

        let task = Task {
            id: count,
            owner: owner.clone(),
            description,
            status: TaskStatus::Pending,
        };

        // Use enum variant for task storage
        env.storage()
            .instance()
            .set(&StorageKey::Task(count), &task);
        env.storage().instance().set(&StorageKey::TaskCount, &count);

        log!(&env, "Task created with ID: {}", count);
        count
    }

    /// Mark a task as completed
    pub fn complete_task(env: Env, owner: Address, task_id: u32) -> bool {
        owner.require_auth();

        let mut task: Task = env
            .storage()
            .instance()
            .get(&StorageKey::Task(task_id))
            .expect("Task not found");

        if task.owner != owner {
            log!(&env, "Unauthorized: caller is not task owner");
            return false;
        }

        if task.status == TaskStatus::Completed {
            log!(&env, "Task already completed");
            return false;
        }

        task.status = TaskStatus::Completed;
        env.storage()
            .instance()
            .set(&StorageKey::Task(task_id), &task);

        log!(&env, "Task {} marked as completed", task_id);
        true
    }

    /// Get a specific task
    pub fn get_task(env: Env, task_id: u32) -> Task {
        env.storage()
            .instance()
            .get(&StorageKey::Task(task_id))
            .expect("Task not found")
    }

    /// Get total number of tasks
    pub fn get_task_count(env: Env) -> u32 {
        env.storage()
            .instance()
            .get(&StorageKey::TaskCount)
            .unwrap_or(0)
    }
}
```

**Contract Summary:**

| Component | Purpose |
|-----------|---------|
| `StorageKey` enum | Type-safe storage keys (like Guest Book) |
| `TaskStatus` enum | Represents task state (Pending/Completed) |
| `Task` struct | Complete task data with status |
| `create_task` | Add new tasks (returns ID) |
| `complete_task` | Update task status (returns bool) |
| `get_task` | Retrieve task by ID |
| `get_task_count` | Get total count |

---

## Testing Your Contract

Now let's learn how to comprehensively test state transitions and authorization logic.

---

### Test Module Setup

The tests are in a separate module:
```rust
mod test;
```

And the test file begins with:
```rust
#![cfg(test)]
use super::*;
use soroban_sdk::{testutils::Address as _, Env, String};
```

**What's new:**
- Tests can be in a separate file/module
- `#![cfg(test)]` at the top of the test file
- Import from parent module with `use super::*`

**Same setup patterns as Guest Book!**

---

### Test 1: Create Single Task

Let's verify basic task creation:
```rust
#[test]
fn test_create_task() {
    let env = Env::default();

    let contract_id = env.register(TaskManager, ());
    let client = TaskManagerClient::new(&env, &contract_id);

    let owner = Address::generate(&env);
    let description = String::from_str(&env, "Write Soroban tests");

    // Mock auth for owner
    env.mock_all_auths();

    let task_id = client.create_task(&owner, &description);

    assert_eq!(task_id, 1);
    assert_eq!(client.get_task_count(), 1);

    let task = client.get_task(&task_id);
    assert_eq!(task.id, 1);
    assert_eq!(task.owner, owner);
    assert_eq!(task.description, description);
    assert_eq!(task.status, TaskStatus::Pending);
}
```

**What we're testing:**

---

#### Setup
```rust
let env = Env::default();
let contract_id = env.register(TaskManager, ());
let client = TaskManagerClient::new(&env, &contract_id);

let owner = Address::generate(&env);
let description = String::from_str(&env, "Write Soroban tests");

env.mock_all_auths();
```

**Standard test setup:**
- Create environment
- Register contract
- Create client
- Generate test data
- Mock authentication

**All familiar from Guest Book!**

---

#### Capture Return Value
```rust
let task_id = client.create_task(&owner, &description);
```

**New pattern:** Store the returned task ID

---

#### Verify Return Value
```rust
assert_eq!(task_id, 1);
```

- First task should return ID `1`
- Tests the return value

---

#### Verify Count
```rust
assert_eq!(client.get_task_count(), 1);
```

- Total count should be `1`
- Tests counter logic

---

#### Verify Task Data
```rust
let task = client.get_task(&task_id);
assert_eq!(task.id, 1);
assert_eq!(task.owner, owner);
assert_eq!(task.description, description);
assert_eq!(task.status, TaskStatus::Pending);
```

**Comprehensive verification:**
- ID matches
- Owner matches
- Description matches
- **Status is Pending** (tests state initialization)

---

### Test 2: Create Multiple Tasks

Tests that counter increments correctly:
```rust
#[test]
fn test_create_multiple_tasks() {
    let env = Env::default();

    let contract_id = env.register(TaskManager, ());
    let client = TaskManagerClient::new(&env, &contract_id);

    let owner1 = Address::generate(&env);
    let owner2 = Address::generate(&env);

    env.mock_all_auths();

    let task1_id = client.create_task(&owner1, &String::from_str(&env, "Task 1"));
    let task2_id = client.create_task(&owner2, &String::from_str(&env, "Task 2"));
    let task3_id = client.create_task(&owner1, &String::from_str(&env, "Task 3"));

    assert_eq!(task1_id, 1);
    assert_eq!(task2_id, 2);
    assert_eq!(task3_id, 3);
    assert_eq!(client.get_task_count(), 3);
}
```

**What we're testing:**
- Sequential ID assignment
- Counter increments correctly
- Works across multiple users
- Total count is accurate

**Expected storage:**
```
StorageKey::TaskCount → 3
StorageKey::Task(1) → owner1's task
StorageKey::Task(2) → owner2's task
StorageKey::Task(3) → owner1's task
```

---

### Test 3: Complete Task

Tests the complete lifecycle:
```rust
#[test]
fn test_complete_task() {
    let env = Env::default();
    let contract_id = env.register(TaskManager, ());

    let client = TaskManagerClient::new(&env, &contract_id);

    let owner = Address::generate(&env);

    env.mock_all_auths();

    let task_id = client.create_task(&owner, &String::from_str(&env, "Test task"));

    // Complete the task
    let result = client.complete_task(&owner, &task_id);
    assert_eq!(result, true);

    // Verify status changed
    let task = client.get_task(&task_id);
    assert_eq!(task.status, TaskStatus::Completed);
}
```

**What we're testing:**

---

#### Create Task
```rust
let task_id = client.create_task(&owner, &String::from_str(&env, "Test task"));
```

- Task starts in `Pending` state

---

#### Complete Task
```rust
let result = client.complete_task(&owner, &task_id);
assert_eq!(result, true);
```

**Testing bool return:**
- Function should return `true` (success)
- Using `assert_eq!(result, true)`

---

#### Verify State Change
```rust
let task = client.get_task(&task_id);
assert_eq!(task.status, TaskStatus::Completed);
```

- Load task again
- Status should be `Completed`
- **Tests state transition!**

---

### Test 4: Cannot Complete Twice

Tests state validation:
```rust
#[test]
fn test_complete_task_twice() {
    let env = Env::default();
    let contract_id = env.register(TaskManager, ());

    let client = TaskManagerClient::new(&env, &contract_id);

    let owner = Address::generate(&env);

    env.mock_all_auths();

    let task_id = client.create_task(&owner, &String::from_str(&env, "Test task"));

    // Complete once - should succeed
    assert_eq!(client.complete_task(&owner, &task_id), true);

    // Complete again - should fail
    assert_eq!(client.complete_task(&owner, &task_id), false);
}
```

**What we're testing:**

**First completion:**
```rust
assert_eq!(client.complete_task(&owner, &task_id), true);
```
- Returns `true` (success)
- Status changes: Pending → Completed

**Second completion:**
```rust
assert_eq!(client.complete_task(&owner, &task_id), false);
```
- Returns `false` (expected failure)
- Status check catches this
- **Tests state machine rules!**

**State transition enforcement:**
```
First call:  Pending → Completed ✓
Second call: Completed → Completed ✗ (rejected)
```

---

### Test 5: Unauthorized Completion

Tests ownership validation:
```rust
#[test]
fn test_complete_task_unauthorized() {
    let env = Env::default();
    let contract_id = env.register(TaskManager, ());

    let client = TaskManagerClient::new(&env, &contract_id);

    let owner = Address::generate(&env);
    let unauthorized_user = Address::generate(&env);

    env.mock_all_auths();

    let task_id = client.create_task(&owner, &String::from_str(&env, "Test task"));

    // Try to complete as different user - should fail
    let result = client.complete_task(&unauthorized_user, &task_id);
    assert_eq!(result, false);

    // Verify status unchanged
    let task = client.get_task(&task_id);
    assert_eq!(task.status, TaskStatus::Pending);
}
```

**What we're testing:**

---

#### Create Two Users
```rust
let owner = Address::generate(&env);
let unauthorized_user = Address::generate(&env);
```

- Owner who created the task
- Different user trying to modify it

---

#### Create as Owner
```rust
let task_id = client.create_task(&owner, &String::from_str(&env, "Test task"));
```

- Task belongs to `owner`

---

#### Try to Complete as Wrong User
```rust
let result = client.complete_task(&unauthorized_user, &task_id);
assert_eq!(result, false);
```

**Critical test:**
- Unauthorized user tries to complete
- Ownership check fails
- Returns `false`
- **Tests authorization!**

---

#### Verify No Changes
```rust
let task = client.get_task(&task_id);
assert_eq!(task.status, TaskStatus::Pending);
```

- Status still `Pending`
- Unauthorized attempt didn't change anything
- **Data integrity preserved!**

---

### Test 6: Get Nonexistent Task

Tests error handling:
```rust
#[test]
#[should_panic(expected = "Task not found")]
fn test_get_nonexistent_task() {
    let env = Env::default();
    let contract_id = env.register(TaskManager, ());

    let client = TaskManagerClient::new(&env, &contract_id);

    // Try to get a task that doesn't exist
    client.get_task(&999);
}
```

**Using `#[should_panic]`:**
- Test expects a panic
- Verifies error message matches
- Tests error handling

**Same pattern as Guest Book!**

---

### Test 7: Complete Nonexistent Task
```rust
#[test]
#[should_panic(expected = "Task not found")]
fn test_complete_nonexistent_task() {
    let env = Env::default();
    let contract_id = env.register(TaskManager, ());

    let client = TaskManagerClient::new(&env, &contract_id);

    let owner = Address::generate(&env);

    env.mock_all_auths();

    // Try to complete a task that doesn't exist
    client.complete_task(&owner, &999);
}
```

**What we're testing:**
- Attempting to complete non-existent task
- Should panic (data integrity issue)
- Tests error path in `complete_task`

---

### Test 8: Empty Task Count
```rust
#[test]
fn test_empty_task_count() {
    let env = Env::default();
    let contract_id = env.register(TaskManager, ());

    let client = TaskManagerClient::new(&env, &contract_id);

    // Count should be 0 initially
    assert_eq!(client.get_task_count(), 0);
}
```

**Testing edge case:**
- No tasks created yet
- Count should be `0` (from `unwrap_or(0)`)
- Tests default value handling

---

### Test 9: Task Ownership (Comprehensive)

The most comprehensive test:
```rust
#[test]
fn test_task_ownership() {
    let env = Env::default();
    let contract_id = env.register(TaskManager, ());

    let client = TaskManagerClient::new(&env, &contract_id);

    let owner1 = Address::generate(&env);
    let owner2 = Address::generate(&env);

    env.mock_all_auths();

    let task1_id = client.create_task(&owner1, &String::from_str(&env, "Owner 1 task"));
    let task2_id = client.create_task(&owner2, &String::from_str(&env, "Owner 2 task"));

    let task1 = client.get_task(&task1_id);
    let task2 = client.get_task(&task2_id);

    assert_eq!(task1.owner, owner1);
    assert_eq!(task2.owner, owner2);

    // Owner 1 can complete their own task
    assert_eq!(client.complete_task(&owner1, &task1_id), true);

    // Owner 1 cannot complete owner 2's task
    assert_eq!(client.complete_task(&owner1, &task2_id), false);

    // Owner 2 can complete their own task
    assert_eq!(client.complete_task(&owner2, &task2_id), true);
}
```

**Complete workflow test:**

1. **Create two users and their tasks**
2. **Verify ownership is stored correctly**
3. **Test owner can complete own task**
4. **Test owner cannot complete other's task**
5. **Test other owner can complete their own task**

**This test covers:**
- Task creation by multiple users
- Ownership storage
- Ownership validation
- State transitions
- Cross-user interactions

---

### Test Summary

**Test Coverage:**

| Test | What It Validates |
|------|-------------------|
| `test_create_task` | Basic creation and data storage |
| `test_create_multiple_tasks` | Counter increments, multiple users |
| `test_complete_task` | State transition works |
| `test_complete_task_twice` | State validation prevents duplicates |
| `test_complete_task_unauthorized` | Ownership authorization |
| `test_get_nonexistent_task` | Error handling (panic) |
| `test_complete_nonexistent_task` | Error handling in complete |
| `test_empty_task_count` | Default value handling |
| `test_task_ownership` | Comprehensive multi-user workflow |

**Running tests:**
```bash
cargo test
```

**Expected output:**
```
running 9 tests
test test::test_create_task ... ok
test test::test_create_multiple_tasks ... ok
test test::test_complete_task ... ok
test test::test_complete_task_twice ... ok
test test::test_complete_task_unauthorized ... ok
test test::test_get_nonexistent_task ... ok
test test::test_complete_nonexistent_task ... ok
test test::test_empty_task_count ... ok
test test::test_task_ownership ... ok

test result: ok. 9 passed; 0 failed
```

---

## Common Pitfalls

### Pitfall 1: Forgetting Typed Storage Keys

**Incorrect (using strings):**
```rust
let count: u32 = env.storage().instance().get(&"COUNT").unwrap_or(0);
env.storage().instance().set(&count, &task);
```

**Problems:**
- `"COUNT"` vs `"count"` vs `"Count"` are all different
- Typos only caught at runtime
- Hard to refactor
- No IDE autocomplete

**Correct (using typed enum):**
```rust
let count: u32 = env.storage().instance().get(&StorageKey::TaskCount).unwrap_or(0);
env.storage().instance().set(&StorageKey::Task(count), &task);
```

**Benefits:**
- Compiler catches typos
- IDE autocomplete works
- Easy to refactor
- Self-documenting

**This is why we learned typed keys in Guest Book first!**

---

### Pitfall 2: Forgetting to Make Task Mutable

**Incorrect:**
```rust
pub fn complete_task(env: Env, owner: Address, task_id: u32) -> bool {
    owner.require_auth();
    
    let task: Task = env.storage().instance().get(&StorageKey::Task(task_id)).expect("Task not found");
    //  ^^^^ Missing 'mut'
    
    task.status = TaskStatus::Completed;  // Compiler error!
    env.storage().instance().set(&StorageKey::Task(task_id), &task);
    true
}
```

**The Error:**
```
error[E0594]: cannot assign to `task.status`, as `task` is not declared as mutable
```

**The Fix:**
```rust
let mut task: Task = env.storage().instance().get(&StorageKey::Task(task_id)).expect("Task not found");
//  ^^^ Add mut
```

---

### Pitfall 3: Forgetting to Save After Updating

**Incorrect:**
```rust
pub fn complete_task(env: Env, owner: Address, task_id: u32) -> bool {
    owner.require_auth();
    
    let mut task: Task = env.storage().instance().get(&StorageKey::Task(task_id)).expect("Task not found");
    
    if task.owner != owner { return false; }
    if task.status == TaskStatus::Completed { return false; }
    
    task.status = TaskStatus::Completed;
    // Missing: env.storage().instance().set(&StorageKey::Task(task_id), &task);
    
    log!(&env, "Task {} marked as completed", task_id);
    true
}
```

**The Problem:**
- Status updated in memory
- Never written to storage
- Next load still shows `Pending`!

**The Fix:**
```rust
task.status = TaskStatus::Completed;
env.storage().instance().set(&StorageKey::Task(task_id), &task);  // Don't forget!
```

---

### Pitfall 4: Wrong Ownership Check

**Incorrect:**
```rust
// Bad - checking authentication but not ownership:
pub fn complete_task(env: Env, owner: Address, task_id: u32) -> bool {
    owner.require_auth();  // Only checks authentication!
    
    let mut task = get_task(task_id);
    task.status = TaskStatus::Completed;
    save_task(task);
    true
}
```

**The Problem:**
- Anyone can complete anyone's tasks!
- Security vulnerability

**The Fix:**
```rust
pub fn complete_task(env: Env, owner: Address, task_id: u32) -> bool {
    owner.require_auth();
    
    let mut task = get_task(task_id);
    
    // Add ownership check:
    if task.owner != owner {
        return false;
    }
    
    task.status = TaskStatus::Completed;
    save_task(task);
    true
}
```

---

### Pitfall 5: Not Validating State Before Transition

**Incorrect:**
```rust
// Bad - no validation:
pub fn complete_task(env: Env, owner: Address, task_id: u32) -> bool {
    owner.require_auth();
    
    let mut task = get_task(task_id);
    
    if task.owner != owner { return false; }
    
    // Missing: check if already completed!
    task.status = TaskStatus::Completed;
    save_task(task);
    true
}
```

**The Problem:**
- Can "complete" an already-completed task
- Wastes gas
- Violates state machine

**The Fix:**
```rust
if task.status == TaskStatus::Completed {
    log!(&env, "Task already completed");
    return false;
}
```

---

### Pitfall 6: Inconsistent Storage Keys

**Incorrect:**
```rust
// Creating with one key type:
env.storage().instance().set(&StorageKey::Task(count), &task);

// Reading with different key:
env.storage().instance().get(&count)  // Wrong! Not using StorageKey
```

**The Problem:**
- Different keys access different storage
- Data appears missing

**The Fix:**
```rust
// Always use the same typed key:
env.storage().instance().set(&StorageKey::Task(count), &task);
env.storage().instance().get(&StorageKey::Task(count))
```

---

## Practice Exercises

### Exercise 1: Add InProgress Status (Intermediate)

**Goal:** Add a third state to the task lifecycle

**Requirements:**
1. Add `InProgress` variant to `TaskStatus` enum
2. Add `start_task` function to transition Pending → InProgress
3. Modify `complete_task` to require InProgress status
4. Write tests for the new state transitions

**State Machine:**
```
Pending ──start──> InProgress ──complete──> Completed
```

**Starter Code:**
```rust
#[contracttype]
#[derive(Clone, PartialEq, Debug)]
pub enum TaskStatus {
    Pending,
    InProgress,  // Add this
    Completed,
}

// Add this function:
pub fn start_task(env: Env, owner: Address, task_id: u32) -> bool {
    owner.require_auth();
    
    let mut task: Task = env
        .storage()
        .instance()
        .get(&StorageKey::Task(task_id))
        .expect("Task not found");
    
    // TODO: Implement
    // 1. Verify owner
    // 2. Check task is Pending
    // 3. Change to InProgress
    // 4. Save and return true
}

// Modify complete_task:
pub fn complete_task(env: Env, owner: Address, task_id: u32) -> bool {
    // TODO: Add check
    // Only allow completing if status is InProgress
}
```

---

### Exercise 2: Add Task Deletion (Intermediate)

**Goal:** Allow users to delete their own tasks

**Requirements:**
1. Create `delete_task` function
2. Only owner can delete
3. Can delete tasks in any status
4. Use typed storage keys
5. Write comprehensive tests

**Hint for deletion:**
```rust
// To remove from storage:
env.storage().instance().remove(&StorageKey::Task(task_id));
```

**Function Signature:**
```rust
pub fn delete_task(env: Env, owner: Address, task_id: u32) -> bool {
    // TODO: Implement
}
```

---

### Exercise 3: Add Task Description Update (Advanced)

**Goal:** Allow updating task description

**Requirements:**
1. Create `update_description` function
2. Only owner can update
3. Can only update Pending tasks
4. Validate description is not empty
5. Use typed storage keys throughout

**Function Signature:**
```rust
pub fn update_description(env: Env, owner: Address, task_id: u32, new_description: String) -> bool {
    // TODO: Implement
}
```

---

### Exercise 4: Get Tasks by Owner (Advanced)

**Goal:** Retrieve all tasks for a specific owner

**Requirements:**
1. Create `get_tasks_by_owner` function
2. Return a vector of task IDs
3. Filter by owner address
4. Consider efficiency (iterating through all tasks)

**Note:** This requires iterating through storage, which can be expensive!

**Function Signature:**
```rust
use soroban_sdk::Vec;

pub fn get_tasks_by_owner(env: Env, owner: Address) -> Vec<u32> {
    // TODO: Implement
    // Hint: Loop from 1 to task_count
    // Check each task's owner
    // Add matching IDs to vector
}
```

---

### Exercise 5: Add Task Priority (Advanced)

**Goal:** Add priority levels to tasks

**Requirements:**
1. Create `TaskPriority` enum (Low, Medium, High)
2. Add priority field to Task struct
3. Create `set_priority` function
4. Maintain typed storage throughout
5. Write comprehensive tests

**Starter Code:**
```rust
#[contracttype]
#[derive(Clone, PartialEq, Debug)]
pub enum TaskPriority {
    Low,
    Medium,
    High,
}

#[contracttype]
#[derive(Clone, Debug)]
pub struct Task {
    pub id: u32,
    pub owner: Address,
    pub description: String,
    pub status: TaskStatus,
    pub priority: TaskPriority,  // Add this
}
```

---

## Assessment

### Knowledge Check Questions

**Question 1:** Why do we use typed enums like `StorageKey` instead of raw strings for storage keys?

<details>
<summary>Click to see answer</summary>

**Benefits of Typed Enums:**

1. **Compile-Time Safety:**
```rust
   // Typed enum - compiler catches errors:
   StorageKey::TasCount  // ✗ Compiler error: no variant 'TasCount'
   
   // Raw strings - runtime errors:
   "TASK_COUNT"  // ✓ Compiles but might be wrong key
   "TaskCount"   // ✓ Different string, different key!
```

2. **IDE Support:**
   - Autocomplete suggests correct variants
   - Can't accidentally type wrong key
   - Easier to discover available keys

3. **Refactoring:**
   - Rename once in enum definition
   - All uses update automatically
   - Compiler finds all references

4. **Self-Documentation:**
```rust
   // Clear what keys exist:
   pub enum StorageKey {
       TaskCount,    // Obvious: stores count
       Task(u32),    // Obvious: stores tasks by ID
   }
   
   // vs strings scattered throughout code
```

5. **Pattern Matching:**
```rust
   match key {
       StorageKey::TaskCount => // Handle count
       StorageKey::Task(id) => // Handle task
   }
```

**This is why Guest Book taught this pattern first - it's fundamental to good Soroban contracts!**
</details>

---

**Question 2:** Explain the two levels of authorization in `complete_task`. Why do we need both?

<details>
<summary>Click to see answer</summary>

**Two Levels:**

**Level 1: Authentication**
```rust
owner.require_auth();
```
- **What:** Verifies the caller signed the transaction
- **Purpose:** Proves you are who you claim to be
- **Prevents:** Impersonation (pretending to be someone else)

**Level 2: Ownership**
```rust
if task.owner != owner {
    return false;
}
```
- **What:** Verifies the caller owns this specific resource
- **Purpose:** Proves you have permission to modify this task
- **Prevents:** Unauthorized modifications (changing others' tasks)

**Why Both Are Needed:**

Authentication alone isn't enough:
```rust
// With only authentication:
alice.require_auth();  // ✓ Proves caller is Alice
// But Alice could complete Bob's tasks!
```

Ownership check alone isn't enough:
```rust
// Without authentication:
if task.owner == owner {  // ✓ Caller claims to be owner
    // But did they actually sign the transaction?
    // Or did someone spoof the owner parameter?
}
```

**Together:**
1. Authentication: "Are you really Alice?" ✓
2. Ownership: "Does this task belong to Alice?" ✓
3. Both must pass for authorization ✓

**Real-world analogy:**
- Authentication = Showing ID to prove who you are
- Ownership = Showing deed to prove you own the house
- You need both to modify your property
</details>

---

**Question 3:** Why does `complete_task` return `bool` instead of panicking? When should you use each approach?

<details>
<summary>Click to see answer</summary>

**Why return bool:**

Some failures are **expected** and **normal**:
- Wrong owner trying to complete task
- Task already completed
- These are legitimate scenarios, not errors

Returning `bool` allows:
- Graceful handling
- Caller can check and respond
- Transaction still succeeds (just returns false)

**When to panic:**

Use `panic!` / `expect()` for **unexpected errors**:
```rust
// Task doesn't exist - unexpected, data integrity issue:
let task = env.storage()
    .instance()
    .get(&StorageKey::Task(task_id))
    .expect("Task not found");
```

**When to return bool:**

Use `bool` for **expected failures**:
```rust
// Wrong owner - expected scenario:
if task.owner != owner {
    return false;
}

// Already completed - expected scenario:
if task.status == TaskStatus::Completed {
    return false;
}
```

**Decision Table:**

| Scenario | Action | Reason |
|----------|--------|--------|
| Task doesn't exist | Panic | Data integrity issue |
| Wrong owner | Return false | Expected authorization failure |
| Already completed | Return false | Expected state validation failure |
| Invalid task ID | Panic | Caller error |
| Storage corruption | Panic | Critical system error |

**Benefits of bool returns:**
- Caller can handle gracefully
- Can retry or show user-friendly message
- Transaction succeeds, operation fails
- Better UX (can show why it failed)
</details>

---

**Question 4:** Compare the storage patterns between Guest Book and Task Manager. What's similar and what's different?

<details>
<summary>Click to see answer</summary>

**Similarities:**

1. **Typed Storage Keys:**
```rust
   // Guest Book:
   pub enum DataKey {
       Count,
       Message(u32),
   }
   
   // Task Manager:
   pub enum StorageKey {
       TaskCount,
       Task(u32),
   }
```

2. **Counter Pattern:**
```rust
   // Both use a counter for IDs:
   let mut count = env.storage().instance()
       .get(&key)
       .unwrap_or(0);
   count += 1;
```

3. **Sequential IDs:**
   - First item gets ID 1
   - Second gets ID 2
   - Counter tracks total and next ID

4. **Storage Structure:**
```rust
   // Both store:
   // - Count/TaskCount → total number
   // - Message(id)/Task(id) → individual items
```

**Differences:**

1. **Mutability:**
```rust
   // Guest Book - append only:
   post_message()  // Creates new, never modifies
   
   // Task Manager - can update:
   complete_task()  // Modifies existing task
```

2. **Data Complexity:**
```rust
   // Guest Book Message:
   pub struct Message {
       author: Address,
       text: String,
   }
   
   // Task Manager - adds state:
   pub struct Task {
       id: u32,
       owner: Address,
       description: String,
       status: TaskStatus,  // State tracking!
   }
```

3. **Authorization:**
```rust
   // Guest Book - simple:
   author.require_auth();  // Only authentication
   
   // Task Manager - complex:
   owner.require_auth();           // Authentication
   if task.owner != owner { ... }  // + Ownership
```

4. **Return Values:**
```rust
   // Guest Book - void:
   pub fn post_message(...)  // No return
   
   // Task Manager - returns values:
   pub fn create_task(...) -> u32  // Returns ID
   pub fn complete_task(...) -> bool  // Returns success
```

**Key Insight:**

Task Manager **builds on** Guest Book's patterns:
- Same typed storage key approach
- Same counter/ID system
- Adds state management
- Adds update operations
- Adds complex authorization

This is intentional - learn fundamentals first, then add complexity!
</details>

---

**Question 5:** Walk through what happens in storage when a task is created, completed, and attempted to complete again.

<details>
<summary>Click to see answer</summary>

**Initial State:**
```
Storage: {}
```

---

**Step 1: Create Task**

Call: `create_task(alice, "Learn Soroban")`

**Process:**
1. `require_auth()` verifies Alice signed ✓
2. Read count: `get(&StorageKey::TaskCount)` → None, use 0
3. Increment: `count = 0 + 1 = 1`
4. Create task:
```rust
   Task {
       id: 1,
       owner: alice,
       description: "Learn Soroban",
       status: TaskStatus::Pending
   }
```
5. Store task: `set(&StorageKey::Task(1), &task)`
6. Store count: `set(&StorageKey::TaskCount, &1)`
7. Return `1`

**Storage After Create:**
```
Storage: {
    StorageKey::TaskCount → 1,
    StorageKey::Task(1) → Task {
        id: 1,
        owner: alice,
        description: "Learn Soroban",
        status: Pending
    }
}
```

---

**Step 2: Complete Task**

Call: `complete_task(alice, 1)`

**Process:**
1. `require_auth()` verifies Alice signed ✓
2. Load task: `get(&StorageKey::Task(1))` →
```rust
   Task {
       id: 1,
       owner: alice,
       description: "Learn Soroban",
       status: Pending
   }
```
3. Check ownership: `task.owner (alice) == owner (alice)` ✓
4. Check status: `task.status (Pending) != Completed` ✓
5. Update: `task.status = TaskStatus::Completed`
6. Save: `set(&StorageKey::Task(1), &updated_task)`
7. Return `true`

**Storage After Complete:**
```
Storage: {
    StorageKey::TaskCount → 1,
    StorageKey::Task(1) → Task {
        id: 1,
        owner: alice,
        description: "Learn Soroban",
        status: Completed  // Changed!
    }
}
```

---

**Step 3: Try to Complete Again**

Call: `complete_task(alice, 1)`

**Process:**
1. `require_auth()` verifies Alice signed ✓
2. Load task: Task with `status: Completed`
3. Check ownership: `task.owner == owner` ✓
4. Check status: `task.status == Completed` ✗
5. **Validation fails!**
6. Log: "Task already completed"
7. Return `false`
8. **No storage changes**

**Storage After Failed Attempt:**
```
Storage: {
    StorageKey::TaskCount → 1,
    StorageKey::Task(1) → Task {
        id: 1,
        owner: alice,
        description: "Learn Soroban",
        status: Completed  // Unchanged
    }
}
```

**Summary:**
```
Creation:    {} → Pending state added
Completion:  Pending → Completed
Retry:       Completed → Completed (rejected, no change)
```

**Key Points:**
- Typed keys used throughout
- State transitions are validated
- Failed operations don't modify storage
- Each step uses `StorageKey` enum
</details>

---

## Summary and Next Steps

### What You've Learned

Congratulations! You've completed the Task Manager tutorial. Here's what you now understand:

**Core Concepts:**
- Using enums to represent state (TaskStatus)
- Implementing state machines with transition rules
- Complex authorization with ownership validation
- Updating existing data in storage
- Returning success/failure indicators
- **Applying typed storage keys (from Guest Book)**

**Advanced Patterns:**
- Two-level authorization (authentication + ownership)
- State validation before transitions
- Mutable data updates
- Expected vs unexpected errors
- Consistent use of typed storage keys

**Testing Skills:**
- Testing state transitions
- Testing authorization failures
- Testing edge cases (double completion)
- Using `assert!` for boolean returns
- Comprehensive multi-user tests

---

### Key Takeaways

1. **Build on Fundamentals**
   - Guest Book taught typed storage keys
   - Task Manager applies the same pattern
   - Consistency across contracts matters

2. **State Machines Are Powerful**
   - Enums represent distinct states
   - Validation enforces transition rules
   - Type safety prevents invalid states

3. **Authorization Has Layers**
   - Authentication: Who are you?
   - Ownership: Can you modify this?
   - Both are necessary for security

4. **Type Safety Everywhere**
   - Typed storage keys prevent errors
   - Enums for state are type-safe
   - Compiler catches mistakes early

---

### Evolution of Concepts

| Concept | Guest Book | Task Manager |
|---------|-----------|--------------|
| Storage Keys | `DataKey` enum ✓ | `StorageKey` enum ✓ |
| State | No state tracking | `TaskStatus` enum |
| Authorization | Simple (is author) | Complex (is owner + validation) |
| Data Updates | Append only | Modify existing |
| Error Handling | Panic on error | Return bool for expected failures |
| Return Values | Void functions | Returns ID and bool |

---

### What's Next

**Recommended Next Step: Advanced Contracts**

You're now ready for contracts that combine everything:

**Escrow Contract:**
- Multiple parties (Buyer, Seller, Arbiter)
- Token transfers
- Complex state machines (multiple paths)
- Time locks
- Dispute resolution

**Other Options:**

**Voting/Governance Contract:**
- Proposal states
- Weighted voting
- Time constraints
- Result calculation

**Marketplace Contract:**
- Item listings
- Purchase mechanism
- Ownership transfer
- Complex authorization

---

### Homework Assignment

#### Part 1: Practice Exercises 

Implement **at least two** exercises:

1. **Add InProgress Status** (Exercise 1) 
2. **Implement Task Deletion** (Exercise 2) 
3. **Bonus:** Add Description Update (Exercise 3)

**Requirements:**
- Use typed storage keys throughout
- Write comprehensive tests
- Follow patterns from the tutorial

---

#### Part 2: Written Reflection (Required)

Answer these questions (3-4 sentences each):

1. **Typed Storage Keys:**
   - How are typed storage keys used in both Guest Book and Task Manager?
   - What problems do they solve?
   - Why is this pattern important?

2. **State Machines:**
   - Explain what a state machine is
   - How do enums help implement state machines?
   - Give a real-world example

3. **Authorization Patterns:**
   - Explain authentication vs authorization
   - Why do we need both in `complete_task`?
   - What could go wrong with only one?

4. **Progression:**
   - How does Task Manager build on Guest Book?
   - What new concepts did you learn?
   - Which pattern was most valuable?

---

#### Part 3: Design Challenge (Optional)

**Simplified Auction Contract:**

**Requirements:**
- Use typed storage keys (like `StorageKey` enum)
- Auction states: Open, Closed
- Users can place bids
- Track highest bid and bidder
- Only creator can close auction

**Deliverables:**
1. Storage key enum definition
2. Data structures (enums, structs)
3. State transition rules
4. Function signatures
5. Authorization strategy
6. Test plan

---

### Final Thoughts

You've made excellent progress! You now understand:

- How typed storage keys provide safety and consistency
- How to model processes as state machines
- How to implement layered authorization
- How to safely update on-chain data
- How to handle expected and unexpected failures

**The key insight:** Patterns learned in Guest Book (typed storage keys) apply everywhere in Soroban development. This consistency makes building complex contracts easier.

**Keep building!** Try extending Task Manager with your own features. The patterns you've learned are fundamental to all smart contract development.





**End of Tutorial**
