# Testing the Ticket NFT Smart Contract

This section outlines a comprehensive testing strategy for the Ticket NFT smart contract written in Move. The tests cover all critical functionalities, including module initialization, soulbound and transferable ticket minting, ticket transfers, ticket validation, and view functions. Each test ensures the contract behaves as expected under both success and failure conditions, adhering to the principles outlined in the provided document management system testing guide.

## 1. Test Environment Setup

The test environment setup initializes the blockchain state, including timestamps, coin systems, and account balances, to simulate a realistic economic environment for ticket purchases.

```move
#[test_only]
fun setup_test(
    admin: &signer,
    user1: &signer,
    user2: &signer,
) {
    // Initialize timestamp system for purchase and usage times
    let aptos_framework = account::create_account_for_test(@aptos_framework);
    timestamp::set_time_has_started_for_testing(&aptos_framework);

    // Initialize APT coin system for payments
    let (burn_cap, mint_cap) = aptos_coin::initialize_for_test(&aptos_framework);
    
    // Fund test accounts with 1000 APT (8 decimal places)
    give_coins(&mint_cap, admin, 1000000000000);
    give_coins(&mint_cap, user1, 1000000000000);
    give_coins(&mint_cap, user2, 1000000000000);

    // Clean up capabilities
    coin::destroy_burn_cap(burn_cap);
    coin::destroy_mint_cap(mint_cap);

    // Initialize the ticketing module
    init_module(admin);
}
```

**Why This Setup?**

- **Timestamps**: Required for tracking ticket purchase and usage times.
- **Coin System**: Ensures users can pay the ticket price (100 APT).
- **Resource Account**: The ticketing system uses a resource account to manage funds, which must be initialized and registered for AptosCoin.
- **Module Initialization**: Sets up the ticket collection and global state, including ticket counters and event handles.

## 2. Testing Module Initialization

**Purpose**: Verify that the ticketing module initializes correctly, creating the collection, resource account, and necessary resources.

```move
#[test(admin = @ticket_nft, user1 = @0x11, user2 = @0x12)]
fun test_init_module(admin: &signer, user1: &signer, user2: &signer) acquires TicketState, ResourceAccountCap {
    setup_test(admin, user1, user2);
    let resource_addr = account::create_resource_address(&@ticket_nft, SEED);
    
    // Verify resource account and structures
    assert!(exists<TicketState>(resource_addr), E_NOT_INITIALIZED);
    assert!(exists<TicketStore>(resource_addr), E_NOT_INITIALIZED);
    assert!(exists<ResourceAccountCap>(@ticket_nft), E_NO_SIGNER_CAP);
    
    // Verify resource account is registered for APT
    assert!(coin::is_account_registered<AptosCoin>(resource_addr), E_NOT_INITIALIZED);
    
    // Verify initial state
    let state = borrow_global<TicketState>(resource_addr);
    assert!(state.ticket_counter == 0, E_INVALID_STATE);
    assert!(state.max_tickets == 1000, E_INVALID_STATE);
}
```

**Why This Matters**: Initialization bugs could prevent ticket minting or fund management, breaking the entire ticketing system.

## 3. Testing Soulbound Ticket Minting

**Purpose**: Ensure soulbound tickets are minted correctly, stored in the `TicketStore`, and have proper initial state.

```move
#[test(admin = @ticket_nft, user1 = @0x11, user2 = @0x12)]
fun test_mint_soulbound_ticket(admin: &signer, user1: &signer, user2: &signer) acquires TicketState, TicketStore, TicketInfo, ResourceAccountCap {
    setup_test(admin, user1, user2);
    let user1_addr = signer::address_of(user1);
    let event_id = 1;
    let metadata_hash = string::utf8(b"ipfs://Qm1");
    let token_name = string::utf8(b"Soulbound Ticket #1");
    let token_uri = string::utf8(b"https://example.com/ticket1");
    
    mint_soulbound_ticket(user1, event_id, metadata_hash, token_name, token_uri);
    
    // Verify ticket count
    let ticket_count = get_ticket_count();
    assert!(ticket_count == 1, E_INVALID_STATE);
    
    // Verify ticket details
    let resource_addr = get_resource_address(@ticket_nft);
    let ticket_store = borrow_global<TicketStore>(resource_addr);
    let ticket_obj = table::borrow(&ticket_store.token_addresses, 0);
    let ticket_addr = object::object_address(ticket_obj);
    let ticket = borrow_global<TicketInfo>(ticket_addr);
    
    assert!(ticket.ticket_id == 0, E_INVALID_DATA);
    assert!(ticket.metadata_hash == metadata_hash, E_INVALID_DATA);
    assert!(ticket.event_id == event_id, E_INVALID_DATA);
    assert!(!ticket.is_used, E_INVALID_STATE);
    assert!(ticket.owner == user1_addr, E_NOT_AUTHORIZED);
    assert!(ticket.is_soulbound, E_INVALID_STATE);
    assert!(ticket.purchase_time == timestamp::now_seconds(), E_INVALID_DATA);
    assert!(ticket.usage_time == 0, E_INVALID_STATE);
}
```

**Why This Matters**: Soulbound tickets are non-transferable and critical for events requiring verified ownership. Errors could lead to invalid tickets or incorrect ownership.

## 4. Testing Transferable Ticket Minting

**Purpose**: Verify that transferable tickets are minted correctly and allow transfers.

```move
#[test(admin = @ticket_nft, user1 = @0x11, user2 = @0x12)]
fun test_mint_transferable_ticket(admin: &signer, user1: &signer, user2: &signer) acquires TicketState, TicketStore, TicketInfo, ResourceAccountCap {
    setup_test(admin, user1, user2);
    let user1_addr = signer::address_of(user1);
    let event_id = 2;
    let metadata_hash = string::utf8(b"ipfs://Qm2");
    let token_name = string::utf8(b"Transferable Ticket #1");
    let token_uri = string::utf8(b"https://example.com/ticket2");
    
    mint_transferable_ticket(user1, event_id, metadata_hash, token_name, token_uri);
    
    // Verify ticket count
    let ticket_count = get_ticket_count();
    assert!(ticket_count == 1, E_INVALID_STATE);
    
    // Verify ticket details
    let resource_addr = get_resource_address(@ticket_nft);
    let ticket_store = borrow_global<TicketStore>(resource_addr);
    let ticket_obj = table::borrow(&ticket_store.token_addresses, 0);
    let ticket_addr = object::object_address(ticket_obj);
    let ticket = borrow_global<TicketInfo>(ticket_addr);
    
    assert!(ticket.ticket_id == 0, E_INVALID_DATA);
    assert!(ticket.metadata_hash == metadata_hash, E_INVALID_DATA);
    assert!(ticket.event_id == event_id, E_INVALID_DATA);
    assert!(!ticket.is_used, E_INVALID_STATE);
    assert!(ticket.owner == user1_addr, E_NOT_AUTHORIZED);
    assert!(!ticket.is_soulbound, E_INVALID_STATE);
    assert!(ticket.purchase_time == timestamp::now_seconds(), E_INVALID_DATA);
    assert!(ticket.usage_time == 0, E_INVALID_STATE);
}
```

**Why This Matters**: Transferable tickets allow secondary markets, so correct minting ensures flexibility while maintaining data integrity.

## 5. Testing Ticket Transfer

### 5.1 Successful Transfer

**Purpose**: Verify that transferable tickets can be transferred to new owners.

```move
#[test(admin = @ticket_nft, user1 = @0x11, user2 = @0x12)]
fun test_transfer_ticket(admin: &signer, user1: &signer, user2: &signer) acquires TicketState, TicketStore, TicketInfo, ResourceAccountCap {
    setup_test(admin, user1, user2);
    let user2_addr = signer::address_of(user2);
    
    mint_transferable_ticket(
        user1,
        1,
        string::utf8(b"ipfs://Qm1"),
        string::utf8(b"Transferable Ticket #1"),
        string::utf8(b"https://example.com/ticket1")
    );
    
    let resource_addr = get_resource_address(@ticket_nft);
    let ticket_store = borrow_global<TicketStore>(resource_addr);
    let ticket_obj = table::borrow(&ticket_store.token_addresses, 0);
    
    transfer_ticket(user1, user2_addr, *ticket_obj);
    
    let (is_used, owner, _, _, is_soulbound) = get_ticket_status(*ticket_obj);
    assert!(owner == user2_addr, E_NOT_AUTHORIZED);
    assert!(!is_used, E_INVALID_STATE);
    assert!(!is_soulbound, E_INVALID_STATE);
}
```

### 5.2 Error Cases for Transfer

**Soulbound Ticket Transfer**:

```move
#[test(admin = @ticket_nft, user1 = @0x11, user2 = @0x12)]
#[expected_failure(abort_code = E_NOT_AUTHORIZED)]
fun test_transfer_soulbound_ticket(admin: &signer, user1: &signer, user2: &signer) acquires TicketState, TicketStore, TicketInfo, ResourceAccountCap {
    setup_test(admin, user1, user2);
    
    mint_soulbound_ticket(
        user1,
        1,
        string::utf8(b"ipfs://Qm1"),
        string::utf8(b"Soulbound Ticket #1"),
        string::utf8(b"https://example.com/ticket1")
    );
    
    let resource_addr = get_resource_address(@ticket_nft);
    let ticket_store = borrow_global<TicketStore>(resource_addr);
    let ticket_obj = table::borrow(&ticket_store.token_addresses, 0);
    
    transfer_ticket(user1, signer::address_of(user2), *ticket_obj);
}
```

**Non-Owner Transfer**:

```move
#[test(admin = @ticket_nft, user1 = @0x11, user2 = @0x12)]
#[expected_failure(abort_code = E_NOT_AUTHORIZED)]
fun test_transfer_non_owner(admin: &signer, user1: &signer, user2: &signer) acquires TicketState, TicketStore, TicketInfo, ResourceAccountCap {
    setup_test(admin, user1, user2);
    
    mint_transferable_ticket(
        user1,
        1,
        string::utf8(b"ipfs://Qm1"),
        string::utf8(b"Transferable Ticket #1"),
        string::utf8(b"https://example.com/ticket1")
    );
    
    let resource_addr = get_resource_address(@ticket_nft);
    let ticket_store = borrow_global<TicketStore>(resource_addr);
    let ticket_obj = table::borrow(&ticket_store.token_addresses, 0);
    
    transfer_ticket(user2, signer::address_of(user1), *ticket_obj);
}
```

**Used Ticket Transfer**:

```move
#[test(admin = @ticket_nft, user1 = @0x11, user2 = @0x12)]
#[expected_failure(abort_code = E_TICKET_USED)]
fun test_transfer_used_ticket(admin: &signer, user1: &signer, user2: &signer) acquires TicketState, TicketStore, TicketInfo, ResourceAccountCap {
    setup_test(admin, user1, user2);
    
    mint_transferable_ticket(
        user1,
        1,
        string::utf8(b"ipfs://Qm1"),
        string::utf8(b"Transferable Ticket #1"),
        string::utf8(b"https://example.com/ticket1")
    );
    
    let resource_addr = get_resource_address(@ticket_nft);
    let ticket_store = borrow_global<TicketStore>(resource_addr);
    let ticket_obj = table::borrow(&ticket_store.token_addresses, 0);
    
    validate_ticket(user1, *ticket_obj);
    transfer_ticket(user1, signer::address_of(user2), *ticket_obj);
}
```

**Why These Tests?**: Transfers are critical for transferable tickets. These tests ensure only valid transfers occur, protecting event integrity and user rights.

## 6. Testing Ticket Validation

**Purpose**: Verify that tickets can be validated by their owners, updating the usage state.

```move
#[test(admin = @ticket_nft, user1 = @0x11, user2 = @0x12)]
fun test_validate_ticket(admin: &signer, user1: &signer, user2: &signer) acquires TicketState, TicketStore, TicketInfo, ResourceAccountCap {
    setup_test(admin, user1, user2);
    
    mint_soulbound_ticket(
        user1,
        1,
        string::utf8(b"ipfs://Qm1"),
        string::utf8(b"Soulbound Ticket #1"),
        string::utf8(b"https://example.com/ticket1")
    );
    
    let resource_addr = get_resource_address(@ticket_nft);
    let ticket_store = borrow_global<TicketStore>(resource_addr);
    let ticket_obj = table::borrow(&ticket_store.token_addresses, 0);
    
    validate_ticket(user1, *ticket_obj);
    
    let (is_used, owner, _, _, _) = get_ticket_status(*ticket_obj);
    assert!(is_used, E_INVALID_STATE);
    assert!(owner == signer::address_of(user1), E_NOT_AUTHORIZED);
}
```

### Error Cases for Validation

**Non-Owner Validation**:

```move
#[test(admin = @ticket_nft, user1 = @0x11, user2 = @0x12)]
#[expected_failure(abort_code = E_NOT_AUTHORIZED)]
fun test_validate_non_owner(admin: &signer, user1: &signer, user2: &signer) acquires TicketState, TicketStore, TicketInfo, ResourceAccountCap {
    setup_test(admin, user1, user2);
    
    mint_soulbound_ticket(
        user1,
        1,
        string::utf8(b"ipfs://Qm1"),
        string::utf8(b"Soulbound Ticket #1"),
        string::utf8(b"https://example.com/ticket1")
    );
    
    let resource_addr = get_resource_address(@ticket_nft);
    let ticket_store = borrow_global<TicketStore>(resource_addr);
    let ticket_obj = table::borrow(&ticket_store.token_addresses, 0);
    
    validate_ticket(user2, *ticket_obj);
}
```

**Already Used Ticket**:

```move
#[test(admin = @ticket_nft, user1 = @0x11, user2 = @0x12)]
#[expected_failure(abort_code = E_TICKET_USED)]
fun test_validate_used_ticket(admin: &signer, user1: &signer, user2: &signer) acquires TicketState, TicketStore, TicketInfo, ResourceAccountCap {
    setup_test(admin, user1, user2);
    
    mint_soulbound_ticket(
        user1,
        1,
        string::utf8(b"ipfs://Qm1"),
        string::utf8(b"Soulbound Ticket #1"),
        string::utf8(b"https://example.com/ticket1")
    );
    
    let resource_addr = get_resource_address(@ticket_nft);
    let ticket_store = borrow_global<TicketStore>(resource_addr);
    let ticket_obj = table::borrow(&ticket_store.token_addresses, 0);
    
    validate_ticket(user1, *ticket_obj);
    validate_ticket(user1, *ticket_obj);
}
```

**Why These Tests?**: Validation ensures tickets are used only once by authorized owners, critical for event access control.

## 7. Testing Insufficient Funds

**Purpose**: Verify that minting fails when a user lacks sufficient APT.

```move
#[test(admin = @ticket_nft, user1 = @0x11, user2 = @0x12)]
#[expected_failure(abort_code = E_INSUFFICIENT_FUNDS)]
fun test_mint_insufficient_funds(admin: &signer, user1: &signer, user2: &signer) acquires TicketState, TicketStore, ResourceAccountCap {
    setup_test(admin, user1, user2);
    
    // Drain user1's funds
    let user_addr = signer::address_of(user1);
    let balance = coin::balance<AptosCoin>(user_addr);
    let coins = coin::withdraw<AptosCoin>(user1, balance);
    coin::deposit(get_resource_address(@ticket_nft), coins);
    
    mint_soulbound_ticket(
        user1,
        1,
        string::utf8(b"ipfs://Qm1"),
        string::utf8(b"Soulbound Ticket #1"),
        string::utf8(b"https://example.com/ticket1")
    );
}
```

**Why This Matters**: Ensures economic security by preventing users from minting tickets without sufficient funds.

## 8. Testing Ticket Limit

**Purpose**: Verify that the contract enforces the maximum ticket limit (1000).

```move
#[test(admin = @ticket_nft, user1 = @0x11, user2 = @0x12)]
#[expected_failure(abort_code = E_TICKET_LIMIT_REACHED)]
fun test_mint_limit_reached(admin: &signer, user1: &signer, user2: &signer) acquires TicketState, TicketStore, ResourceAccountCap {
    setup_test(admin, user1, user2);
    
    // Mint 1000 tickets
    let i = 0;
    while (i < 1000) {
        let token_name = string_utils::format1(&b"Ticket #{}", i);
        let token_uri = string_utils::format1(&b"https://example.com/ticket{}", i);
        mint_transferable_ticket(
            user1,
            1,
            string::utf8(b"ipfs://Qm1"),
            token_name,
            token_uri
        );
        i = i + 1;
    };
    
    // Attempt to mint one more
    mint_transferable_ticket(
        user1,
        1,
        string::utf8(b"ipfs://Qm1"),
        string::utf8(b"Ticket #1000"),
        string::utf8(b"https://example.com/ticket1000")
    );
}
```

**Why This Matters**: The ticket limit prevents oversupply, ensuring event capacity constraints are respected.

## 9. Testing View Functions

**Purpose**: Ensure view functions return accurate data about tickets and user ownership.

```move
#[test(admin = @ticket_nft, user1 = @0x11, user2 = @0x12)]
fun test_view_functions(admin: &signer, user1: &signer, user2: &signer) acquires TicketState, TicketStore, TicketInfo, ResourceAccountCap {
    setup_test(admin, user1, user2);
    let user1_addr = signer::address_of(user1);
    
    // Mint two tickets
    mint_soulbound_ticket(
        user1,
        1,
        string::utf8(b"ipfs://Qm1"),
        string::utf8(b"Soulbound Ticket #1"),
        string::utf8(b"https://example.com/ticket1")
    );
    mint_transferable_ticket(
        user1,
        2,
        string::utf8(b"ipfs://Qm2"),
        string::utf8(b"Transferable Ticket #1"),
        string::utf8(b"https://example.com/ticket2")
    );
    
    // Test get_ticket_count
    let ticket_count = get_ticket_count();
    assert!(ticket_count == 2, E_INVALID_STATE);
    
    // Test get_tickets_by_user
    let user_tickets = get_tickets_by_user(user1_addr);
    assert!(vector::length(&user_tickets) == 2, E_INVALID_STATE);
    
    let ticket1 = vector::borrow(&user_tickets, 0);
    assert!(ticket1.event_id == 1, E_INVALID_DATA);
    assert!(ticket1.is_soulbound, E_INVALID_STATE);
    
    let ticket2 = vector::borrow(&user_tickets, 1);
    assert!(ticket2.event_id == 2, E_INVALID_DATA);
    assert!(!ticket2.is_soulbound, E_INVALID_STATE);
    
    // Test get_ticket_status
    let resource_addr = get_resource_address(@ticket_nft);
    let ticket_store = borrow_global<TicketStore>(resource_addr);
    let ticket_obj = table::borrow(&ticket_store.token_addresses, 0);
    let (is_used, owner, event_id, metadata_hash, is_soulbound) = get_ticket_status(*ticket_obj);
    assert!(!is_used, E_INVALID_STATE);
    assert!(owner == user1_addr, E_NOT_AUTHORIZED);
    assert!(event_id == 1, E_INVALID_DATA);
    assert!(metadata_hash == string::utf8(b"ipfs://Qm1"), E_INVALID_DATA);
    assert!(is_soulbound, E_INVALID_STATE);
}
```

**Non-Existent Ticket Status**:

```move
#[test(admin = @ticket_nft, user1 = @0x11, user2 = @0x12)]
#[expected_failure(abort_code = E_TICKET_NOT_FOUND)]
fun test_get_status_non_existent(admin: &signer, user1: &signer, user2: &signer) acquires TicketInfo, ResourceAccountCap {
    setup_test(admin, user1, user2);
    
    let resource_signer = get_resource_signer();
    let constructor_ref = token::create(
        &resource_signer,
        string::utf8(COLLECTION_NAME),
        string::utf8(b"Soulbound Event Ticket"),
        string::utf8(b"Fake Ticket"),
        option::none(),
        string::utf8(b"https://example.com/fake")
    );
    let fake_token = object::object_from_constructor_ref<Token>(&constructor_ref);
    
    get_ticket_status(fake_token);
}
```

**Why These Tests?**: View functions are essential for front-end integration, providing accurate ticket data for users and event organizers.

## 10. Edge Case Testing

**Purpose**: Test boundary conditions like empty strings for metadata.

```move
#[test(admin = @ticket_nft, user1 = @0x11, user2 = @0x12)]
fun test_mint_empty_strings(admin: &signer, user1: &signer, user2: &signer) acquires TicketState, TicketStore, TicketInfo, ResourceAccountCap {
    setup_test(admin, user1, user2);
    
    mint_soulbound_ticket(
        user1,
        1,
        string::utf8(b""),
        string::utf8(b""),
        string::utf8(b"")
    );
    
    let resource_addr = get_resource_address(@ticket_nft);
    let ticket_store = borrow_global<TicketStore>(resource_addr);
    let ticket_obj = table::borrow(&ticket_store.token_addresses, 0);
    let ticket = borrow_global<TicketInfo>(object::object_address(ticket_obj));
    
    assert!(ticket.metadata_hash == string::utf8(b""), E_INVALID_DATA);
}
```

**Why This Matters**: Users may submit empty inputs, and the contract should handle these gracefully to avoid state corruption.

## 11. Running Tests

To execute the tests, use the Aptos CLI:

```bash
# Run all tests
aptos move test

# Run specific tests
aptos move test --filter test_mint_soulbound_ticket

# Verbose output for debugging
aptos move test --verbose

# Generate coverage report
aptos move test --coverage
```

**Coverage Importance**: High test coverage ensures all code paths are exercised, reducing the risk of untested vulnerabilities.

## 12. Best Practices Summary

- **Comprehensive Setup**: Initializes a realistic blockchain environment with timestamps and funds.
- **Success and Failure Testing**: Uses `#[expected_failure]` to verify error handling for unauthorized actions and invalid states.
- **Descriptive Error Codes**: Uses constants like `E_NOT_AUTHORIZED` and `E_TICKET_USED` for clear error messages.
- **Edge Cases**: Tests empty inputs to ensure robust handling.
- **State Verification**: Checks all state changes after operations (e.g., ticket count, ownership).
- **Isolation**: Ensures tests are independent and deterministic.
- **Clear Naming**: Uses descriptive test names like `test_mint_soulbound_ticket`.
- **Documentation**: Comments explain the purpose of each test scenario.

## 13. Why This Testing Matters

Testing the Ticket NFT contract ensures:

- **Financial Security**: Protects user funds during ticket purchases.
- **Data Integrity**: Maintains accurate ticket ownership and usage status.
- **Event Access Control**: Ensures only authorized owners can validate tickets, preventing fraud.
- **User Trust**: Prevents unauthorized transfers or validations, maintaining confidence.
- **Integration Safety**: Provides reliable view functions for front-end applications.

By thoroughly testing all scenarios, you mitigate the risk of costly bugs in an immutable blockchain environment, safeguarding both users and the event ticketing system's reputation.
