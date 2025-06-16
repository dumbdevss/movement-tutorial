# Document Management Contract Tutorial - Implementation Guide

This guide walks through each TODO item in the secure document management system with clear explanations and implementation details. Follow these steps to complete your Move module implementation.

## Source Code Reference

The complete implementation of this smart contract can be found in the repository:

**Repository:** [dumbdevss/document-management](https://github.com/dumbdevss/document-management)  
**Branch:** `smart-contract`

You can view or clone the finished code to compare against your implementation as you follow along with this guide.

## Understanding Key Concepts Before We Start

### What is a Resource Account?

A **resource account** is a special type of account in Aptos that's controlled programmatically rather than by a private key. Think of it as a "smart contract wallet" that your module can control. We use resource accounts because:

- They provide a deterministic address for storing global state
- They can hold and manage coins (like collecting fees)
- They're perfect for modules that need to act as intermediaries
- The original deployer maintains control through a `SignerCapability`

### Why Do We Need Events?

**Events** are crucial for dApp functionality because:

- Blockchains don't have built-in notification systems
- Frontend applications need to know when things happen on-chain
- Events provide a searchable, chronological log of activities
- They enable features like activity feeds, notifications, and analytics
- Off-chain indexers use events to build databases for faster queries

### The Dual Storage Strategy Explained

We use both a `Table` and a `vector` for storing documents because each serves different purposes:

- **Table**: Provides O(1) lookup by document ID - perfect for "get document by ID" operations
- **Vector**: Enables iteration and filtering - essential for "get all documents for user" queries
- This hybrid approach optimizes for both individual document access and bulk operations

## TODO 1: Define Constants

```move
// Constants
const SEED: vector<u8> = b"secure_docs";
const UPLOAD_FEE: u64 = 5000000; // 0.05 APT (in microAPT, 1 APT = 10^8 microAPT)
```

**Why These Constants Matter:**

The `SEED` constant ensures our resource account has a **deterministic address**. This means every deployment of this module will create a resource account at the same address, making it predictable for frontend integration.

The `UPLOAD_FEE` serves multiple purposes:

- **Spam prevention**: Small fees discourage frivolous document uploads
- **Resource sustainability**: Helps cover storage costs on the blockchain
- **Value alignment**: Users are more likely to use the system responsibly when there's a small cost

**Implementation steps:**

1. Define `SEED` as a byte vector to create a deterministic resource account
2. Set `UPLOAD_FEE` in microAPT (0.05 APT = 5,000,000 microAPT)
3. These constants will be used throughout the module for consistency

## TODO 2: Create Document Struct

```move
struct Document has store, drop, copy {
    id: String,              // Unique document ID
    name: String,            // Document name
    ipfs_hash: String,       // IPFS hash for off-chain storage
    created_at: u64,         // Timestamp of creation
    owner: address,          // Document owner
    signatures: vector<address>, // List of signers
    allowed_signers: vector<address>, // Users allowed to sign
}
```

**Understanding Move Abilities:**

- `store`: Allows the struct to be stored inside other structs and global storage
- `drop`: Allows the struct to be discarded/destroyed when no longer needed
- `copy`: Allows the struct to be copied (useful for returning data from functions)

**Why IPFS for Storage?**
We store only the IPFS hash on-chain because:

- Blockchains are expensive for large data storage
- IPFS provides decentralized, content-addressed storage
- The hash acts as both a unique identifier and integrity check
- Users can retrieve files from IPFS using just the hash

**Implementation steps:**

1. Define a struct with `store`, `drop`, and `copy` abilities
2. Include fields for document metadata (ID, name, IPFS hash)
3. Add timestamp field for audit purposes (crucial for legal documents)
4. Include ownership tracking with `owner` field
5. Add vectors to track both actual signatures and authorized signers (separation of permission and action)

## TODO 3: Create State Struct

```move
struct DocState has key {
    admin: address,
    documents: Table<String, Document>, // Maps document ID to Document
    doc_counter: u64,          // Tracks total documents
    document_vec: vector<Document>, // Vector of documents for iteration
    upload_events: EventHandle<UploadEvent>,
    sign_events: EventHandle<SignEvent>,
    share_events: EventHandle<ShareEvent>,
}
```

**Why We Need Global State:**
The `key` ability makes this a **global resource** - there's exactly one instance per account. This serves as our module's "database" and ensures consistency across all operations.

**The Counter Pattern:**
`doc_counter` provides valuable analytics and can be used for:

- Generating sequential document IDs if needed
- Rate limiting if needed

**Event Handles Explained:**
Each `EventHandle` is like a "channel" for a specific type of event. When something happens (upload, sign, share), we emit an event that external systems can listen to. This enables:

- Real-time notifications in the frontend
- Activity logging
- Analytics and reporting
- Integration with other systems

**Implementation steps:**

1. Create a struct with `key` ability for global storage
2. Store admin address for privileged operations
3. Use both a Table (for O(1) lookups) and a vector (for iteration)
4. Include a counter to track the number of documents
5. Add EventHandles for each event type to enable off-chain tracking

## TODO 4: Create Event Structs

```move
#[event]
struct UploadEvent has drop, store {
    id: String,
    name: String,
    ipfs_hash: String,
    owner: address,
    created_at: u64,
}

#[event]
struct SignEvent has drop, store {
    id: String,
    signer: address,
}

#[event]
struct ShareEvent has drop, store {
    id: String,
    allowed_signer: address,
}
```

**Event Design Philosophy:**
Each event contains just enough information for external systems to understand what happened and potentially take action. We avoid including too much data to keep events lightweight and focused.

**The `#[event]` Annotation:**
This tells the Move compiler that these structs are meant to be emitted as events, which enables special handling and indexing by the blockchain infrastructure.

**Implementation steps:**

1. Create structs for each event type with `#[event]` annotation
2. Include `drop` and `store` abilities for all event structs
3. Each event contains relevant data for off-chain tracking
4. Keep events compact with only necessary information

## TODO 5: Initialize Module

```move
fun init_module(admin: &signer) {
    let admin_addr = signer::address_of(admin);

    // Create resource account
    let (resource_signer, signer_cap) = account::create_resource_account(admin, SEED);
    let resource_addr = signer::address_of(&resource_signer);

    // Register resource account for APT
    if (!coin::is_account_registered<AptosCoin>(resource_addr)) {
        coin::register<AptosCoin>(&resource_signer);
    };

    // Initialize DocState under resource account
    move_to(&resource_signer, DocState {
        admin: admin_addr,
        documents: table::new(),
        doc_counter: 0,
        document_vec: vector::empty(),
        upload_events: account::new_event_handle<UploadEvent>(&resource_signer),
        sign_events: account::new_event_handle<SignEvent>(&resource_signer),
        share_events: account::new_event_handle<ShareEvent>(&resource_signer),
    });

    // Store SignerCapability under admin's address
    move_to(admin, ResourceAccountCap {
        signer_cap,
    });
}
```

**The Module Initialization Pattern:**
`init_module` runs exactly once when the module is first published. It's perfect for setting up initial state and creating necessary accounts.

**Understanding SignerCapability:**
The `SignerCapability` is like a "master key" that allows the admin to:

- Control the resource account programmatically
- Make transactions on behalf of the resource account
- Manage funds and state within the resource account

We store it under the admin's address so only they can use it for administrative functions.

**Why Register for APT?**
Before an account can receive APT tokens, it must be registered. This is a one-time setup that creates the necessary data structures for token storage.

**Implementation steps:**

1. Get the admin's address for authorization
2. Create a resource account using the `SEED` constant
3. Register the resource account for Aptos Coin operations
4. Initialize the `DocState` with empty collections and event handles
5. Store the `SignerCapability` for the resource account under the admin's address

## TODO 6-8: Helper Functions

```move
fun get_resource_address(): address acquires ResourceAccountCap {
    let cap = borrow_global<ResourceAccountCap>(@document_management);
    account::get_signer_capability_address(&cap.signer_cap)
}

fun assert_document_exists(state: &DocState, id: &String) {
    assert!(table::contains(&state.documents, *id), error::not_found(E_DOCUMENT_NOT_FOUND));
}

fun assert_is_owner(document: &Document, account: &signer) {
    assert!(document.owner == signer::address_of(account), error::permission_denied(E_NOT_AUTHORIZED));
}
```

**The Helper Function Strategy:**
These utility functions implement common patterns we'll use repeatedly:

- **DRY Principle**: Instead of repeating address lookup logic, we centralize it
- **Clear Error Messages**: Specific assertions provide better debugging information
- **Security**: Authorization checks prevent unauthorized actions

**Why Separate Assertions?**
Breaking down validation into specific functions makes code:

- More readable and maintainable
- Easier to test individually
- Reusable across different functions
- Better for error reporting

**Implementation steps:**

1. Create a helper to get the resource account address
2. Implement document existence validation
3. Create an ownership assertion helper

## TODO 9-10: Document Upload Function Logic

```move
// Handle payment
assert_user_has_enough_apt(user_addr, UPLOAD_FEE);
let payment = coin::withdraw<AptosCoin>(user, UPLOAD_FEE);
coin::deposit<AptosCoin>(get_resource_address(), payment);

// Create document
let document = Document {
    id,
    name,
    ipfs_hash,
    created_at: timestamp::now_seconds(),
    owner: user_addr,
    signatures: vector::empty(),
    allowed_signers: vector::singleton(user_addr), // Owner can sign by default
};

table::add(&mut state.documents, id, document);
vector::push_back(&mut state.document_vec, document);
state.doc_counter = state.doc_counter + 1;

// Emit upload event
event::emit_event(&mut state.upload_events, UploadEvent {
    id,
    name,
    ipfs_hash,
    owner: user_addr,
    created_at: timestamp::now_seconds(),
});
```

**The Payment Flow:**

1. **Pre-flight check**: Verify user has enough APT before starting
2. **Atomic withdrawal**: Take the exact fee amount from user's account
3. **Secure deposit**: Transfer to our resource account immediately

This pattern ensures either the entire operation succeeds or fails - no partial states.

**Default Permissions Design:**
We automatically add the document owner to `allowed_signers` because:

- Owners should always be able to sign their own documents
- It simplifies the initial workflow
- Users can remove themselves later if needed

**Maintaining Data Consistency:**
Notice we update both storage structures (`table` and `vector`) in the same transaction. This ensures they never get out of sync.

**Implementation steps:**

1. Verify the user has sufficient APT balance
2. Withdraw the upload fee and deposit to resource account
3. Create a new Document struct with appropriate fields
4. Store the document in both table and vector
5. Increment document counter
6. Emit upload event for off-chain tracking

## TODO 11: Implement Add Signer Function

```move
public entry fun add_signer(
    owner: &signer,
    id: String,
    new_signer: address
) acquires DocState, ResourceAccountCap {
    let state = borrow_global_mut<DocState>(get_resource_address());
    assert_document_exists(state, &id);

    // Borrow the document from the table
    let document_table = table::borrow_mut(&mut state.documents, id);

    // Find the document in the vector
    let (found, i) = vector::index_of(&state.document_vec, document_table);
    assert!(found, error::not_found(E_DOCUMENT_NOT_FOUND));

    // Borrow the document from the vector
    let document_vec = vector::borrow_mut(&mut state.document_vec, i);

    // Assert the owner is authorized
    assert_is_owner(document_table, owner);

    // Add the new signer if not already present
    if (!vector::contains(&document_table.allowed_signers, &new_signer)) {
        // Update both table and vector
        vector::push_back(&mut document_table.allowed_signers, new_signer);
        vector::push_back(&mut document_vec.allowed_signers, new_signer);

        // Emit the share event
        event::emit_event(&mut state.share_events, ShareEvent {
            id: id,
            allowed_signer: new_signer,
        });
    }
}
```

**The Synchronization Challenge:**
Managing both Table and vector storage requires careful coordination. We must:

1. Find the same document in both structures
2. Update both simultaneously
3. Ensure they remain consistent

**Idempotent Operations:**
We check `if (!vector::contains(...))` to make this function **idempotent** - calling it multiple times with the same parameters has the same effect as calling it once. This prevents:

- Duplicate entries
- Confusing event emissions

**Access Control Pattern:**
The function is `public entry` (callable from transactions) but includes authorization checks. This is the standard pattern for user-facing functions that modify state.

**Implementation steps:**

1. Access the global state
2. Verify document exists and caller is owner
3. Update both table and vector entries for consistency
4. Only add signer if not already present (idempotent behavior)
5. Emit event for off-chain tracking

## TODO 12: Implement Sign Document Function

```move
public entry fun sign_document(
    signer_acc: &signer,
    id: String
) acquires DocState, ResourceAccountCap {
    let signer_addr = signer::address_of(signer_acc);
    let state = borrow_global_mut<DocState>(get_resource_address());
    assert_document_exists(state, &id);

    // Borrow the document from the table
    let document_table = table::borrow_mut(&mut state.documents, id);

    // Find the document in the vector
    let (found, i) = vector::index_of(&state.document_vec, document_table);
    assert!(found, error::not_found(E_DOCUMENT_NOT_FOUND));

    // Borrow the document from the vector
    let document_vec = vector::borrow_mut(&mut state.document_vec, i);

    // Assert the signer is allowed and hasn't already signed
    assert_allowed_to_sign(document_table, signer_addr);
    assert_not_already_signed(document_table, signer_addr);

    // Update signatures in both structures
    vector::push_back(&mut document_table.signatures, signer_addr);
    vector::push_back(&mut document_vec.signatures, signer_addr);

    // Emit sign event
    event::emit_event(&mut state.sign_events, SignEvent {
        id,
        signer: signer_addr,
    });
}
```

**Preventing Double-Signing:**
We check both authorization and previous signature status because:

- **Authorization**: Not everyone should be able to sign every document
- **Anti-replay**: Each person should only sign once per document
- **Data integrity**: Prevents signatures array from getting polluted

**Why Events Matter Here:**
Signing events are particularly important because they often trigger:

- Communication with external systems like email notifications to other parties
- Legal timestamps for compliance
- User interface updates

**Implementation steps:**

1. Get signer address and access global state
2. Verify document exists
3. Verify signer is authorized and hasn't signed already
4. Update both table and vector entries
5. Emit signature event

## TODO 13: Implement Remove Signer Function

```move
public entry fun remove_signer(
    owner: &signer,
    id: String,
    remove_signer: address
) acquires DocState, ResourceAccountCap {
    let state = borrow_global_mut<DocState>(get_resource_address());
    assert_document_exists(state, &id);

    // Borrow the document from the table
    let document_table = table::borrow_mut(&mut state.documents, id);

    // Find the document in the vector
    let (found, i) = vector::index_of(&state.document_vec, document_table);
    assert!(found, error::not_found(E_DOCUMENT_NOT_FOUND));

    // Borrow the document from the vector
    let document_vec = vector::borrow_mut(&mut state.document_vec, i);

    // Assert the owner is authorized
    assert_is_owner(document_table, owner);

    // Remove the signer if present
    let (found, index) = vector::index_of(&document_table.allowed_signers, &remove_signer);
    if (found) {
        vector::remove(&mut document_table.allowed_signers, index);
        vector::remove(&mut document_vec.allowed_signers, index);
    } else {
        abort error::not_found(E_SIGNER_NOT_FOUND);
    };
}
```

**Why Allow Signer Removal?**
Document signing workflows often change:

- Someone leaves the organization
- Responsibilities shift
- Initial setup included wrong people
- Privacy requirements change

**The Security Consideration:**
Removing a signer doesn't invalidate their existing signature - this is intentional. Once someone has signed, that action is permanent and auditable. We're only removing future signing permission.

**Error Handling Strategy:**
We explicitly abort with `E_SIGNER_NOT_FOUND` rather than silently succeeding. This gives clear feedback when someone tries to remove a signer who wasn't authorized in the first place.

**Implementation steps:**

1. Access global state and verify document exists
2. Verify caller is document owner
3. Find signer in the allowed signers list
4. Remove signer from both table and vector entries
5. Abort if signer not found

## TODO 14-16: Implement View Functions

### Understanding View Functions

View functions are **read-only** operations that don't modify blockchain state. They're perfect for:

- Frontend data fetching
- Analytics and reporting
- Validation checks
- Public information access

The `#[view]` annotation optimizes these functions for querying.

### TODO 14: Implement `get_document` View Function

```move
#[view]
public fun get_document(id: String): (String, String, u64, address, vector<address>, vector<address>) acquires DocState, ResourceAccountCap {
    let state = borrow_global<DocState>(get_resource_address());
    assert_document_exists(state, &id);

    let doc = table::borrow(&state.documents, id);
    (doc.name, doc.ipfs_hash, doc.created_at, doc.owner, doc.signatures, doc.allowed_signers)
}
```

**Return Tuple Strategy:**
We return a tuple instead of the Document struct because:

- Tuples are more efficient for RPC calls
- Frontend code can destructure easily
- Provides only the necessary public information

**Implementation steps:**

1. Add `#[view]` annotation to mark this as a read-only function
2. Acquire global state using `borrow_global` (not `borrow_global_mut`)
3. Verify document exists with the helper function
4. Return document data as a tuple of values
5. Include all relevant document fields (name, IPFS hash, timestamp, owner, signatures, allowed signers)

### TODO 15: Implement `get_documents_for_signer` View Function

```move
#[view]
public fun get_documents_for_signer(
    user_addr: address
): vector<Document> acquires DocState, ResourceAccountCap {
    let state = borrow_global<DocState>(get_resource_address());
    let result = vector::empty<Document>();
    let len = vector::length(&state.document_vec);
    let i = 0;

    while (i < len) {
        let doc = vector::borrow(&state.document_vec, i);
        if (vector::contains(&doc.allowed_signers, &user_addr)) {
            vector::push_back(&mut result, *doc);
        };
        i = i + 1;
    };

    result
}
```

**Implementation steps:**

1. Add `#[view]` annotation for read-only function
2. Create an empty result vector to store matching documents
3. Iterate through all documents in the `document_vec`
4. For each document, check if the user's address is in the `allowed_signers` list
5. If so, add the document to the result vector
6. Return the filtered list of documents

### TODO 16: Implement `get_documents_by_signer` View Function

```move
#[view]
public fun get_documents_by_signer(
    user_addr: address
): vector<Document> acquires DocState, ResourceAccountCap {
    let state = borrow_global<DocState>(get_resource_address());
    let result = vector::empty<Document>();
    let len = vector::length(&state.document_vec);
    let i = 0;

    while (i < len) {
        let doc = vector::borrow(&state.document_vec, i);
        if (doc.owner == user_addr) {
            vector::push_back(&mut result, *doc);
        };
        i = i + 1;
    };

    result
}
```

**Ownership vs. Signing Rights:**
This function returns documents **owned** by the user, which is different from documents they're **allowed to sign**. This distinction enables different UI views:

- "My Documents" (documents I created/own)
- "Documents to Sign" (documents others created but I can sign)

**Implementation steps:**

1. Add `#[view]` annotation for read-only function
2. Create an empty result vector to store matching documents
3. Iterate through all documents in the `document_vec`
4. For each document, check if the user's address matches the document's `owner` field
5. If so, add the document to the result vector
6. Return the filtered list of documents

## Architectural Design Decisions Explained

### 1. **The Resource Account Pattern**

We use a resource account instead of storing state under the module deployer's account because:

- **Separation of concerns**: Module logic is separate from admin identity
- **Upgradeability**: Admin can change without affecting the module
- **Fee collection**: Resource account can hold and manage collected fees
- **Predictability**: Same address across different deployments

### 2. **Dual Storage Strategy**

The combination of Table and vector storage serves different access patterns:

- **Table**: O(1) access for "get document by ID" operations
- **Vector**: O(n) iteration for "get all documents matching criteria"
- **Consistency**: Both must be updated together to prevent data corruption

### 3. **Event-Driven Architecture**

Events enable the dApp to be **reactive** rather than polling-based:

- Real-time notifications when documents are signed
- Activity feeds and audit logs
- Integration with external systems (email, webhooks)
- Analytics and business intelligence

### 4. **Permission Model**

The separation between `allowed_signers` and `signatures` provides flexibility:

- **allowed_signers**: Who *can* sign (permission)
- **signatures**: Who *has* signed (action log)
- This enables workflows where signing permissions change over time

### 5. **Fee Structure**

The upload fee serves multiple purposes beyond revenue:

- **Spam prevention**: Small barrier to entry
- **Quality signal**: Users think twice before uploading
- **Sustainability**: Covers storage and computational costs
- **Economic alignment**: Users with skin in the game use the system responsibly

## Security Considerations

### Access Control

Every state-changing function includes appropriate authorization checks:

- Document owners can add/remove signers
- Only authorized signers can sign documents
- Admin functions are protected by ownership verification

### Data Integrity

The dual storage approach requires careful synchronization:

- Both Table and vector must be updated atomically
- Inconsistency would break queries and create undefined behavior
- Helper functions encapsulate this complexity

## Conclusion

This document management system demonstrates several important blockchain development patterns:

1. **Resource Account Management**: Using resource accounts for module state and fee collection
2. **Hybrid Storage Design**: Combining different data structures for optimal access patterns
3. **Event-Driven Architecture**: Enabling reactive frontend and external integrations
4. **Security-First Design**: Comprehensive authorization and validation throughout
5. **Economic Incentives**: Using fees to align user behavior with system health

The system balances functionality, security, and performance while providing a foundation for real-world document signing workflows.

Next Tutorial: [Frontend Integration Guide](./building_the_document_management_interface);