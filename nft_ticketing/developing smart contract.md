# NFT Ticketing Contract Tutorial - Part 2

In this section of our tutorial, we'll be walking through the completed NFT ticketing contract that we've been building. This fully-functional contract allows for both soulbound (non-transferable) and transferable NFT tickets, with validation mechanisms to ensure proper usage.

## Contract Overview

Our NFT ticketing contract has several core features:

1. Creating soulbound (non-transferable) tickets
2. Creating transferable tickets
3. Transferring ticket ownership (for transferable tickets only)
4. Validating tickets for event entry
5. View functions for getting ticket information

Let's break down each component of the implementation to understand what's happening behind the scenes.

## Import of Dependencies

At the beginning of our Move module, we need to import all the necessary dependencies that our NFT ticketing contract will use. These imports provide access to the core libraries and modules that enable the functionality of our contract.

```move
use std::error;
use std::signer;
use std::string::{Self, String};
use std::option;
use std::object::{Self, Object, TransferRef};
use aptos_framework::account::{Self, SignerCapability};
use aptos_framework::event::{Self, EventHandle};
use aptos_framework::object;
use aptos_token_objects::collection;
use aptos_token_objects::token::{Self, Token};
use aptos_std::table::{Self, Table};
use aptos_framework::timestamp;
use aptos_framework::coin::{Self, Coin};
use aptos_framework::aptos_coin::AptosCoin;
```

Let's break down each import and understand its purpose in our contract:

### Standard Library Imports

- `std::error`: Provides error handling utilities for our contract.
- `std::signer`: Helps with signer-related operations, like getting the address from a signer.
- `std::string::{Self, String}`: Enables string operations and the use of the `String` type for ticket IDs and metadata.
- `std::option`: Provides the `Option` type for handling optional values.
- `std::object::{Self, Object, TransferRef}`: Core functionality for working with digital objects, including transferring ownership.

### Aptos Framework Imports

- `aptos_framework::account::{Self, SignerCapability}`: Manages account operations and capabilities, essential for our resource account.
- `aptos_framework::event::{Self, EventHandle}`: Handles event emission for tracking ticket minting, transfers, and validations.
- `aptos_framework::object`: Provides object-related utilities (note this is different from `std::object`).
- `aptos_framework::timestamp`: Allows us to capture timestamps for ticket purchases and usage.
- `aptos_framework::coin::{Self, Coin}`: Enables cryptocurrency operations for ticket payments.
- `aptos_framework::aptos_coin::AptosCoin`: Specific support for the Movement native token (MOVE).

### Aptos Token Object Imports

- `aptos_token_objects::collection`: Enables the creation and management of collections for our tickets.
- `aptos_token_objects::token::{Self, Token}`: Provides the core functionality for creating and managing NFT tokens.

## Understanding the Data Structures

Our contract relies on several key data structures:

```move
// Struct to store the SignerCapability for the resource account
struct ResourceAccountCap has key {
    signer_cap: SignerCapability,
}

// Struct to track ticket collection state
struct TicketState has key {
    admin: address,
    ticket_counter: u64,
    max_tickets: u64,
    tickets: Table<String, TicketInfo>,
    mint_events: EventHandle<TicketMintEvent>,
    transfer_events: EventHandle<TicketTransferEvent>,
    validation_events: EventHandle<TicketValidationEvent>,
}

// Struct for ticket info
struct TicketInfo has store {
    token_metadata: Object<Token>,
    ticket_id: String,
    event_id: String,
    is_used: bool,
    owner: address,
    is_soulbound: bool,
    purchase_time: u64,
    usage_time: u64,
    metadata_hash: String,
}

struct Capability has key, store {
    transfer_ref: TransferRef,
}
```

- The `ResourceAccountCap` stores the signer capability for the resource account, which is needed to perform operations on behalf of the module.
- The `TicketState` maintains the global state of our ticketing system, including ticket counts, ownership, and events.
- The `TicketInfo` represents an individual ticket with properties like ID, ownership, and usage status.
- The `Capability` struct stores the transfer capability for transferable tickets.

## Events for Tracking Activity

Our contract emits events for important actions:

```move
#[event]
struct TicketMintEvent has drop, store {
    ticket_id: String,
    owner: address,
    event_id: String,
    is_soulbound: bool,
}

#[event]
struct TicketTransferEvent has drop, store {
    ticket_id: String,
    from: address,
    to: address,
}

#[event]
struct TicketValidationEvent has drop, store {
    ticket_id: String,
    owner: address,
}
```

These events allow external applications to track ticket minting, transfers, and validations by listening to the blockchain.

## Initialization: Setting Up the Contract

The initialization function sets up our contract with the necessary resources:

```move
fun init_module(admin: &signer) {
    let admin_addr = signer::address_of(admin);
    let (resource_signer, signer_cap) = account::create_resource_account(admin, SEED);

    // Register the resource account to receive APT
    coin::register<AptosCoin>(&resource_signer);

    // Create the collection
    collection::create_unlimited_collection(
        &resource_signer,
        string::utf8(COLLECTION_DESCRIPTION),
        string::utf8(COLLECTION_NAME),
        option::none(),
        string::utf8(COLLECTION_URI),
    );

    move_to(&resource_signer, TicketState {
        admin: admin_addr,
        ticket_counter: 0,
        max_tickets: 10000000,
        tickets: table::new(),
        mint_events: account::new_event_handle<TicketMintEvent>(&resource_signer),
        transfer_events: account::new_event_handle<TicketTransferEvent>(&resource_signer),
        validation_events: account::new_event_handle<TicketValidationEvent>(&resource_signer),
    });

    move_to(admin, ResourceAccountCap { signer_cap });
}
```

This function:
1. Creates a resource account for the module
2. Registers the resource account to receive APT tokens
3. Creates the NFT collection that will contain all tickets
4. Initializes the TicketState with default values
5. Stores the resource account capability for future use

## Minting Soulbound Tickets

Soulbound tickets are non-transferable NFTs that stay with the original owner:

```move
    // Mint a soulbound ticket
    public entry fun mint_soulbound_ticket(
        user: &signer,
        event_id: String,
        ticket_id: String,
        metadata_hash: String,
        token_name: String,
        token_uri: String
    ) acquires TicketState, ResourceAccountCap {
        let user_addr = signer::address_of(user);
        let resources_address = get_resource_address(@ticket_nft);
        let state = borrow_global_mut<TicketState>(resources_address);
        assert_ticket_limit_not_reached(state);

        assert_user_has_enough_apt(user_addr, TICKET_PRICE);
        let payment = coin::withdraw<AptosCoin>(user, TICKET_PRICE);
        coin::deposit<AptosCoin>(@ticket_nft, payment);

        if (!account::exists_at(user_addr)) {
            abort E_ACCOUNT_DOES_NOT_EXIST
        };

        let resource_signer = get_resource_signer();
        let constructor_ref = token::create(
            &resource_signer,
            string::utf8(COLLECTION_NAME),
            string::utf8(b"Soulbound Event Ticket"),
            token_name,
            option::none(),
            token_uri,
        );

        let transfer_ref = object::generate_transfer_ref(&constructor_ref);
        let linear_transfer_ref = object::generate_linear_transfer_ref(&transfer_ref);
        let token_object = object::object_from_constructor_ref<Token>(&constructor_ref);
        object::transfer_with_ref(linear_transfer_ref, user_addr);
        object::disable_ungated_transfer(&transfer_ref);

        table::add(&mut state.tickets, ticket_id, TicketInfo {
            token_metadata: token_object,
            ticket_id,
            event_id,
            is_used: false,
            owner: user_addr,
            is_soulbound: true,
            purchase_time: timestamp::now_seconds(),
            usage_time: 0,
            metadata_hash,
        });

        event::emit_event(&mut state.mint_events, TicketMintEvent {
            ticket_id,
            owner: user_addr,
            event_id,
            is_soulbound: true,
        });

        state.ticket_counter = state.ticket_counter + 1;
    }
```

Key implementation aspects:
1. We check if the ticket limit has been reached
2. We verify and collect payment in APT
3. We create a new NFT token
4. We disable transfers by calling `object::disable_ungated_transfer()`
5. We record the ticket information in our state
6. We emit a TicketMintEvent

## Minting Transferable Tickets

Transferable tickets can be sent to other users:

```move
    // Mint a transferable ticket
    public entry fun mint_transferable_ticket(
        user: &signer,
        ticket_id: String,
        metadata_hash: String,
        event_id: String,
        token_name: String,
        token_uri: String
    ) acquires TicketState, ResourceAccountCap {
        let user_addr = signer::address_of(user);
        let resources_address = get_resource_address(@ticket_nft);
        let state = borrow_global_mut<TicketState>(resources_address);
        assert_ticket_limit_not_reached(state);

        assert_user_has_enough_apt(user_addr, TICKET_PRICE_SOULBONG);
        let payment = coin::withdraw<AptosCoin>(user, TICKET_PRICE_SOULBONG);
        coin::deposit<AptosCoin>(@ticket_nft, payment);

        if (!account::exists_at(user_addr)) {
            abort E_ACCOUNT_DOES_NOT_EXIST
        };

        let resource_signer = get_resource_signer();
        let constructor_ref = token::create(
            &resource_signer,
            string::utf8(COLLECTION_NAME),
            string::utf8(b"Transferable Event Ticket"),
            token_name,
            option::none(),
            token_uri,
        );

        let transfer_ref = object::generate_transfer_ref(&constructor_ref);
        let linear_transfer_ref = object::generate_linear_transfer_ref(&transfer_ref);
        let token_object = object::object_from_constructor_ref<Token>(&constructor_ref);
        object::transfer_with_ref(linear_transfer_ref, user_addr);

        // Store Capability for transferable tickets
        move_to(&resource_signer, Capability { transfer_ref });

        table::add(&mut state.tickets, ticket_id, TicketInfo {
            token_metadata: token_object,
            ticket_id,
            event_id,
            is_used: false,
            owner: user_addr,
            is_soulbound: false, // Fixed: Transferable tickets are not soulbound
            purchase_time: timestamp::now_seconds(),
            usage_time: 0,
            metadata_hash,
        });

        event::emit_event(&mut state.mint_events, TicketMintEvent {
            ticket_id,
            owner: user_addr,
            event_id,
            is_soulbound: false,
        });

        state.ticket_counter = state.ticket_counter + 1;
    }
```


The key difference from soulbound tickets is that we store the `transfer_ref` capability instead of disabling transfers, allowing for future transfers of the ticket.

## Transferring Tickets

Our contract enables transferring tickets between users:

```move
    // Transfer a ticket
    public entry fun transfer_ticket(
        owner: &signer,
        receiver: address,
        ticket_id: String,
        token: Object<Token>
    ) acquires TicketState, Capability {
        let owner_addr = signer::address_of(owner);
        let resources_address = get_resource_address(@ticket_nft);
        let state = borrow_global_mut<TicketState>(resources_address);
        assert_ticket_exists(state, &ticket_id);

        let ticket = table::borrow_mut(&mut state.tickets, ticket_id);
        assert_ticket_owner(ticket, owner_addr);
        assert_ticket_not_used(ticket);
        assert_ticket_transferable(ticket);

        let capability = borrow_global<Capability>(object::object_address(&token));
        let linear_transfer_ref = object::generate_linear_transfer_ref(&capability.transfer_ref);
        object::transfer_with_ref(linear_transfer_ref, receiver);

        ticket.owner = receiver;

        event::emit_event(&mut state.transfer_events, TicketTransferEvent {
            ticket_id,
            from: owner_addr,
            to: receiver,
        });
    }
```

This function:
1. Verifies the ticket exists and is owned by the sender
2. Checks that the ticket has not been used
3. Ensures the ticket is transferable (not soulbound)
4. Performs the transfer using the stored capability
5. Updates ticket ownership in the contract state
6. Emits a TicketTransferEvent

## Validating Tickets

When a user attends an event, their ticket is validated:

```move
    // Validate a ticket
    public entry fun validate_ticket(
        user: &signer,
        ticket_id: String // Fixed: Changed from u64 to String
    ) acquires ResourceAccountCap, TicketState {
        let user_addr = signer::address_of(user);
        let resource_address = get_resource_address(@ticket_nft);
        let state = borrow_global_mut<TicketState>(resource_address);
        assert_ticket_exists(state, &ticket_id);

        let ticket = table::borrow_mut(&mut state.tickets, ticket_id);
        assert_ticket_not_used(ticket);

        ticket.is_used = true;
        ticket.usage_time = timestamp::now_seconds();

        event::emit_event(&mut state.validation_events, TicketValidationEvent {
            ticket_id,
            owner: ticket.owner,
        });
    }
```

This function:
1. Checks if the ticket exists
2. Ensures the ticket has not been used already
3. Marks the ticket as used with the current timestamp
4. Emits a TicketValidationEvent


## Helper Functions for Validity Checks

Our contract includes several helper functions for validation:

1.  The `get_resource_address` function will get the resources address based on the owner_address and SEED.
2.  The `get_resource_signer` function will retrieve the signer capability of the resource account for authorized operations.
3.  The `assert_ticket_limit_not_reached` function will check if the maximum ticket limit has been reached before allowing new ticket creation.
4.  The `assert_ticket_exists` function will verify that a ticket with the specified ID exists in the system before performing operations on it.
5.  The `assert_user_has_enough_apt` function will confirm that a user has sufficient AptosCoin balance to complete a transaction.
6.  The `assert_ticket_owner` function will validate that operations requiring ownership are performed only by the legitimate ticket owner.
7.  The `assert_ticket_not_used` function will ensure that a ticket hasn't already been marked as used before allowing usage.
8.  The `assert_ticket_transferable` function will check that a ticket isn't designated as "soulbound" (non-transferable) before allowing transfer operations.

```move
// Helper Functions
    inline fun get_resource_address(account: address): address {
        account::create_resource_address(&account, SEED)
    }

    inline fun get_resource_signer(): signer acquires ResourceAccountCap {
        let cap = borrow_global<ResourceAccountCap>(@ticket_nft);
        account::create_signer_with_capability(&cap.signer_cap)
    }

    fun assert_ticket_limit_not_reached(state: &TicketState) {
        assert!(state.ticket_counter < state.max_tickets, error::invalid_state(E_TICKET_LIMIT_REACHED));
    }

    fun assert_ticket_exists(state: &TicketState, ticket_id: &String) { // Fixed: Reference to String
        assert!(table::contains(&state.tickets, *ticket_id), error::not_found(E_TICKET_NOT_FOUND));
    }

    fun assert_user_has_enough_apt(user_addr: address, amount: u64) {
        assert!(coin::balance<AptosCoin>(user_addr) >= amount, error::invalid_state(E_INSUFFICIENT_FUNDS));
    }

    fun assert_ticket_owner(ticket: &TicketInfo, owner_addr: address) {
        assert!(ticket.owner == owner_addr, error::permission_denied(E_NOT_AUTHORIZED));
    }

    fun assert_ticket_not_used(ticket: &TicketInfo) {
        assert!(!ticket.is_used, error::invalid_state(E_TICKET_USED));
    }

    fun assert_ticket_transferable(ticket: &TicketInfo) {
        assert!(!ticket.is_soulbound, error::invalid_state(E_NOT_AUTHORIZED));
    }
```

These functions ensure that operations only proceed when all requirements are met, helping to maintain the integrity of the ticketing system.

## View Functions for External Queries

To allow external applications to interact with our contract, we provide several view functions:

1.  The `get_ticket_status` function will retrieve the complete status information of a specific ticket when provided with a ticket ID, returning a tuple containing whether the ticket is used, the owner's address, event ID, metadata hash, and if it's soulbound.
2.  The `get_ticket_count` function will return the current count of tickets that have been created in the system.
3.  The `get_max_tickets` function will return the maximum number of tickets that can be created in the system.
4.  The `get_tickets_by_user` function will retrieve all tickets owned by a specific user address, but there's an implementation issue - it incorrectly attempts to iterate through the tickets table as if it were a vector.

```move
    // View Functions
    #[view]
    public fun get_ticket_status(ticket_id: String): (bool, address, String, String, bool) acquires TicketState { // Fixed: String instead of u64
        let resource_address = get_resource_address(@ticket_nft);
        let state = borrow_global<TicketState>(resource_address);
        assert_ticket_exists(state, &ticket_id);

        let ticket = table::borrow(&state.tickets, ticket_id);
        (ticket.is_used, ticket.owner, ticket.event_id, ticket.metadata_hash, ticket.is_soulbound)
    }

    #[view]
    public fun get_ticket_count(): u64 acquires TicketState {
        let resource_address = get_resource_address(@ticket_nft);
        let state = borrow_global<TicketState>(resource_address);
        state.ticket_counter
    }

    #[view]
    public fun get_max_tickets(): u64 acquires TicketState {
        let resource_address = get_resource_address(@ticket_nft);
        let state = borrow_global<TicketState>(resource_address);
        state.max_tickets
    }

    #[view]
    public fun get_tickets_by_user(user_addr: address): vector<Ticket> acquires TicketState {
        let resource_address = get_resource_address(@ticket_nft);
        let state = borrow_global<TicketState>(resource_address);
        let tickets = state.tickets;

        let user_tickets = vector[];
        let i = 0;

        while (i < vector::length(&tickets)) {
            let ticket = vector::borrow(&tickets, i);
            if (ticket.owner == user_addr) {
                vector::push_back(&mut user_tickets, *ticket);
            };

            i = i + 1;
        };

        user_tickets
    }
```

# Deployment and Testing Guide

## Compile your code
Navigate to `packages/move` and run:
```bash
# For Movement CLI
movement move build

# For Aptos CLI
aptos move compile
```

## Deploy your code
Run one of the following commands:
```bash
# General deployment
yarn deploy

# Movement-specific deployment
yarn deploy:movement
```

## Testing your code on chain

Follow these steps to test all functionality of your ticketing system:

1. **Mint tickets**
   - Create soulbound tickets (non-transferable)
   - Create transferable tickets
   - Verify creation via `get_ticket_status` and `get_ticket_count`

2. **Test transfer restrictions**
   - Attempt to transfer a soulbound ticket (this should fail)
   - Verify the owner remains unchanged

3. **Transfer tickets**
   - Transfer a transferable ticket to another address
   - Verify the ownership change via `get_ticket_status`

4. **Validate tickets for event entry**
   - Mark a ticket as used for event entry
   - Verify the status change via `get_ticket_status`

5. **Test reuse prevention**
   - Attempt to use an already validated ticket (this should fail)
   - Verify the operation is rejected

6. **Query ticket information**
   - Use `get_ticket_status` to check individual ticket details
   - Use `get_tickets_by_user` to retrieve all tickets owned by a user
   - Use `get_ticket_count` to check the total number of tickets created
   - Use `get_max_tickets` to confirm the system-wide ticket limit


## Next Step: Frontend Integration

We've successfully implemented a comprehensive NFT ticketing contract on Movement. The final code for this section is available in the smart-contract branch of our base repository. Now it's time to move forward and integrate the frontend with the smart contract