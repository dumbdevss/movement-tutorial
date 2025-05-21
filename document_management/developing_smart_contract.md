# NFT Ticketing Contract Tutorial - Implementation Guide

This guide walks through each TODO item in the secure document management system with clear explanations and implementation details. Follow these steps to complete your Move module implementation.

## Source Code Reference

The complete implementation of this smart contract can be found in the repository:

**Repository:** [dumbdevss/document-management](https://github.com/dumbdevss/document-management)  
**Branch:** `smart-contract`

You can view or clone the finished code to compare against your implementation as you follow along with this guide.

## TODO 1: Define Constants

```move
// Constants
const SEED: vector<u8> = b"secure_docs";
const UPLOAD_FEE: u64 = 5000000; // 0.05 APT (in microAPT, 1 APT = 10^8 microAPT)
```

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

**Implementation steps:**

1. Define a struct with `store`, `drop`, and `copy` abilities
2. Include fields for document metadata (ID, name, IPFS hash)
3. Add timestamp field for audit purposes
4. Include ownership tracking with `owner` field
5. Add vectors to track both actual signatures and authorized signers

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

**Implementation steps:**

1. Access the global state
2. Verify document exists and caller is owner
3. Update both table and vector entries for consistency
4. Only add signer if not already present
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

**Implementation steps:**

1. Access global state and verify document exists
2. Verify caller is document owner
3. Find signer in the allowed signers list
4. Remove signer from both table and vector entries
5. Abort if signer not found

## TODO 14-16: Implement View Functions

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

**Implementation steps:**

1. Add `#[view]` annotation for read-only function
2. Create an empty result vector to store matching documents
3. Iterate through all documents in the `document_vec`
4. For each document, check if the user's address matches the document's `owner` field
5. If so, add the document to the result vector
6. Return the filtered list of documents

## Key Implementation Considerations

1. **Dual Storage Strategy**: The implementation uses both a Table (for fast lookups by ID) and a vector (for iteration), which requires careful synchronization between the two.

2. **Resource Account Pattern**: The module uses Aptos's resource account pattern to manage state and collect fees.

3. **Data Consistency**: When updating document data (adding/removing signers, collecting signatures), both storage structures must be updated in sync.

4. **Authorization Checks**: Before performing sensitive operations, the code verifies that the caller has appropriate permissions.

5. **Event Emission**: Events are emitted for all state-changing operations to enable off-chain tracking.

6. **View Functions**: The module provides several view functions for easy data access without modifying state.

## Testing Recommendations

To ensure your implementation works correctly:

1. Test document upload with valid and invalid IPFS hashes
2. Verify fee collection works correctly
3. Test adding and removing signers with different permission scenarios
4. Verify signature collection with authorized and unauthorized signers
5. Test view functions to ensure they return expected data
6. Verify event emission for all state-changing operations

## Conclusion

Congratulations! You've successfully built a complete blockchain document management smart contract by:

1. Implementing the Move smart contract with:
   - Document creation and storage
   - Signer management functionality
   - Secure signature verification

Next Tutorial: [Testing the Document Management DApp](./building_the_document_management_interface)
