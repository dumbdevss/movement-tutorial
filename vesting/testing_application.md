# Testing the Vesting Smart Contract

This section outlines a comprehensive testing strategy for the `blockchain::vesting` smart contract written in Move. The tests cover all critical functionalities, including module initialization, depositing funds, creating single and multiple vesting streams, claiming vested tokens, and view functions. The approach follows the structure and best practices from the provided NFT Marketplace and Pump For Fun testing guides, ensuring robust validation of success and failure scenarios.

## 1. Test Environment Setup

The test environment initializes the blockchain state, including timestamps and the AptosCoin system, to simulate a realistic environment for vesting operations.

```move
#[test_only]
fun setup_test(
    admin: &signer,
    user1: &signer,
    user2: &signer,
) {
    // Initialize timestamp for vesting schedules
    let aptos_framework = account::create_account_for_test(@aptos_framework);
    timestamp::set_time_has_started_for_testing(&aptos_framework);

    // Initialize APT coin system for funding
    let (burn_cap, mint_cap) = aptos_coin::initialize_for_test(&aptos_framework);
    
    // Fund test accounts with 1 APT (100,000,000 octas)
    give_coins(&mint_cap, admin, 100000000);
    give_coins(&mint_cap, user1, 100000000);
    give_coins(&mint_cap, user2, 100000000);

    // Clean up capabilities
    coin::destroy_burn_cap(burn_cap);
    coin::destroy_mint_cap(mint_cap);

    // Initialize the vesting module
    init_module(admin);
}
```

**Why This Setup?**

- **Timestamps**: Essential for tracking vesting schedules, cliffs, and claim times.
- **Coin System**: Ensures the admin can deposit APT and users can receive vested tokens.
- **Module Initialization**: Sets up the `VestingContract` and `State` resources, including the resource account and event handles.
- **Account Funding**: Provides sufficient APT for deposits and vesting stream creation.

## 2. Testing Module Initialization

**Purpose**: Verify that the module initializes correctly, creating the resource account, registering it for AptosCoin, and setting up the `VestingContract` and `State`.

```move
#[test(admin = @blockchain, user1 = @0x123, user2 = @0x456)]
fun test_init_module(admin: &signer, user1: &signer, user2: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user2);
    let resource_addr = account::create_resource_address(&@blockchain, SEED);
    
    // Verify resource account and structures
    assert!(exists<VestingContract>(resource_addr), 1000);
    assert!(exists<State>(resource_addr), 1001);
    assert!(coin::is_account_registered<AptosCoin>(resource_addr), 1002);
    
    let contract = borrow_global<VestingContract>(resource_addr);
    assert!(contract.owner == resource_addr, 1003);
    assert!(simple_map::length(&contract.streams) == 0, 1004);
    assert!(vector::length(&contract.streams_vec) == 0, 1005);
}
```

**Why This Matters**: Initialization errors could prevent the contract from managing funds or streams, breaking the vesting system.

## 3. Testing Deposit

### 3.1 Successful Deposit

**Purpose**: Ensure the admin can deposit APT into the vesting contract.

```move
#[test(admin = @blockchain, user1 = @0x123, user2 = @0x456)]
fun test_deposit_success(admin: &signer, user1: &signer, user2: &signer) acquires State {
    setup_test(admin, user1, user2);
    let amount = 1000000; // 0.01 APT
    deposit(admin, amount);
    
    let resource_addr = account::create_resource_address(&@blockchain, SEED);
    assert!(coin::balance<AptosCoin>(resource_addr) == amount, 2000);
}
```

### 3.2 Error Cases for Deposit

**Non-Owner Deposit**:

```move
#[test(admin = @blockchain, user1 = @0x123)]
#[expected_failure(abort_code = ERROR_NOT_OWNER)]
fun test_deposit_not_owner(admin: &signer, user1: &signer) acquires State {
    setup_test(admin, user1, user1);
    deposit(user1, 1000000);
}
```

**Zero Amount Deposit**:

```move
#[test(admin = @blockchain, user1 = @0x123)]
#[expected_failure(abort_code = ERROR_INVALID_AMOUNT)]
fun test_deposit_zero_amount(admin: &signer, user1: &signer) acquires State {
    setup_test(admin, user1, user1);
    deposit(admin, 0);
}
```

**Why These Tests?**: Deposits fund the vesting contract. These tests ensure only the admin can deposit and that invalid amounts are rejected.

## 4. Testing Stream Creation

### 4.1 Successful Single Stream Creation

**Purpose**: Verify that a single vesting stream is created correctly with valid parameters.

```move
#[test(admin = @blockchain, user1 = @0x123, user2 = @0x456)]
fun test_create_stream_success(admin: &signer, user1: &signer, user2: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user2);
    deposit(admin, 1000000);
    let stream_id = string::utf8(b"stream1");
    let user_addr = signer::address_of(user1);
    let total_amount = 100000; // 0.001 APT
    let duration = 86400; // 1 day
    let cliff = 43200; // 12 hours
    
    create_stream(admin, user_addr, total_amount, duration, cliff, stream_id);
    
    let (total, start_time, cliff_time, duration_time, claimed) = get_stream(stream_id);
    assert!(total == total_amount, 3000);
    assert!(duration == duration_time, 3001);
    assert!(cliff == cliff_time, 3002);
    assert!(claimed == 0, 3003);
    
    let contract = borrow_global<VestingContract>(account::create_resource_address(&@blockchain, SEED));
    assert!(simple_map::contains_key(&contract.streams, &stream_id), 3004);
    assert!(vector::length(&contract.streams_vec) == 1, 3005);
}
```

### 4.2 Error Cases for Single Stream Creation

**Non-Owner Creation**:

```move
#[test(admin = @blockchain, user1 = @0x123)]
#[expected_failure(abort_code = ERROR_NOT_OWNER)]
fun test_create_stream_not_owner(admin: &signer, user1: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user1);
    deposit(admin, 1000000);
    let stream_id = string::utf8(b"stream1");
    create_stream(user1, signer::address_of(user1), 100000, 86400, 43200, stream_id);
}
```

**Stream Already Exists**:

```move
#[test(admin = @blockchain, user1 = @0x123)]
#[expected_failure(abort_code = ERROR_STREAM_EXISTS)]
fun test_create_stream_already_exists(admin: &signer, user1: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user1);
    deposit(admin, 1000000);
    let stream_id = string::utf8(b"stream1");
    create_stream(admin, signer::address_of(user1), 100000, 86400, 43200, stream_id);
    create_stream(admin, signer::address_of(user1), 100000, 86400, 43200, stream_id);
}
```

**Zero Duration**:

```move
#[test(admin = @blockchain, user1 = @0x123)]
#[expected_failure(abort_code = ERROR_INVALID_DURATION)]
fun test_create_stream_zero_duration(admin: &signer, user1: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user1);
    deposit(admin, 1000000);
    let stream_id = string::utf8(b"stream1");
    create_stream(admin, signer::address_of(user1), 100000, 0, 0, stream_id);
}
```

**Cliff Exceeds Duration**:

```move
#[test(admin = @blockchain, user1 = @0x123)]
#[expected_failure(abort_code = ERROR_CLIFF_EXCEEDS_DURATION)]
fun test_create_stream_cliff_exceeds_duration(admin: &signer, user1: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user1);
    deposit(admin, 1000000);
    let stream_id = string::utf8(b"stream1");
    create_stream(admin, signer::address_of(user1), 100000, 86400, 86401, stream_id);
}
```

**Insufficient Funds**:

```move
#[test(admin = @blockchain, user1 = @0x123)]
#[expected_failure(abort_code = ERROR_INSUFFICIENT_FUNDS)]
fun test_create_stream_insufficient_funds(admin: &signer, user1: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user1);
    let stream_id = string::utf8(b"stream1");
    create_stream(admin, signer::address_of(user1), 100000, 86400, 43200, stream_id);
}
```

**Why These Tests?**: Stream creation is critical for setting up vesting schedules. These tests ensure only the admin can create streams, parameters are validated, and sufficient funds are available.

## 5. Testing Multiple Stream Creation

### 5.1 Successful Multiple Stream Creation

**Purpose**: Verify that multiple vesting streams can be created in a single transaction.

```move
#[test(admin = @blockchain, user1 = @0x123, user2 = @0x456)]
fun test_create_multiple_streams_success(admin: &signer, user1: &signer, user2: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user2);
    deposit(admin, 1000000);
    
    let users = vector::empty<address>();
    vector::push_back(&mut users, signer::address_of(user1));
    vector::push_back(&mut users, signer::address_of(user2));
    
    let amounts = vector::empty<u64>();
    vector::push_back(&mut amounts, 100000);
    vector::push_back(&mut amounts, 200000);
    
    let durations = vector::empty<u64>();
    vector::push_back(&mut durations, 86400);
    vector::push_back(&mut durations, 86400);
    
    let cliffs = vector::empty<u64>();
    vector::push_back(&mut cliffs, 43200);
    vector::push_back(&mut cliffs, 43200);
    
    let stream_ids = vector::empty<string::String>();
    vector::push_back(&mut stream_ids, string::utf8(b"stream1"));
    vector::push_back(&mut stream_ids, string::utf8(b"stream2"));
    
    create_multiple_streams(admin, users, amounts, durations, cliffs, stream_ids);
    
    let streams = get_all_streams();
    assert!(vector::length(&streams) == 2, 4000);
    
    let (total1, _, _, _, claimed1) = get_stream(string::utf8(b"stream1"));
    let (total2, _, _, _, claimed2) = get_stream(string::utf8(b"stream2"));
    assert!(total1 == 100000, 4001);
    assert!(total2 == 200000, 4002);
    assert!(claimed1 == 0, 4003);
    assert!(claimed2 == 0, 4004);
}
```

### 5.2 Error Case for Multiple Stream Creation

**Mismatched Input Vectors**:

```move
#[test(admin = @blockchain, user1 = @0x123, user2 = @0x456)]
#[expected_failure(abort_code = ERROR_INVALID_STREAM_IDS)]
fun test_create_multiple_streams_mismatch(admin: &signer, user1: &signer, user2: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user2);
    deposit(admin, 1000000);
    
    let users = vector::empty<address>();
    vector::push_back(&mut users, signer::address_of(user1));
    
    let amounts = vector::empty<u64>();
    vector::push_back(&mut amounts, 100000);
    
    let durations = vector::empty<u64>();
    vector::push_back(&mut durations, 86400);
    
    let cliffs = vector::empty<u64>();
    vector::push_back(&mut cliffs, 43200);
    
    let stream_ids = vector::empty<string::String>();
    create_multiple_streams(admin, users, amounts, durations, cliffs, stream_ids);
}
```

**Why These Tests?**: Batch stream creation improves efficiency. These tests ensure correct handling of multiple streams and validation of input consistency.

## 6. Testing Claiming Vested Tokens

### 6.1 Successful Claim

**Purpose**: Verify that beneficiaries can claim vested tokens after the cliff period.

```move
#[test(admin = @blockchain, user1 = @0x123, user2 = @0x456)]
fun test_claim_success(admin: &signer, user1: &signer, user2: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user2);
    deposit(admin, 1000000);
    let stream_id = string::utf8(b"stream1");
    let user_addr = signer::address_of(user1);
    create_stream(admin, user_addr, 100000, 86400, 0, stream_id);
    
    // Fast forward to end of vesting period
    timestamp::fast_forward_seconds(86400);
    
    let initial_balance = coin::balance<AptosCoin>(user_addr);
    claim(user1, stream_id, 100000);
    
    let final_balance = coin::balance<AptosCoin>(user_addr);
    assert!(final_balance == initial_balance + 100000, 5000);
    
    let (_, _, _, _, claimed_amount) = get_stream(stream_id);
    assert!(claimed_amount == 100000, 5001);
}
```

### 6.2 Error Cases for Claiming

**Non-Existent Stream**:

```move
#[test(admin = @blockchain, user1 = @0x123)]
#[expected_failure(abort_code = ERROR_STREAM_NOT_FOUND)]
fun test_claim_nonexistent_stream(admin: &signer, user1: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user1);
    deposit(admin, 1000000);
    claim(user1, string::utf8(b"stream1"), 100000);
}
```

**Claim Before Cliff**:

```move
#[test(admin = @blockchain, user1 = @0x123)]
#[expected_failure(abort_code = ERROR_CLIFF_HAS_NOT_PASSED)]
fun test_claim_before_cliff(admin: &signer, user1: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user1);
    deposit(admin, 1000000);
    let stream_id = string::utf8(b"stream1");
    create_stream(admin, signer::address_of(user1), 100000, 86400, 43200, stream_id);
    claim(user1, stream_id, 100000);
}
```

**Nothing to Claim**:

```move
#[test(admin = @blockchain, user1 = @0x123)]
#[expected_failure(abort_code = ERROR_NOTHING_TO_CLAIM)]
fun test_claim_nothing_to_claim(admin: &signer, user1: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user1);
    deposit(admin, 1000000);
    let stream_id = string::utf8(b"stream1");
    create_stream(admin, signer::address_of(user1), 100000, 86400, 0, stream_id);
    timestamp::fast_forward_seconds(86400);
    claim(user1, stream_id, 100000);
    claim(user1, stream_id, 1);
}
```

**Invalid Claim Amount**:

```move
#[test(admin = @blockchain, user1 = @0x123)]
#[expected_failure(abort_code = ERROR_INVALID_AMOUNT)]
fun test_claim_excessive_amount(admin: &signer, user1: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user1);
    deposit(admin, 1000000);
    let stream_id = string::utf8(b"stream1");
    create_stream(admin, signer::address_of(user1), 100000, 86400, 0, stream_id);
    timestamp::fast_forward_seconds(43200);
    claim(user1, stream_id, 100001);
}
```

**Why These Tests?**: Claiming is the primary user interaction. These tests ensure tokens are released correctly, only after the cliff, and within vested limits.

## 7. Testing Vested Amount Calculation

**Purpose**: Verify the `get_vested_amount` function calculates the correct vested amount based on time.

```move
#[test(admin = @blockchain, user1 = @0x123, user2 = @0x456)]
fun test_get_vested_amount(admin: &signer, user1: &signer, user2: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user2);
    deposit(admin, 1000000);
    let stream_id = string::utf8(b"stream1");
    let start_time = timestamp::now_seconds();
    create_stream(admin, signer::address_of(user1), 100000, 86400, 43200, stream_id);
    
    // Before cliff
    let vested = get_vested_amount(stream_id, start_time);
    assert!(vested == 0, 6000);
    
    // Halfway through vesting (after cliff)
    let halfway_time = start_time + 64800; // 43200 (cliff) + 21600 (half of remaining)
    timestamp::fast_forward_seconds(64800);
    vested = get_vested_amount(stream_id, halfway_time);
    assert!(vested >= 49999 && vested <= 50001, 6001); // ~50% vested
    
    // After full vesting
    timestamp::fast_forward_seconds(86400);
    vested = get_vested_amount(stream_id, start_time + 86400);
    assert!(vested == 100000, 6002);
}
```

**Why This Matters**: Accurate vesting calculations ensure beneficiaries receive the correct amount of tokens based on the vesting schedule.

## 8. Testing View Functions

### 8.1 Get Stream Details

**Purpose**: Verify that `get_stream` returns accurate stream data.

```move
#[test(admin = @blockchain, user1 = @0x123, user2 = @0x456)]
fun test_get_stream(admin: &signer, user1: &signer, user2: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user2);
    deposit(admin, 1000000);
    let stream_id = string::utf8(b"stream1");
    let total_amount = 100000;
    let duration = 86400;
    let cliff = 43200;
    create_stream(admin, signer::address_of(user1), total_amount, duration, cliff, stream_id);
    
    let (total, start_time, cliff_time, duration_time, claimed) = get_stream(stream_id);
    assert!(total == total_amount, 7000);
    assert!(duration == duration_time, 7001);
    assert!(cliff == cliff_time, 7002);
    assert!(claimed == 0, 7003);
}
```

### 8.2 Get All Streams

**Purpose**: Verify that `get_all_streams` returns all vesting streams.

```move
#[test(admin = @blockchain, user1 = @0x123, user2 = @0x456)]
fun test_get_all_streams(admin: &signer, user1: &signer, user2: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user2);
    deposit(admin, 1000000);
    let stream_id1 = string::utf8(b"stream1");
    let stream_id2 = string::utf8(b"stream2");
    create_stream(admin, signer::address_of(user1), 100000, 86400, 43200, stream_id1);
    create_stream(admin, signer::address_of(user2), 200000, 86400, 43200, stream_id2);
    
    let streams = get_all_streams();
    assert!(vector::length(&streams) == 2, 8000);
    let stream1 = vector::borrow(&streams, 0);
    let stream2 = vector::borrow(&streams, 1);
    assert!(stream1.total_amount == 100000, 8001);
    assert!(stream2.total_amount == 200000, 8002);
}
```

### 8.3 Get Streams for User

**Purpose**: Verify that `get_streams_for_user` returns the correct streams for a beneficiary.

```move
#[test(admin = @blockchain, user1 = @0x123, user2 = @0x456)]
fun test_get_streams_for_user(admin: &signer, user1: &signer, user2: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user2);
    deposit(admin, 1000000);
    let stream_id1 = string::utf8(b"stream1");
    let stream_id2 = string::utf8(b"stream2");
    create_stream(admin, signer::address_of(user1), 100000, 86400, 43200, stream_id1);
    create_stream(admin, signer::address_of(user1), 200000, 86400, 43200, stream_id2);
    
    let streams = get_streams_for_user(signer::address_of(user1));
    assert!(option::is_some(&streams), 9000);
    let streams_vec = option::extract(&mut streams);
    assert!(vector::length(&streams_vec) == 2, 9001);
    
    let no_streams = get_streams_for_user(signer::address_of(user2));
    assert!(option::is_none(&no_streams), 9002);
}
```

**Why These Tests?**: View functions are essential for front-end integration, providing accurate stream data for users and administrators.

## 9. Edge Case Testing

**Purpose**: Test boundary conditions like empty stream IDs.

```move
#[test(admin = @blockchain, user1 = @0x123, user2 = @0x456)]
fun test_create_stream_empty_id(admin: &signer, user1: &signer, user2: &signer) acquires VestingContract, State {
    setup_test(admin, user1, user2);
    deposit(admin, 1000000);
    let stream_id = string::utf8(b"");
    create_stream(admin, signer::address_of(user1), 100000, 86400, 43200, stream_id);
    
    let (total, _, _, _, _) = get_stream(stream_id);
    assert!(total == 100000, 10000);
}
```

**Why This Matters**: Users may submit empty inputs, and the contract should handle these gracefully to avoid state corruption.

## 10. Running Tests

To execute the tests, use the Aptos CLI:

```bash
# Run all tests
aptos move test

# Run specific tests
aptos move test --filter test_create_stream_success

# Verbose output for debugging
aptos move test --verbose

# Generate coverage report
aptos move test --coverage
```

**Coverage Importance**: High test coverage ensures all code paths, including edge cases and error conditions, are tested, reducing the risk of vulnerabilities.

## 11. Best Practices Summary

- **Comprehensive Setup**: Initializes timestamps, coins, and module state for realistic testing.
- **Success and Failure Testing**: Uses `#[expected_failure]` to verify error handling for invalid inputs and unauthorized actions.
- **Descriptive Error Codes**: Employs constants like `ERROR_NOT_OWNER` and `ERROR_NOTHING_TO_CLAIM` for clear error reporting.
- **Edge Cases**: Tests empty stream IDs to ensure robust input handling.
- **State Verification**: Checks contract state, balances, and event emissions after operations.
- **Isolation**: Ensures tests are independent and deterministic.
- **Clear Naming**: Uses descriptive test names like `test_claim_success`.
- **Documentation**: Comments explain the purpose of each test scenario.

## 12. Why This Testing Matters

Testing the vesting contract ensures:

- **Financial Security**: Protects funds by ensuring only the admin can deposit and create streams.
- **Vesting Accuracy**: Maintains correct vesting schedules and claimable amounts.
- **Access Control**: Ensures only authorized users (admin for deposits/streams, beneficiaries for claims) can interact.
- **User Trust**: Prevents unauthorized claims and ensures accurate token distribution.
- **Integration Safety**: Provides reliable view functions for front-end applications.

By thoroughly testing all scenarios, you mitigate the risk of costly bugs in an immutable blockchain environment, safeguarding users and the vesting system's integrity.
