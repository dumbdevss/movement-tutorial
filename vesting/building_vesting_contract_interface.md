# Vesting Contract Tutorial - Implementation Guide

This guide walks through each TODO item in the token vesting system with clear explanations and implementation details. Follow these steps to complete your Move module implementation for a secure, feature-rich vesting contract.

## Source Code Reference

The complete implementation of this smart contract demonstrates advanced Move patterns including resource accounts, dual storage strategies, and comprehensive event handling for token vesting workflows.

## Understanding Key Concepts Before We Start

### What is Token Vesting?

**Token vesting** is a mechanism that releases tokens to beneficiaries over time according to a predetermined schedule. It's commonly used for:

- Employee compensation (releasing stock options over years)
- Investor token allocations (preventing immediate dumps)
- Founder token lockups (building long-term commitment)
- Community rewards (gradual distribution)

### The Cliff Period Explained

A **cliff period** is an initial waiting period where no tokens can be claimed, even if some time has passed. For example:

- 4-year vesting with 1-year cliff means no tokens for first year
- After cliff, tokens vest normally over remaining period
- Protects against short-term participants

### Why Resource Accounts for Vesting?

Resource accounts are perfect for vesting contracts because:

- They hold the actual tokens being vested
- Provide deterministic addresses for fund management
- Enable programmatic token distribution
- Separate contract logic from admin identity
- Allow secure multi-user token custody

### Event-Driven Vesting Architecture

Events are crucial for vesting systems because:

- Beneficiaries need notifications when tokens become available
- Compliance requires audit trails of all token movements
- Analytics track vesting progress across all streams
- Integration with payroll and HR systems
- Tax reporting requires precise timing records

## TODO 1: Define Constants

```move
const SEED: vector<u8> = b"vesting";
const ERROR_NOT_OWNER: u64 = 1;
const ERROR_STREAM_EXISTS: u64 = 2;
const ERROR_STREAM_NOT_FOUND: u64 = 3;
const ERROR_INVALID_DURATION: u64 = 4;
const ERROR_NO_VESTED_TOKENS: u64 = 5;
const ERROR_CLIFF_EXCEEDS_DURATION: u64 = 6;
const ERROR_NOTHING_TO_CLAIM: u64 = 7;
const ERROR_INVALID_AMOUNT: u64 = 8;
const ERROR_CLIFF_HAS_NOT_PASSED: u64 = 9;
const ERROR_INSUFFICIENT_FUNDS: u64 = 10;
const ERROR_INVALID_STREAM_IDS: u64 = 11;
```

**Why These Constants Matter:**

The `SEED` constant ensures our resource account has a **deterministic address** for token custody. This predictability is crucial for:

- Frontend integration knowing where to check balances
- Multi-deployment consistency across networks
- Simplified address management for administrators

**Comprehensive Error Handling:**
The extensive error codes provide specific feedback for different failure scenarios:

- **Ownership errors**: Unauthorized actions
- **Stream management**: Duplicate or missing streams
- **Validation errors**: Invalid parameters
- **Claiming errors**: Insufficient vesting or funds
- **Batch operation errors**: Input validation for multiple streams

**Implementation steps:**

1. Define `SEED` for deterministic resource account creation
2. Create specific error codes for each failure scenario
3. Use descriptive names that clearly indicate the error condition

## TODO 2: Create StreamCreatedEvent Struct

```move
struct StreamCreatedEvent has drop, store {
    beneficiary: address,
    total_amount: u64,
    start_time: u64,
    cliff: u64,
    duration: u64
}
```

**Event Design for Vesting:**
This event captures all essential information for external systems:

- **beneficiary**: Who will receive the tokens
- **total_amount**: Complete allocation (for planning and compliance)
- **start_time**: When vesting begins (crucial for calculations)
- **cliff**: Waiting period (important for beneficiary expectations)
- **duration**: Total vesting timeline (for scheduling and analytics)

**Why This Data Structure:**
External systems (HR platforms, tax software, beneficiary dashboards) need this information to:

- Send notifications to beneficiaries
- Calculate future vesting schedules
- Generate compliance reports
- Plan cash flow and token distribution

**Implementation steps:**

1. Include `#[event]` annotation above the struct (if using newer Move versions)
2. Add `drop` and `store` abilities for event handling
3. Include all critical vesting parameters for external system integration

## TODO 3: Create ClaimCreatedEvent Struct

```move
struct ClaimCreatedEvent has drop, store {
    beneficiary: address,
    amount: u64,
    timestamp: u64
}
```

**Claim Event Purpose:**
This event tracks actual token distributions and is essential for:

- **Tax reporting**: Each claim is a taxable event
- **Compliance**: Regulatory reporting of token movements
- **Analytics**: Understanding claim patterns and timing
- **Notifications**: Informing beneficiaries of successful claims

**Minimal but Complete Design:**
We include only essential claim information:

- **beneficiary**: Who claimed (for accounting)
- **amount**: How much was claimed (for records)
- **timestamp**: When it happened (for compliance and tax purposes)

**Implementation steps:**

1. Create a lightweight event struct for claim tracking
2. Focus on essential information needed for compliance and notifications
3. Ensure timestamp precision for accurate reporting

## TODO 4: Create VestingStream Struct

```move
struct VestingStream has store, copy, drop {
    stream_id: String,
    beneficiary: address,
    total_amount: u64,
    start_time: u64,
    cliff: u64,
    duration: u64,
    claimed_amount: u64,
    creator: address
}
```

**Understanding Move Abilities for Vesting:**
- **store**: Allows storage in global state and other structs
- **copy**: Essential for dual storage strategy (Table + vector)
- **drop**: Allows cleanup when streams are no longer needed

**Complete Vesting Information:**
Each stream contains everything needed for vesting calculations:
- **stream_id**: Unique identifier for management and queries
- **beneficiary**: Token recipient (immutable after creation)
- **total_amount**: Full allocation (basis for all calculations)
- **start_time**: Vesting commencement (timestamp-based)
- **cliff**: Initial waiting period (duration in seconds)
- **duration**: Total vesting period (from start to full vesting)
- **claimed_amount**: Tracking already distributed tokens
- **creator**: Audit trail for who created the stream

**Why Track Creator:**
The creator field enables:
- Audit trails for compliance
- Permission management (only creator can modify)
- Analytics on who creates streams
- Support for multi-admin scenarios

**Implementation steps:**
1. Define struct with appropriate Move abilities
2. Include all fields necessary for vesting calculations
3. Add tracking fields for audit and management purposes
4. Use appropriate data types (u64 for amounts and timestamps, String for IDs)

## TODO 5: Create VestingContract Struct

```move
struct VestingContract has key, copy, drop {
    owner: address,
    streams: SimpleMap<String, VestingStream>,
    streams_vec: vector<VestingStream>
}
```

**The Dual Storage Strategy Explained:**
We use both `SimpleMap` and `vector` for different access patterns:
- **SimpleMap**: O(1) lookup by stream ID - perfect for "get specific stream" operations
- **Vector**: Enables iteration and filtering - essential for "get all streams for user" queries
- **Consistency**: Both must be updated together to prevent data corruption

**Why `key` Ability:**
The `key` ability makes this a **global resource** stored under the resource account, ensuring:
- Single source of truth for all vesting data
- Atomic operations across all streams
- Consistent state management

**Owner vs. Creator Pattern:**
- **owner**: Contract-level administrator (resource account)
- **creator**: Stream-level creator (tracked per stream)
This separation allows flexible administration while maintaining audit trails.

**Implementation steps:**
1. Create struct with `key` ability for global storage
2. Store owner address for contract-level permissions
3. Use both SimpleMap (for fast lookups) and vector (for iteration)
4. Ensure both storage structures contain identical data

## TODO 6: Create State Struct

```move
struct State has key {
    signer_cap: account::SignerCapability,
    stream_created: event::EventHandle<StreamCreatedEvent>,
    claimed: event::EventHandle<ClaimCreatedEvent>
}
```

**SignerCapability for Token Management:**
The `SignerCapability` allows the contract to:
- Make transactions on behalf of the resource account
- Distribute tokens to beneficiaries
- Manage the token treasury securely
- Perform administrative functions programmatically

**Event Handles for Real-Time Updates:**
Each `EventHandle` provides:
- **stream_created**: Notifications when new vesting schedules are established
- **claimed**: Real-time updates when tokens are distributed
- External system integration points
- Analytics and reporting capabilities

**Implementation steps:**
1. Store `SignerCapability` for programmatic account control
2. Create event handles for each event type
3. Ensure proper event handle initialization in `init_module`

## TODO 7: Initialize Module

```move
fun init_module(admin: &signer) {
    let (resource_signer, signer_cap) = account::create_resource_account(admin, SEED);

    // Initialize the vesting contract with resource account as owner
    let streams = simple_map::create();
    let streams_vec = vector::empty<VestingStream>();
    let vesting_contract = VestingContract {
        owner: signer::address_of(&resource_signer),
        streams,
        streams_vec
    };

    // Register the resource account for AptosCoin
    if (!coin::is_account_registered<AptosCoin>(signer::address_of(&resource_signer))) {
        coin::register<AptosCoin>(&resource_signer);
    };

    move_to(&resource_signer, vesting_contract);
    move_to(&resource_signer, State {
        signer_cap,
        stream_created: account::new_event_handle<StreamCreatedEvent>(&resource_signer),
        claimed: account::new_event_handle<ClaimCreatedEvent>(&resource_signer)
    })
}
```

**Module Initialization Strategy:**
`init_module` runs once during deployment and sets up:
- Resource account for token custody
- Empty data structures ready for vesting streams
- Event handles for external system integration
- Coin registration for AptosCoin operations

**Resource Account as Treasury:**
The resource account serves as the vesting contract's treasury:
- Holds all tokens to be vested
- Controlled programmatically via `SignerCapability`
- Provides security through deterministic addressing
- Enables atomic operations across all vesting streams

**Implementation steps:**
1. Create resource account with deterministic address
2. Initialize empty data structures for vesting streams
3. Register resource account for AptosCoin operations
4. Store both contract state and management capabilities
5. Set up event handles for external system integration

## TODO 8: Implement Deposit Function

```move
public entry fun deposit(
    owner: &signer,
    amount: u64
) acquires State {
    let resources_address = account::create_resource_address(&@blockchain, SEED);
    assert!(signer::address_of(owner) == @blockchain, ERROR_NOT_OWNER);
    assert!(amount > 0, ERROR_INVALID_AMOUNT);

    let state = borrow_global<State>(resources_address);
    let resource_signer = account::create_signer_with_capability(&state.signer_cap);

    let coins = coin::withdraw<AptosCoin>(owner, amount);
    coin::deposit<AptosCoin>(signer::address_of(&resource_signer), coins);
}
```

**Treasury Management Pattern:**
The deposit function serves as the contract's funding mechanism:
- **Owner-only access**: Prevents unauthorized funding
- **Validation**: Ensures positive amounts
- **Atomic transfer**: Withdrawal and deposit happen together
- **Resource account custody**: Tokens go directly to contract treasury

**Why Separate Deposit Function:**
Rather than funding during stream creation, separate deposit allows:
- Bulk funding before creating multiple streams
- Better cash flow management
- Clearer audit trails of fund movements
- Flexible funding strategies

**Implementation steps:**
1. Verify caller is authorized contract owner
2. Validate deposit amount is positive
3. Use SignerCapability to control resource account
4. Perform atomic withdrawal from owner and deposit to treasury

## TODO 9: Implement Create Stream Function

```move
public entry fun create_stream(
    owner: &signer,
    user: address,
    total_amount: u64,
    duration: u64,
    cliff: u64,
    stream_id: String
) acquires VestingContract, State {
    let resources_address = account::create_resource_address(&@blockchain, SEED);
    assert!(signer::address_of(owner) == @blockchain, ERROR_NOT_OWNER);
    assert!(!has_stream(stream_id), ERROR_STREAM_EXISTS);
    assert!(duration > 0, ERROR_INVALID_DURATION);
    assert!(total_amount > 0, ERROR_INVALID_AMOUNT);
    assert!(cliff <= duration, ERROR_CLIFF_EXCEEDS_DURATION);
    assert!(coin::balance<AptosCoin>(resources_address) > total_amount, ERROR_INSUFFICIENT_FUNDS);

    let vesting_stream = VestingStream {
        stream_id,
        beneficiary: user,
        total_amount,
        start_time: timestamp::now_seconds(),
        duration,
        cliff,
        claimed_amount: 0,
        creator: signer::address_of(owner)
    };

    let contract = borrow_global_mut<VestingContract>(resources_address);
    let state = borrow_global_mut<State>(resources_address);
    simple_map::add(&mut contract.streams, stream_id, vesting_stream);
    vector::push_back(&mut contract.streams_vec, vesting_stream);

    event::emit_event(&mut state.stream_created, StreamCreatedEvent{
        beneficiary: vesting_stream.beneficiary,
        total_amount: vesting_stream.total_amount,
        start_time: vesting_stream.start_time,
        duration: vesting_stream.duration,
        cliff: vesting_stream.cliff
    })
}
```

**Comprehensive Validation Strategy:**
Before creating streams, we validate:
- **Authorization**: Only owner can create streams
- **Uniqueness**: Stream IDs must be unique
- **Business logic**: Duration and amounts must be positive
- **Logical consistency**: Cliff cannot exceed total duration
- **Financial feasibility**: Sufficient funds must be available

**Immediate Start Time:**
We use `timestamp::now_seconds()` for start time because:
- Provides consistent, blockchain-based timing
- Prevents gaming with future start times
- Simplifies beneficiary expectations
- Enables immediate vesting calculations

**Dual Storage Maintenance:**
Critical to update both storage structures:
1. Add to SimpleMap for O(1) lookups
2. Add to vector for iteration capabilities
3. Maintain consistency between both structures

**Implementation steps:**
1. Perform comprehensive input validation
2. Create VestingStream with current timestamp
3. Update both storage structures atomically
4. Emit event for external system integration
5. Ensure data consistency across all operations

## TODO 10: Implement Create Multiple Streams Function

```move
public entry fun create_multiple_streams(
    owner: &signer,
    users: vector<address>,
    total_amounts: vector<u64>,
    durations: vector<u64>,
    cliffs: vector<u64>,
    stream_ids: vector<String>
) acquires VestingContract, State {
    let resources_address = account::create_resource_address(&@blockchain, SEED);
    assert!(signer::address_of(owner) == @blockchain, ERROR_NOT_OWNER);
    let len = vector::length(&users);
    assert!(len == vector::length(&total_amounts), ERROR_INVALID_AMOUNT);
    assert!(len == vector::length(&durations), ERROR_INVALID_DURATION);
    assert!(len == vector::length(&cliffs), ERROR_INVALID_DURATION);
    assert!(len == vector::length(&stream_ids), ERROR_INVALID_STREAM_IDS);

    // Calculate total required amount
    let total_required = 0;
    let i = 0;
    while (i < len) {
        let amount = *vector::borrow(&total_amounts, i);
        total_required = total_required + amount;
        i = i + 1;
    };
    assert!(coin::balance<AptosCoin>(resources_address) >= total_required, ERROR_INSUFFICIENT_FUNDS);

    let contract = borrow_global_mut<VestingContract>(resources_address);
    let state = borrow_global_mut<State>(resources_address);

    i = 0;
    while (i < len) {
        let user = *vector::borrow(&users, i);
        let total_amount = *vector::borrow(&total_amounts, i);
        let duration = *vector::borrow(&durations, i);
        let cliff = *vector::borrow(&cliffs, i);
        let stream_id = *vector::borrow(&stream_ids, i);

        assert!(!has_stream_multiple(*contract, stream_id), ERROR_STREAM_EXISTS);
        assert!(duration > 0, ERROR_INVALID_DURATION);
        assert!(total_amount > 0, ERROR_INVALID_AMOUNT);
        assert!(cliff <= duration, ERROR_CLIFF_EXCEEDS_DURATION);

        let vesting_stream = VestingStream {
            stream_id,
            beneficiary: user,
            total_amount,
            start_time: timestamp::now_seconds(),
            duration,
            cliff,
            claimed_amount: 0,
            creator: signer::address_of(owner)
        };

        simple_map::add(&mut contract.streams, stream_id, vesting_stream);
        vector::push_back(&mut contract.streams_vec, vesting_stream);

        event::emit_event(&mut state.stream_created, StreamCreatedEvent{
            beneficiary: vesting_stream.beneficiary,
            total_amount: vesting_stream.total_amount,
            start_time: vesting_stream.start_time,
            duration: vesting_stream.duration,
            cliff: vesting_stream.cliff
        });

        i = i + 1;
    }
}
```

**Batch Operations Benefits:**
Creating multiple streams atomically provides:
- **Gas efficiency**: Single transaction for multiple streams
- **Consistency**: All streams start at same time
- **Simplified management**: Bulk operations for large organizations
- **Reduced complexity**: Single approval for multiple vestings

**Pre-Flight Financial Validation:**
We calculate total required funds upfront to:
- Prevent partial execution (all or nothing)
- Provide clear error messages
- Avoid leaving the system in inconsistent states
- Enable proper resource planning

**Input Validation Strategy:**
For batch operations, we validate:
- All input vectors have identical lengths
- Individual stream parameters are valid
- No duplicate stream IDs within the batch
- Sufficient total funding for all streams

**Implementation steps:**
1. Validate all input vectors have equal lengths
2. Calculate and verify total required funding
3. Iterate through inputs with individual validation
4. Create and store each vesting stream
5. Emit events for each stream creation
6. Ensure atomic success or failure

## TODO 11: Implement Get Vested Amount Function

```move
#[view]
public fun get_vested_amount(
    stream_id: String,
    current_time: u64
): u64 acquires VestingContract {
    let resources_address = account::create_resource_address(&@blockchain, SEED);
    assert!(has_stream(stream_id), ERROR_STREAM_NOT_FOUND);
    let contract = borrow_global<VestingContract>(resources_address);

    let stream = simple_map::borrow(&contract.streams, &stream_id);

    // If we're still in cliff period, return 0
    if (current_time < (stream.start_time + stream.cliff)) {
        0
    } else {
        // After cliff period, if no tokens claimed, return total amount
        if (stream.claimed_amount == 0 && current_time >= (stream.start_time + stream.duration)) {
            stream.total_amount
        } else {
            // Otherwise calculate the normal vesting schedule
            calculate_current_vested_without_cliff_amount(
                stream.total_amount,
                stream.start_time,
                stream.duration,
                current_time
            )
        }
    }
}
```

**Vesting Calculation Logic:**
The function handles three scenarios:
1. **Before cliff**: No tokens vested (return 0)
2. **After full duration**: All tokens vested (return total)
3. **During vesting**: Proportional calculation based on elapsed time

**Why Accept Current Time Parameter:**
Rather than always using `now()`, accepting time enables:
- **Testing**: Simulate different time scenarios
- **Queries**: Check vesting at specific future dates
- **Flexibility**: Frontend can show projections
- **Gas efficiency**: Avoid multiple timestamp calls

**Integration with Cliff Period:**
The cliff period creates a "step function" in vesting:
- Zero tokens before cliff ends
- Normal linear vesting after cliff
- Simplified user experience with clear rules

**Implementation steps:**
1. Verify stream exists before calculations
2. Handle cliff period with zero vesting
3. Handle completed vesting with full amount
4. Use helper function for proportional calculations during active vesting
5. Return appropriate vested amount for current time

## TODO 12: Implement Claim Function

```move
public entry fun claim(
    beneficiary: &signer,
    stream_id: String,
    amount_to_claim: u64
) acquires VestingContract, State {
    let resources_address = account::create_resource_address(&@blockchain, SEED);
    let beneficiary_addr = signer::address_of(beneficiary);
    assert!(has_stream(stream_id), ERROR_STREAM_NOT_FOUND);
    let contract = borrow_global_mut<VestingContract>(resources_address);
    let state = borrow_global_mut<State>(resources_address);
    let resource_signer = account::create_signer_with_capability(&state.signer_cap);

    let stream = simple_map::borrow_mut(&mut contract.streams, &stream_id);

    // Find the stream in the vector
    let (found, i) = vector::index_of(&contract.streams_vec, stream);
    assert!(found, ERROR_STREAM_NOT_FOUND);
    let stream_vec = vector::borrow_mut(&mut contract.streams_vec, i);

    let now_seconds = timestamp::now_seconds();

    // Check if cliff period has passed
    assert!(now_seconds >= (stream.cliff + stream.start_time), ERROR_CLIFF_HAS_NOT_PASSED);
    if (now_seconds >= (stream.duration + stream.start_time)) {
        now_seconds = stream.duration + stream.start_time;
    };
    assert!(amount_to_claim > 0, ERROR_INVALID_AMOUNT);

    let current_vested = calculate_current_vested_without_cliff_amount(
        stream.total_amount,
        stream.start_time,
        stream.duration,
        now_seconds
    );

    // Check if we've already claimed all currently vested tokens
    assert!(stream.claimed_amount < current_vested, ERROR_NOTHING_TO_CLAIM);

    // Calculate actual claimable amount (minus what's already been claimed)
    let actual_claimable = current_vested - stream.claimed_amount;
    assert!(actual_claimable > 0, ERROR_NOTHING_TO_CLAIM);
    assert!(amount_to_claim <= actual_claimable, ERROR_INVALID_AMOUNT);

    assert!(coin::balance<AptosCoin>(resources_address) > amount_to_claim, ERROR_INSUFFICIENT_FUNDS);
    let coin_with = coin::withdraw<AptosCoin>(&resource_signer, amount_to_claim);
    coin::deposit<AptosCoin>(beneficiary_addr, coin_with);

    // Update claimed amount in both the map and vector
    stream.claimed_amount = stream.claimed_amount + amount_to_claim;
    stream_vec.claimed_amount = stream_vec.claimed_amount + amount_to_claim;

    event::emit_event(&mut state.claimed, ClaimCreatedEvent{
        beneficiary: beneficiary_addr,
        amount: amount_to_claim,
        timestamp: now_seconds
    })
}
```

**Flexible Claiming Strategy:**
Beneficiaries can claim:
- **Partial amounts**: Don't have to claim everything at once
- **Full available**: Claim all currently vested tokens
- **Multiple times**: Claim as tokens become available over time

**Comprehensive Validation Chain:**
Before token distribution, we verify:
1. Stream exists and beneficiary is authorized
2. Cliff period has passed
3. Requested amount is positive and available
4. Current vesting calculations are accurate
5. Contract has sufficient funds for transfer

**Dual Storage Synchronization:**
Critical to update both storage structures:
- Update SimpleMap entry for fast lookups
- Update vector entry for iteration queries
- Maintain perfect consistency between both

**Time Boundary Handling:**
We cap the time at vesting completion to:
- Prevent over-vesting calculations
- Simplify edge case handling
- Ensure consistent behavior

**Implementation steps:**
1. Verify stream exists and cliff period has passed
2. Calculate current vested amount based on time elapsed
3. Determine actual claimable amount (vested minus already claimed)
4. Validate claim amount against available tokens
5. Perform atomic token transfer using SignerCapability
6. Update claimed amounts in both storage structures
7. Emit claim event for external systems

## TODO 13-17: Implement View and Helper Functions

### TODO 13: Get Stream Details

```move
#[view]
public fun get_stream(
    stream_id: String
): (u64, u64, u64, u64, u64) acquires VestingContract {
    let resources_address = account::create_resource_address(&@blockchain, SEED);
    assert!(has_stream(stream_id), ERROR_STREAM_NOT_FOUND);
    let contract = borrow_global<VestingContract>(resources_address);

    let stream = simple_map::borrow(&contract.streams, &stream_id);
    (
        stream.total_amount,
        stream.start_time,
        stream.cliff,
        stream.duration,
        stream.claimed_amount
    )
}
```

**View Function Design:**
Returns essential stream information as a tuple for:
- Frontend display of vesting schedules
- API integration with external systems
- Beneficiary self-service portals
- Administrative monitoring and reporting

### TODO 14: Get All Streams

```move
#[view]
public fun get_all_streams(): vector<VestingStream> acquires VestingContract {
    let resources_address = account::create_resource_address(&@blockchain, SEED);
    let contract = borrow_global<VestingContract>(resources_address);
    contract.streams_vec
}
```

**Administrative Overview:**
Enables complete contract visibility for:
- Administrative dashboards
- Compliance reporting
- Analytics and insights
- System health monitoring

### TODO 15: Get Streams for User

```move
#[view]
public fun get_streams_for_user(
    user_address: address
): Option<vector<VestingStream>> acquires VestingContract {
    let resources_address = account::create_resource_address(&@blockchain, SEED);

    if (!exists<VestingContract>(resources_address)) {
        return option::none<vector<VestingStream>>()
    };

    let contract = borrow_global<VestingContract>(resources_address);
    let streams_vec = &contract.streams_vec;
    let len = vector::length(streams_vec);
    let result = vector::empty<VestingStream>();

    let i = 0;
    while (i < len) {
        let stream = vector::borrow(streams_vec, i);
        if (stream.beneficiary == user_address) {
            vector::push_back(&mut result, *stream)
        };
        i = i + 1;
    };

    if (vector::length(&result) > 0) {
        option::some(result)
    } else {
        option::none()
    }
}
```

**Beneficiary-Centric Design:**
Filters streams by beneficiary for:
- Personal vesting dashboards
- Mobile app integration
- Self-service claim interfaces
- Individual progress tracking

**Optional Return Pattern:**
Using `Option` type provides:
- Clear indication when no streams exist
- Distinction between "no streams" and "error"
- Better error handling in frontend code
- Idiomatic Move programming style

### TODO 16-17: Helper Functions

```move
inline fun has_stream(stream_id: String): bool acquires VestingContract {
    let resources_address = account::create_resource_address(&@blockchain, SEED);
    let contract = borrow_global<VestingContract>(resources_address);
    simple_map::contains_key(&contract.streams, &stream_id)
}

inline fun has_stream_multiple(
    contract: VestingContract,
    stream_id: String
): bool {
    simple_map::contains_key(&contract.streams, &stream_id)
}

inline fun calculate_current_vested_without_cliff_amount(
    total: u64,
    start: u64,
    duration: u64,
    current_time: u64,
): u64 {
    let end_date = start + duration;
    if (current_time > end_date) {
        total
    } else {
        let current_duration = current_time - start;
        let total = (current_duration * total) / duration;
        total
    }
}
```

**Helper Function Benefits:**
- **Code reuse**: Avoid duplicating logic
- **Consistency**: Same calculations everywhere
- **Testing**: Easier to test individual components
- **Maintenance**: Single place to update logic

**Linear Vesting Calculation:**
The proportional calculation provides:
- **Predictable vesting**: Easy to understand and compute
- **Fair distribution**: Time-based allocation
- **Gas efficiency**: Simple arithmetic operations
- **Standard practice**: Industry-standard approach

## Architectural Design Decisions Explained

### 1. **Resource Account Treasury Pattern**

Using a resource account for token custody provides:
- **Security**: Programmatic control without private keys
- **Predictability**: Deterministic addresses across deployments
- **Scalability**: Single treasury for all vesting streams
- **Auditability**: Clear separation between admin and funds

### 2. **Dual Storage for Vesting Streams**

The SimpleMap + vector combination optimizes for:
- **Individual lookups**: O(1) access for specific streams
- **Filtered queries**: O(n) iteration for user-specific streams
- **Administrative views**: Complete stream enumeration
- **Data consistency**: Synchronized updates across both structures

### 3. **Event-Driven Vesting Architecture**

Events enable reactive vesting systems:
- **Real-time notifications**: Immediate updates to beneficiaries
- **Compliance tracking**: Audit trails for regulatory requirements
- **Analytics integration**: Data pipeline for business intelligence
- **External system hooks**: Integration with payroll and HR systems

### 4. **Flexible Claiming Model**

Partial claiming provides benefits:

- **Cash flow management**: Beneficiaries control timing
- **Tax optimization**: Strategic claim timing for tax efficiency
- **Risk management**: Gradual token realization
- **User experience**: Flexibility improves adoption

### 5. **Comprehensive Validation Strategy**

Multi-layer validation ensures:

- **Data integrity**: Consistent system state
- **Business logic**: Proper vesting rules enforcement
- **Security**: Authorization and access control
- **User experience**: Clear error messages
