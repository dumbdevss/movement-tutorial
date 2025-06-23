# NFT Ticketing Contract Tutorial - Complete Implementation Guide

This comprehensive guide walks through building a complete NFT ticketing system on the Movement blockchain. We'll implement both soulbound (non-transferable) and transferable tickets with proper validation mechanisms, payment processing, and event tracking.

## Understanding Key Concepts Before We Start

### What is an NFT Ticketing System?

An **NFT ticketing system** transforms traditional event tickets into blockchain-based digital assets. This provides several advantages:

- **Ownership verification**: Cryptographic proof of ticket ownership
- **Anti-counterfeiting**: Impossible to create fake tickets
- **Programmable logic**: Smart contracts enforce business rules
- **Secondary markets**: Controlled resale with royalties
- **Event validation**: Cryptographic proof of attendance

### Soulbound vs Transferable Tickets

**Soulbound Tickets**:

- Cannot be transferred after minting
- Permanently linked to the original purchaser
- Perfect for personal events, conferences, or exclusive access
- Prevents scalping and unauthorized resale

**Transferable Tickets**:

- Can be sold or gifted to other users
- Enable legitimate secondary markets
- Useful for general admission events
- Allow flexibility in attendance planning

### Resource Accounts in NFT Systems

A **resource account** is a special blockchain account controlled by a module, not by a user. In NFT systems, resource accounts are used to:

- Store shared resources and global state for the contract
- Collect and manage fees from ticket sales
- Hold administrative capabilities securely
- Serve as the owner of the NFT collection and minted tokens

## TODO 1: Define Constants and Configuration

```move
// Collection and system configuration
const COLLECTION_NAME: vector<u8> = b"Movement Event Tickets";
const COLLECTION_DESCRIPTION: vector<u8> = b"Official event tickets powered by Movement blockchain";
const COLLECTION_URI: vector<u8> = b"https://api.movementtickets.com/collection";
const SEED: vector<u8> = b"movement_tickets_v1";
const TICKET_PRICE: u64 = 50000000; // 0.5 MOVE (50,000,000 microMOVE)
```

**Why These Constants Matter:**

- **COLLECTION_NAME**: Creates a branded identity for all tickets under this system
- **COLLECTION_DESCRIPTION**: Provides context for NFT marketplaces and wallets
- **COLLECTION_URI**: Points to metadata describing the entire collection
- **SEED**: Ensures deterministic resource account creation for consistent deployment
- **TICKET_PRICE**: Set in microMOVE (1 MOVE = 100,000,000 microMOVE) for precise pricing

## TODO 2: Define Error Codes

```move
// Error codes for comprehensive error handling
const E_NOT_AUTHORIZED: u64 = 1001;
const E_TICKET_USED: u64 = 1002;
const E_TICKET_NOT_FOUND: u64 = 1003;
const E_TICKET_LIMIT_REACHED: u64 = 1004;
const E_ACCOUNT_DOES_NOT_EXIST: u64 = 1005;
const E_INSUFFICIENT_FUNDS: u64 = 1006;
```

**Error Code Strategy:**
Each error provides specific information about what went wrong:

- **E_NOT_AUTHORIZED**: User lacks permission for the requested action
- **E_TICKET_USED**: Attempt to use an already-validated ticket
- **E_TICKET_NOT_FOUND**: Referenced ticket doesn't exist
- **E_TICKET_LIMIT_REACHED**: Maximum ticket count exceeded
- **E_ACCOUNT_DOES_NOT_EXIST**: Target account not initialized
- **E_INSUFFICIENT_FUNDS**: User lacks required MOVE tokens

**Implementation Benefits:**

- Clear error messages improve debugging
- Consistent numbering scheme makes maintenance easier

## TODO 3: Define the TicketState Struct

```move
struct TicketState has key {
    admin: address,                                    // System administrator
    ticket_counter: u64,                              // Total tickets minted
    max_tickets: u64,                                 // Maximum tickets allowed
    token_addresses: Table<u64, Object<Token>>,       // Maps ticket_id to token object
    mint_events: EventHandle<TicketMintEvent>,        // Mint event tracking
    transfer_events: EventHandle<TicketTransferEvent>, // Transfer event tracking
    validation_events: EventHandle<TicketValidationEvent>, // Validation event tracking
}
```

**Understanding the TicketState Design:**

The `key` ability makes this a **global resource** - exactly one instance exists per resource account. This serves as our system's central database.

**Field Explanations:**

- **admin**: The system administrator who can perform privileged operations
- **ticket_counter**: Provides analytics and can prevent spam by limiting ticket creation rates
- **max_tickets**: Implements artificial scarcity or practical limits
- **token_addresses**: Maps sequential ticket IDs to actual NFT token objects for efficient lookup
- **Event Handles**: Enable off-chain systems to track all ticket activities

**Why Use Table for Token Storage?**

- O(1) lookup time for finding tokens by ticket ID
- Scales efficiently as the number of tickets grows
- Provides clean separation between ticket numbering and NFT object addresses

## TODO 4: Define the TicketInfo Struct

```move
struct TicketInfo has store, copy, drop, key {
    ticket_id: u64,              // Unique sequential ID
    metadata_hash: String,        // IPFS hash or other metadata reference
    event_id: u64,               // Links to specific event
    is_used: bool,               // Validation status
    owner: address,              // Current ticket owner
    is_soulbound: bool,          // Transfer restriction flag
    purchase_time: u64,          // Purchase timestamp
    usage_time: u64,             // Validation timestamp (0 if unused)
}
```

**Understanding Move Abilities:**

- **store**: Allows storage inside other structs and global storage
- **copy**: Enables efficient data sharing and return values
- **drop**: Allows automatic cleanup when no longer needed
- **key**: Makes this a global resource stored at token addresses

**Field Design Rationale:**

- **ticket_id**: Sequential numbering for human-readable identification
- **metadata_hash**: Off-chain storage reference (IPFS, ARWEAVE, etc.)
- **event_id**: Enables multi-event support within the same contract
- **is_used**: Prevents double-validation attacks
- **owner**: Tracks current ownership for access control
- **is_soulbound**: Enforces transfer restrictions programmatically
- **purchase_time**: Audit trail and analytics
- **usage_time**: Proof of attendance timestamp

**Why Store at Token Addresses?**
By storing `TicketInfo` at each token's address, we create a direct link between the NFT and its metadata. This pattern:

- Provides O(1) access to ticket information
- Ensures data consistency with token ownership
- Enables easy verification of ticket properties

## TODO 5: Define the TicketMintEvent Struct

```move
#[event]
struct TicketMintEvent has drop, store {
    ticket_id: u64,         // Newly created ticket ID
    owner: address,         // Initial ticket owner
    event_id: u64,         // Associated event
    is_soulbound: bool,    // Transfer restriction status
}
```

**Event Design Philosophy:**
Mint events provide essential information for:

- **Frontend notifications**: "Your ticket has been minted!"
- **Analytics systems**: Track minting patterns and popular events
- **Marketplace integration**: New tickets available for discovery
- **Audit trails**: Complete history of ticket creation

**Why These Specific Fields?**

- **ticket_id**: Enables frontend to immediately fetch detailed ticket information
- **owner**: Allows filtering events by user for personalized dashboards
- **event_id**: Enables event-specific analytics and notifications
- **is_soulbound**: UI can immediately show transfer restrictions

**The `#[event]` Annotation:**
This compiler directive:

- Optimizes the struct for event emission
- Enables automatic indexing by blockchain infrastructure
- Allows efficient querying by off-chain systems

## TODO 6: Implement the init_module Function

```move
fun init_module(admin: &signer) {
    let admin_addr = signer::address_of(admin);

    // Create resource account for ticket management
    let (resource_signer, signer_cap) = account::create_resource_account(admin, SEED);
    let resource_addr = signer::address_of(&resource_signer);

    // Register the resource account to receive APT payments
    if (!coin::is_account_registered<AptosCoin>(resource_addr)) {
        coin::register<AptosCoin>(&resource_signer);
    };

    // Create the NFT collection under the resource account
    collection::create_unlimited_collection(
        &resource_signer,
        string::utf8(COLLECTION_DESCRIPTION),
        string::utf8(COLLECTION_NAME),
        option::none(), // No maximum supply
        string::utf8(COLLECTION_URI),
    );

    // Initialize TicketState under the resource account
    move_to(&resource_signer, TicketState {
        admin: admin_addr,
        ticket_counter: 0,
        max_tickets: 1000000, // Set reasonable limit
        token_addresses: table::new(),
        mint_events: account::new_event_handle<TicketMintEvent>(&resource_signer),
        transfer_events: account::new_event_handle<TicketTransferEvent>(&resource_signer),
        validation_events: account::new_event_handle<TicketValidationEvent>(&resource_signer),
    });

    // Store the SignerCapability under the admin's address
    move_to(admin, ResourceAccountCap {
        signer_cap,
    });
}
```

**The Module Initialization Pattern:**
`init_module` executes exactly once when the module is first deployed. This is the perfect place to:

- Set up the resource account infrastructure
- Create the NFT collection
- Initialize system state
- Establish administrative controls

**Resource Account Setup Process:**

1. **Create Resource Account**: Generates a deterministic address controlled by our module
2. **Register for Payments**: Enables the system to receive MOVE tokens from ticket sales
3. **Create Collection**: Establishes the NFT collection that will contain all tickets
4. **Initialize State**: Sets up the global state with empty collections and event handles
5. **Store Capabilities**: Gives the admin control over the resource account

**Collection Configuration:**

- `create_unlimited_collection`: No hard cap on ticket supply (flexible for different events)
- Collection metadata provides context for NFT marketplaces
- URI should point to collection-level metadata (logo, description, etc.)

**Implementation Steps:**

1. Extract admin address for authorization tracking
2. Create resource account with deterministic seed
3. Enable the resource account to receive token payments
4. Create NFT collection with appropriate metadata
5. Initialize global state with proper event handles
6. Store admin capabilities securely

## TODO 7: Implement the mint_soulbound_ticket Function

```move
public entry fun mint_soulbound_ticket(
    user: &signer,
    event_id: u64,
    metadata_hash: String,
    token_name: String,
    token_uri: String
) acquires TicketState, ResourceAccountCap {
    let user_addr = signer::address_of(user);

    // Get the resource account address and state
    let resource_address = get_resource_address(@ticket_nft);
    let state = borrow_global_mut<TicketState>(resource_address);

    // Validate system limits
    assert_ticket_limit_not_reached(state);

    // Handle payment - check balance and collect fee
    assert_user_has_enough_apt(user_addr, TICKET_PRICE);
    let payment = coin::withdraw<AptosCoin>(user, TICKET_PRICE);
    coin::deposit<AptosCoin>(resource_address, payment);

    // Get the new ticket ID and verify user account exists
    let new_ticket_id = state.ticket_counter + 1;
    assert!(account::exists_at(user_addr), error::not_found(E_ACCOUNT_DOES_NOT_EXIST));

    // Mint the NFT under the resource account
    let resource_signer = get_resource_signer();
    let constructor_ref = token::create(
        &resource_signer,
        string::utf8(COLLECTION_NAME),
        string::utf8(b"Soulbound Movement Ticket"),
        token_name,
        option::none(),
        token_uri,
    );

    // Get the token object and transfer references
    let token_object = object::object_from_constructor_ref<Token>(&constructor_ref);
    let transfer_ref = object::generate_transfer_ref(&constructor_ref);

    // Make the token soulbound by disabling ungated transfers
    object::disable_ungated_transfer(&transfer_ref);

    // Transfer the token to the user
    let linear_transfer_ref = object::generate_linear_transfer_ref(&transfer_ref);
    object::transfer_with_ref(linear_transfer_ref, user_addr);

    // Create and store ticket information
    let ticket_info = TicketInfo {
        ticket_id: new_ticket_id,
        metadata_hash,
        event_id,
        is_used: false,
        owner: user_addr,
        is_soulbound: true,
        purchase_time: timestamp::now_seconds(),
        usage_time: 0,
    };

    // Store TicketInfo at the token's address
    move_to(&object::generate_signer(&constructor_ref), ticket_info);

    // Store token address in state for lookup
    table::add(&mut state.token_addresses, new_ticket_id, token_object);

    // Emit mint event for off-chain tracking
    event::emit_event(&mut state.mint_events, TicketMintEvent {
        ticket_id: new_ticket_id,
        owner: user_addr,
        event_id,
        is_soulbound: true,
    });

    // Update ticket counter
    state.ticket_counter = new_ticket_id;
}
```

**Understanding Soulbound Token Creation:**

The key difference in soulbound tokens is the call to `object::disable_ungated_transfer(&transfer_ref)`. This permanently prevents the token from being transferred through normal means, making it "soulbound" to the original owner.

**Payment Processing Flow:**

1. **Pre-flight Check**: Verify user has sufficient funds before any state changes
2. **Atomic Withdrawal**: Extract exact payment amount from user's account
3. **Secure Deposit**: Transfer funds to resource account immediately
4. **No Partial States**: Either the entire operation succeeds or fails completely

**NFT Creation Process:**

1. **Create Token**: Generate a new NFT with specified metadata
2. **Get References**: Obtain transfer capabilities for the new token
3. **Disable Transfers**: Make the token soulbound (crucial step!)
4. **Initial Transfer**: Move token from resource account to user
5. **Store Metadata**: Create comprehensive ticket information

**Data Storage Strategy:**

- **TicketInfo**: Stored at token address for direct association
- **Token Mapping**: Stored in global state for ID-based lookups
- **Event Emission**: Enables real-time notifications and analytics

**Implementation Steps:**

1. Validate user eligibility and system limits
2. Process payment atomically
3. Create NFT with appropriate metadata
4. Make token soulbound by disabling transfers
5. Transfer token to user
6. Store comprehensive ticket information
7. Update system state and emit events

## TODO 8: Implement the mint_transferable_ticket Function

```move
public entry fun mint_transferable_ticket(
    user: &signer,
    event_id: u64,
    metadata_hash: String,
    token_name: String,
    token_uri: String
) acquires TicketState, ResourceAccountCap {
    let user_addr = signer::address_of(user);

    // Get the resource account address and state
    let resource_address = get_resource_address(@ticket_nft);
    let state = borrow_global_mut<TicketState>(resource_address);

    // Validate system limits
    assert_ticket_limit_not_reached(state);

    // Handle payment - check balance and collect fee
    assert_user_has_enough_apt(user_addr, TICKET_PRICE);
    let payment = coin::withdraw<AptosCoin>(user, TICKET_PRICE);
    coin::deposit<AptosCoin>(resource_address, payment);

    // Get the new ticket ID and verify user account exists
    let new_ticket_id = state.ticket_counter + 1;
    assert!(account::exists_at(user_addr), error::not_found(E_ACCOUNT_DOES_NOT_EXIST));

    // Mint the NFT under the resource account
    let resource_signer = get_resource_signer();
    let constructor_ref = token::create(
        &resource_signer,
        string::utf8(COLLECTION_NAME),
        string::utf8(b"Transferable Movement Ticket"),
        token_name,
        option::none(),
        token_uri,
    );

    // Get the token object and transfer to user
    let token_object = object::object_from_constructor_ref<Token>(&constructor_ref);
    let transfer_ref = object::generate_transfer_ref(&constructor_ref);
    let linear_transfer_ref = object::generate_linear_transfer_ref(&transfer_ref);
    object::transfer_with_ref(linear_transfer_ref, user_addr);

    // Note: We DO NOT disable transfers - this keeps the token transferable

    // Create and store ticket information
    let ticket_info = TicketInfo {
        ticket_id: new_ticket_id,
        metadata_hash,
        event_id,
        is_used: false,
        owner: user_addr,
        is_soulbound: false, // Transferable tickets are not soulbound
        purchase_time: timestamp::now_seconds(),
        usage_time: 0,
    };

    // Store TicketInfo at the token's address
    move_to(&object::generate_signer(&constructor_ref), ticket_info);

    // Store token address in state for lookup
    table::add(&mut state.token_addresses, new_ticket_id, token_object);

    // Emit mint event for off-chain tracking
    event::emit_event(&mut state.mint_events, TicketMintEvent {
        ticket_id: new_ticket_id,
        owner: user_addr,
        event_id,
        is_soulbound: false,
    });

    // Update ticket counter
    state.ticket_counter = new_ticket_id;
}
```

**Key Differences from Soulbound Tickets:**

The critical difference is that we **do not call** `object::disable_ungated_transfer()`. This allows the token to remain transferable through standard NFT transfer mechanisms.

**Transfer Capability Management:**
For transferable tickets, the transfer capability remains active, enabling:

- Standard wallet-to-wallet transfers
- Marketplace transactions
- Gifting and resale functionality
- Integration with existing NFT infrastructure

**Business Logic Considerations:**

- Transferable tickets might have different pricing strategies
- Secondary market royalties could be implemented
- Event organizers might track ownership changes
- Anti-fraud measures might monitor rapid transfers

**Implementation Steps:**

1. Follow the same validation and payment process as soulbound tickets
2. Create NFT with identical process
3. Transfer to user WITHOUT disabling future transfers
4. Store ticket information with `is_soulbound: false`
5. Update system state and emit appropriate events

## TODO 9: Implement the transfer_ticket Function

```move
public entry fun transfer_ticket(
    owner: &signer,
    receiver: address,
    token_address: Object<Token>
) acquires TicketState, TicketInfo {
    let owner_addr = signer::address_of(owner);

    // Verify the token exists and get TicketInfo
    let token_addr = object::object_address(&token_address);
    assert!(object::is_object(token_addr), error::not_found(E_TICKET_NOT_FOUND));
    let ticket = borrow_global_mut<TicketInfo>(token_addr);

    // Perform comprehensive authorization and validation checks
    assert_ticket_owner(ticket, owner_addr);
    assert_ticket_not_used(ticket);
    assert_ticket_transferable(ticket);

    // Update the ticket owner in our records
    ticket.owner = receiver;

    // Perform the actual NFT transfer
    object::transfer(owner, token_address, receiver);

    // Get system state for event emission
    let resource_address = get_resource_address(@ticket_nft);
    let state = borrow_global_mut<TicketState>(resource_address);

    // Emit transfer event for tracking
    event::emit_event(&mut state.transfer_events, TicketTransferEvent {
        ticket_id: ticket.ticket_id,
        from: owner_addr,
        to: receiver,
    });
}
```

**Understanding Token Transfer Mechanics:**

The `object::transfer()` function handles the actual NFT ownership change on the blockchain level. Our contract needs to:

1. Verify the user has permission to transfer
2. Update our internal records to match
3. Emit events for off-chain tracking

**Security Validation Chain:**

- **Token Existence**: Prevents operations on non-existent tokens
- **Ownership Verification**: Only current owner can transfer
- **Usage Status**: Prevents transfer of already-validated tickets
- **Transfer Capability**: Ensures token isn't soulbound

**Why Check Ticket Usage?**
Preventing transfer of used tickets serves several purposes:

- **Event Security**: Used tickets shouldn't circulate
- **User Protection**: Prevents selling worthless tickets
- **Data Integrity**: Maintains clear ownership history

**Event Emission Importance:**
Transfer events enable:

- Real-time notifications to both parties
- Marketplace activity tracking
- Ownership history for analytics
- Integration with external systems

**Implementation Steps:**

1. Verify token exists and access its TicketInfo
2. Perform all security and validation checks
3. Update internal ownership records
4. Execute blockchain-level transfer
5. Emit transfer event for off-chain systems

## TODO 10: Implement the validate_ticket Function

```move
public entry fun validate_ticket(
    user: &signer,
    token_address: Object<Token>
) acquires TicketInfo, TicketState {
    let user_addr = signer::address_of(user);

    // Verify the token exists and get TicketInfo
    let token_addr = object::object_address(&token_address);
    assert!(object::is_object(token_addr), error::not_found(E_TICKET_NOT_FOUND));
    let ticket = borrow_global_mut<TicketInfo>(token_addr);

    // Verify ownership and usage status
    assert_ticket_owner(ticket, user_addr);
    assert_ticket_not_used(ticket);

    // Mark ticket as used and record validation time
    ticket.is_used = true;
    ticket.usage_time = timestamp::now_seconds();

    // Get system state for event emission
    let resource_address = get_resource_address(@ticket_nft);
    let state = borrow_global_mut<TicketState>(resource_address);

    // Emit validation event for tracking and analytics
    event::emit_event(&mut state.validation_events, TicketValidationEvent {
        ticket_id: ticket.ticket_id,
        owner: user_addr,
    });
}
```

**Ticket Validation Philosophy:**

Validation represents the culmination of the ticketing process - when a ticket is actually used for its intended purpose. This function:

- **Authenticates**: Verifies the user owns the ticket
- **Validates**: Confirms the ticket hasn't been used before
- **Records**: Creates an immutable timestamp of usage
- **Notifies**: Emits events for external systems

**Anti-Double-Spending Protection:**
The `assert_ticket_not_used()` check prevents the same ticket from being validated multiple times, which is crucial for:

- Event security (no sneaking in multiple times)
- Data integrity (clear usage records)
- Business logic (capacity planning)

**Implementation Steps:**

1. Verify token exists and access ticket information
2. Confirm user ownership of the ticket
3. Ensure ticket hasn't been previously validated
4. Mark ticket as used with current timestamp
5. Emit validation event for external tracking

## TODO 11: Implement get_resource_address Helper Function

```move
inline fun get_resource_address(account: address): address {
    account::create_resource_address(&account, SEED)
}
```

**Understanding Resource Address Generation:**

This helper function creates a **deterministic address** for our resource account. The address is computed using:

- The deployer's address (passed as `account`)
- The seed constant (defined at module level)
- A cryptographic hash function

**The `inline` Keyword:**
Using `inline` tells the compiler to replace function calls with the actual code, eliminating function call overhead for this simple operation.

## TODO 12: Implement get_resource_signer Helper Function

```move
inline fun get_resource_signer(): signer acquires ResourceAccountCap {
    let cap = borrow_global<ResourceAccountCap>(@ticket_nft);
    account::create_signer_with_capability(&cap.signer_cap)
}
```

**Understanding Signer Capability Usage:**

The `SignerCapability` is like a "master key" that allows our module to:

- Act on behalf of the resource account
- Sign transactions programmatically
- Manage resources stored under the resource account
- Perform administrative operations

**Security Model:**

- The capability is stored under the module deployer's address
- Only functions in this module can access it (through `acquires`)
- The resource account itself cannot use the capability
- This prevents unauthorized access to system functions

**When This Function Is Used:**

- Creating NFT tokens (requires resource account signature)
- Managing the NFT collection
- Performing administrative operations
- Any action that needs resource account authorization

**Error Handling:**
If `ResourceAccountCap` doesn't exist, this function will abort with a runtime error, which is appropriate since the module should always be properly initialized.

**Implementation Details:**

- `borrow_global` accesses the stored capability
- `create_signer_with_capability` converts capability to usable signer
- `inline` eliminates function call overhead
- `acquires` annotation tracks resource access for Move's borrow checker

## TODO 13: Implement assert_user_has_enough_apt Function

```move
fun assert_user_has_enough_apt(user_addr: address, amount: u64) {
    let balance = coin::balance<AptosCoin>(user_addr);
    assert!(balance >= amount, error::invalid_state(E_INSUFFICIENT_FUNDS));
}
```

**Payment Validation Strategy:**

This function implements a **fail-fast** approach to payment validation:

- Check user's balance before any state changes
- Provide clear error message if insufficient funds

**Why Check Before Withdraw?**
While `coin::withdraw` would also fail with insufficient funds, checking first provides:

- **Explicit Validation**: Makes business logic more obvious
- **Consistent Error Handling**: Uses our custom error codes

## TODO 14: Implement get_ticket_status View Function

```move
#[view]
public fun get_ticket_status(token_address: Object<Token>): (bool, address, u64, String, bool) acquires TicketInfo {
    let token_addr = object::object_address(&token_address);
    assert!(object::is_object(token_addr), error::not_found(E_TICKET_NOT_FOUND));
    
    let ticket = borrow_global<TicketInfo>(token_addr);
    (
        ticket.is_used,
        ticket.owner,
        ticket.event_id,
        ticket.metadata_hash,
        ticket.is_soulbound
    )
}
```

**View Function Design Philosophy:**

View functions are **read-only** operations that enable external queries without modifying blockchain state. This function provides essential ticket information for:

- Frontend display and validation
- Event management systems
- Marketplace integrations
- User account dashboards

**Return Tuple Structure:**
The function returns a carefully selected tuple containing:

1. **is_used**: Critical for validation systems
2. **owner**: Current ticket ownership
3. **event_id**: Links ticket to specific event
4. **metadata_hash**: Access to off-chain content
5. **is_soulbound**: Transfer capability status

**Implementation Steps:**

1. Add `#[view]` annotation for read-only optimization
2. Verify token exists to prevent errors
3. Access ticket information safely
4. Return essential data as an efficiently-structured tuple


## TODO 15: Implement get_tickets_by_user View Function

```move
#[view]
public fun get_tickets_by_user(user_addr: address): vector<TicketInfo> acquires TicketState, TicketInfo {
    let resource_address = get_resource_address(@ticket_nft);
    let state = borrow_global<TicketState>(resource_address);
    let user_tickets = vector::empty<TicketInfo>();
    
    let i = 0;
    let len = vector::length(&state.token_addresses);
    
    while (i < len) {
        let token_address = *vector::borrow(&state.token_addresses, i);
        let token_addr = object::object_address(&token_address);
        
        if (exists<TicketInfo>(token_addr)) {
            let ticket = borrow_global<TicketInfo>(token_addr);
            if (ticket.owner == user_addr) {
                vector::push_back(&mut user_tickets, *ticket);
            };
        };
        
        i = i + 1;
    };
    
    user_tickets
}
```

**Understanding User Ticket Aggregation:**

This function demonstrates a critical pattern for blockchain applications: **efficient data aggregation**. Unlike traditional databases where you might use a SQL JOIN or index, blockchain systems require explicit iteration through stored data.

**Error Resilience:**
The `if (exists<TicketInfo>(token_addr))` check ensures the function continues working even if some tokens have been destroyed or corrupted. This defensive programming prevents the entire query from failing due to one problematic token.

**Gas Considerations:**
While this function iterates through all tickets, it's marked as `#[view]` so it doesn't consume gas during execution. However, the response size grows with the total number of tickets in the system, so consider pagination for production deployments.

**Frontend Integration Benefits:**

- **User Dashboard**: Display all tickets owned by a user
- **Portfolio Management**: Show ticket values and status
- **Transfer History**: Track ticket ownership over time
- **Event Participation**: See which events a user has tickets for

**Implementation Steps:**

1. Get the resource account address using the helper function
2. Access the global TicketState to get the token_addresses vector
3. Create an empty vector to store matching tickets
4. Iterate through all token addresses in the system
5. For each token, check if TicketInfo exists at that address
6. If the ticket exists and is owned by the user, add it to results
7. Return the collected vector of user's tickets

## Key Design Decisions Explained

### 1. **Soulbound vs. Transferable Tickets**

The system supports two ticket types because different events have different requirements:

- **Soulbound Tickets**: Perfect for exclusive events, identity verification, or preventing scalping
- **Transferable Tickets**: Enable secondary markets, gift transfers, and flexible ownership

The `is_soulbound` flag in `TicketInfo` and the conditional transfer logic in `mint_soulbound_ticket()` vs `mint_transferable_ticket()` implement this distinction cleanly.

### 2. **Resource Account Architecture**

Using a resource account provides several advantages:

- **Centralized State**: All tickets and system data live at a predictable address
- **Revenue Collection**: APT fees are collected in the resource account
- **Administrative Control**: Admin retains control through stored SignerCapability
- **Upgrade Safety**: Module upgrades don't affect stored data

### 3. **Token Object Integration**

The system leverages Aptos Token Objects for several benefits:

- **Standards Compliance**: Works with existing NFT infrastructure
- **Built-in Functionality**: Transfer, ownership, and metadata handling
- **Ecosystem Integration**: Compatible with wallets, marketplaces, and explorers
- **Efficient Storage**: Token data is stored optimally by the framework

### 4. **Event System Design**

The comprehensive event system enables:

- **Real-time Notifications**: Frontends can react to ticket operations
- **Analytics**: Track ticket sales, transfers, and validation patterns
- **External Integrations**: Connect with email systems, mobile apps, etc.
- **Audit Trails**: Permanent record of all ticket operations

## Conclusion

This tutorial provided a step-by-step guide to building a robust NFT ticketing system on the Movement blockchain. By leveraging Move's resource-oriented programming model, we implemented both soulbound and transferable tickets, secure payment processing, and comprehensive event tracking. The architecture ensures strong security, efficient data access, and seamless integration with external systems. With these foundations, you can extend the system for advanced features such as dynamic pricing, secondary market royalties, or multi-event management. This approach empowers event organizers and users with transparent, tamper-proof ticketing and paves the way for innovative blockchain-powered experiences.

Next Tutorial: [Smart Contract testing](./testing_application)
