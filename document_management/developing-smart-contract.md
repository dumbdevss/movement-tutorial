# NFT Ticketing Contract Tutorial - Part 2

In this section of our tutorial, we'll be walking through the completed secure document management system that we started building earlier. This fully-functional Move module allows users to upload documents with IPFS storage, add signers, and manage digital signatures securely on the Aptos blockchain.

## Contract Overview

The `secure_docs` module provides a robust system for document management with the following key features:
- Document upload with IPFS storage integration
- Fee-based document upload using Aptos Coin (APT)
- Adding and removing authorized signers
- Digital signature collection
- Comprehensive event tracking
- View functions for data access

Let's go through the implementation step by step, following the TODO items from our starter code.

## Step 1: Defining Constants

First, we need to define the module's constants:

```move
// Constants
const SEED: vector<u8> = b"secure_docs";
const UPLOAD_FEE: u64 = 5000000; // 0.05 APT (in microAPT, 1 APT = 10^8 microAPT)
```

The `SEED` is used to create a deterministic resource account, while the `UPLOAD_FEE` defines how much users pay to upload a document.

## Step 2: Creating the Document Struct

Next, we need a struct to store document metadata:

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

This struct includes all necessary fields to track a document's metadata and signature status. The `has store, drop, copy` abilities allow for flexible storage and manipulation.

## Step 3: Creating the State Struct

For global state management, we create:

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

This struct stores the module's state, including document storage (both indexed and in sequence), event handlers, and administrative information.

## Step 4: Creating Event Structs

For event tracking, we define:

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

These events allow off-chain tracking of document uploads, signatures, and permission grants.

## Step 5: Initializing the Module

The `init_module` function sets up our system and it called once the contract is deployed:

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

This creates a resource account to hold the contract state and registers it for APT coin operations.

## Steps 6-8: Helper Functions

These helper functions make our code cleaner:

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

## Steps 9-10: Document Upload Function

The document upload function handles payment and document creation:

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

## Step 11: Adding Signers

The `add_signer` function allows document owners to add authorized signers:

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

## Step 12: Signing Documents

The `sign_document` function enables authorized users to sign documents:

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

## Step 13: Removing Signers

The `remove_signer` function allows document owners to revoke signing privileges:

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

## Steps 14-16: View Functions

Finally, the view functions provide data access without modifying state:

```move
#[view]
public fun get_document(id: String): (String, String, u64, address, vector<address>, vector<address>) acquires DocState, ResourceAccountCap {
    let state = borrow_global<DocState>(get_resource_address());
    assert_document_exists(state, &id);

    let doc = table::borrow(&state.documents, id);
    (doc.name, doc.ipfs_hash, doc.created_at, doc.owner, doc.signatures, doc.allowed_signers)
}

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

## Conclusion

The completed secure document management system provides a robust foundation for blockchain-based document signing. It leverages Aptos's resource accounts, tables, and event systems to create a secure and efficient solution. Key points include:
