# Move Smart Contract Testing Tutorial: Document Management System

## 1. Introduction to Move Testing Framework {#introduction}

The Move programming language provides a robust testing framework that allows developers to write comprehensive unit tests for their smart contracts. This tutorial will walk you through testing the document management smart contract, covering everything from basic test setup to advanced testing patterns.

### Why Testing Matters in Blockchain Development

Testing smart contracts is absolutely critical because:

- **Immutability**: Once deployed on the blockchain, smart contracts cannot be easily modified or patched like traditional software. Bugs become permanent unless you deploy entirely new contracts
- **Financial Security**: Smart contracts often handle valuable assets. A single bug can lead to permanent loss of funds or exploitation by malicious actors
- **User Trust**: Well-tested contracts inspire confidence in users who are entrusting their assets and data to your system
- **Compliance**: Ensures the contract behaves exactly as specified in your business requirements

**Real-world Impact**: Many DeFi protocols have lost millions of dollars due to inadequately tested smart contracts. The DAO hack of 2016 ($60M loss) and numerous DeFi exploits could have been prevented with comprehensive testing.

## 2. Testing Framework Concepts {#concepts}

### 2.1 Test Annotations

Move uses several key annotations for testing. Understanding when and why to use each is crucial:

```move
#[test]                    // Marks a function as a test
#[test_only]              // Code only compiled during testing
#[expected_failure]       // Test expects the function to abort
#[expected_failure(abort_code = X)] // Expects specific abort code
```

**Why `#[test_only]`?** This annotation is essential for helper functions and test-specific code that shouldn't be included in the production contract. It keeps your deployed contract lean and prevents test code from accidentally being called in production.

**Why `#[expected_failure]`?** Security testing requires verifying that your contract properly rejects invalid operations. This annotation lets you test that your error handling works correctly - for example, ensuring unauthorized users can't access restricted functions.

### 2.2 Test Function Signatures

Test functions follow specific patterns based on what you're testing:

```move
// Basic test - for pure logic without blockchain state
#[test]
fun test_function_name() {
    // Test logic here
}

// Test with signers (accounts) - for testing user interactions
#[test(account1 = @0x123, account2 = @0x456)]
fun test_with_accounts(account1: &signer, account2: &signer) {
    // Test logic with accounts
}
```

**Why Signers?** In blockchain systems, every operation is performed by an account (represented by a signer). Testing with multiple signers lets you simulate real-world scenarios where different users interact with your contract. This is crucial for testing access control, ownership, and multi-user workflows.

### 2.3 Test Assertions and Error Handling

Move provides assertion functions for testing:

```move
assert!(condition, error_code);  // Basic assertion
```

**Best Practice**: Always use meaningful error codes in your assertions. Instead of generic numbers, define constants that make test failures immediately understandable:

```move
const E_NOT_AUTHORIZED: u64 = 1;
const E_DOCUMENT_NOT_FOUND: u64 = 2;
```

## 3. Test Environment Setup {#setup}

### Understanding the Aptos Testing Environment

Before diving into our setup function, it's important to understand what we're initializing and why:

**Timestamp System**: Blockchain applications often need to track time for features like expiration dates, cooldowns, or time-based access control. The timestamp system must be explicitly initialized in tests.

**Coin System**: Our document management system charges fees for uploads. We need to set up the APT (Aptos Token) system and provide test accounts with funds to simulate real transactions.

**Resource Accounts**: These are special accounts controlled by smart contracts rather than private keys. They're essential for contracts that need to hold and manage assets independently.

### Step 1: Understanding the Setup Function

Our contract uses a comprehensive setup function that initializes the testing environment:

```move
#[test(admin = @document_management, user1 = @0x456, user2 = @0x789)]
fun setup_test(
    admin: &signer,
    user1: &signer,
    user2: &signer,
) {
    // 1. Initialize timestamp system
    // Why: Many blockchain apps need time-based logic (expiration, cooldowns)
    let aptos_framework = account::create_account_for_test(@aptos_framework);
    timestamp::set_time_has_started_for_testing(&aptos_framework);

    // 2. Initialize APT coin system
    // Why: Our contract charges fees, so we need a working payment system
    let (burn_cap, mint_cap) = aptos_coin::initialize_for_test(&aptos_framework);
    
    // 3. Give coins to test accounts
    // Why: Users need funds to pay for document uploads and transactions
    give_coins(&mint_cap, admin, 100000000);  // 1 APT (8 decimal places)
    give_coins(&mint_cap, user1, 100000000);
    give_coins(&mint_cap, user2, 100000000);

    // 4. Clean up capabilities
    // Why: These are powerful permissions that shouldn't persist after setup
    coin::destroy_burn_cap(burn_cap);
    coin::destroy_mint_cap(mint_cap);

    // 5. Initialize the contract
    // Why: Sets up our document management system's initial state
    init_module(admin);
}
```

**Why This Setup is Necessary**: Unlike traditional software testing where you might just create objects, blockchain testing requires simulating an entire economic environment. Every interaction costs gas, users need balances, and the system needs to track time and state changes across a distributed network.

### Step 2: Helper Functions and Resource Accounts

The `give_coins` function demonstrates how to set up test accounts:

```move
#[test_only]
fun give_coins(mint_cap: &MintCapability<AptosCoin>, to: &signer, amount: u64) {
    let to_addr = signer::address_of(to);
    
    // Create account if it doesn't exist
    // Why: In testing, accounts don't exist until explicitly created
    if (!account::exists_at(to_addr)) {
        account::create_account_for_test(to_addr);
    };
    
    // Register for APT coin
    // Why: Accounts must explicitly register to receive each coin type
    coin::register<AptosCoin>(to);

    // Mint and deposit coins
    // Why: We're simulating the economic conditions users will face
    let coins = coin::mint(amount, mint_cap);
    coin::deposit(to_addr, coins);
}
```

**Resource Accounts Explained**: Our document management system uses a resource account to collect fees and manage contract-owned assets. Unlike regular accounts controlled by private keys, resource accounts are controlled by the smart contract code itself. This is essential for:

- Collecting and managing fees automatically
- Holding assets that belong to the protocol, not individual users
- Enabling complex financial operations like staking, lending, or escrow
- Manage resources like a global state struct

## 4. Step-by-Step Testing Guide {#testing-guide}

### Step 1: Testing Module Initialization

**Purpose**: Verify that the contract initializes correctly and sets up all necessary resources.

```move
#[test(admin = @document_management, user1 = @0x456, user2 = @0x789)]
fun test_init_module(
    admin: &signer,
    user1: &signer,
    user2: &signer
) acquires DocState, ResourceAccountCap {
    // Setup the test environment
    setup_test(admin, user1, user2);

    // Verify resource account was created
    // Why test this? Resource accounts are critical for fee collection
    let resource_addr = get_resource_address();
    assert!(account::exists_at(resource_addr), 0);
    
    // Verify APT registration
    // Why test this? The contract must be able to receive payment
    assert!(coin::is_account_registered<AptosCoin>(resource_addr), 1);
    
    // Verify initial document count is zero
    // Why test this? Ensures clean state and proper initialization
    let doc_count = get_document_count();
    assert!(doc_count == 0, 2);
}
```

**Why This Test Matters**: Initialization failures are among the most catastrophic bugs in smart contracts. If the contract can't properly set up its resource account or register for payments, the entire system becomes unusable after deployment.

### Step 2: Testing Document Upload

#### 2.1 Successful Upload Test

```move
#[test(admin = @document_management, user1 = @0x456, user2 = @0x789)]
fun test_upload_document(
    admin: &signer,
    user1: &signer,
    user2: &signer
) acquires DocState, ResourceAccountCap {
    setup_test(admin, user1, user2);
    let user1_addr = signer::address_of(user1);

    // Prepare test data
    // Why these specific values? Testing realistic but distinct data
    let doc_id = string::utf8(b"doc1");
    let doc_name = string::utf8(b"Test Document");
    let ipfs_hash = string::utf8(b"QmTestHash"); // IPFS hashes start with "Qm"
    
    // Execute the function being tested
    upload_document(user1, doc_name, ipfs_hash, doc_id);

    // Verify document properties
    // Why check each field? Ensures data integrity and proper storage
    let (name, hash, _, owner, signatures, allowed_signers) = get_document(doc_id);
    assert!(name == doc_name, 0);
    assert!(hash == ipfs_hash, 1);
    assert!(owner == user1_addr, 3);
    assert!(vector::length(&signatures) == 0, 4); // New docs have no signatures
    assert!(vector::length(&allowed_signers) == 1, 5); // Owner is automatically allowed
    assert!(vector::contains(&allowed_signers, &user1_addr), 6);

    // Verify global state changes
    // Why test document count? Ensures proper state management
    let doc_count = get_document_count();
    assert!(doc_count == 1, 7);

    // Verify payment was processed
    // Why test this? Fee collection is a core business requirement
    let upload_fee = get_upload_fee();
    assert!(coin::balance<AptosCoin>(user1_addr) == 100000000 - upload_fee, 8);
}
```

**Why We Use Vectors vs Tables**:

- **Vector for `signatures` and `allowed_signers`**: These are small collections that we iterate through frequently (checking permissions, displaying signatures). Vectors are optimal for iteration and small datasets.
- **Table for document storage**: Documents are accessed by ID (O(1) lookup) and can grow large. Tables provide efficient key-based access without the O(n) cost of vector searches.

#### 2.2 Testing Failure Cases

Understanding failure scenarios is crucial for security:

**Duplicate Document Test**:

```move
#[test(admin = @document_management, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_DOCUMENT_ALREADY_EXISTS)]
fun test_upload_duplicate_document(
    admin: &signer,
    user1: &signer,
    user2: &signer
) acquires DocState, ResourceAccountCap {
    setup_test(admin, user1, user2);

    let doc_id = string::utf8(b"doc1");
    let doc_name = string::utf8(b"Test Document");
    let ipfs_hash = string::utf8(b"QmTestHash");
    
    // First upload should succeed
    upload_document(user1, doc_name, ipfs_hash, doc_id);

    // Second upload should fail
    // Why test this? Prevents data corruption and ensures ID uniqueness
    upload_document(user1, doc_name, ipfs_hash, doc_id);
}
```

**Why This Matters**: Without proper duplicate checking, users could accidentally overwrite important documents or malicious actors could replace legitimate documents with fraudulent ones.

**Insufficient Funds Test**:

```move
#[test(admin = @document_management, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_INSUFFICIENT_FUNDS)]
fun test_upload_insufficient_funds(
    admin: &signer,
    user1: &signer,
    user2: &signer
) acquires DocState, ResourceAccountCap {
    setup_test(admin, user1, user2);
    let user1_addr = signer::address_of(user1);

    // Remove all coins from user1
    // Why drain the account? Tests the exact boundary condition
    let balance = coin::balance<AptosCoin>(user1_addr);
    let withdrawn = coin::withdraw<AptosCoin>(user1, balance);
    coin::deposit<AptosCoin>(@document_management, withdrawn);

    // Attempt upload with no funds
    // Why test this? Ensures the payment system works correctly
    let doc_id = string::utf8(b"doc1");
    let doc_name = string::utf8(b"Test Document");
    let ipfs_hash = string::utf8(b"QmTestHash");
    upload_document(user1, doc_name, ipfs_hash, doc_id);
}
```

**Economic Security**: This test ensures users can't bypass payment requirements, which is critical for the protocol's sustainability and prevents spam attacks.

### Step 3: Testing Signer Management

#### 3.1 Adding Signers

```move
#[test(admin = @document_management, user1 = @0x456, user2 = @0x789)]
fun test_add_signer(
    admin: &signer,
    user1: &signer,
    user2: &signer
) acquires DocState, ResourceAccountCap {
    setup_test(admin, user1, user2);
    let user2_addr = signer::address_of(user2);

    // Upload document
    let doc_id = string::utf8(b"doc1");
    let doc_name = string::utf8(b"Test Document");
    let ipfs_hash = string::utf8(b"QmTestHash");
    upload_document(user1, doc_name, ipfs_hash, doc_id);

    // Add signer
    // Why test signer management? Multi-party signing is a core feature
    add_signer(user1, doc_id, user2_addr);

    // Verify signer was added
    let (_, _, _, _, _, allowed_signers) = get_document(doc_id);
    assert!(vector::length(&allowed_signers) == 2, 0); // Owner + new signer
    assert!(vector::contains(&allowed_signers, &user2_addr), 1);
}
```

**Why Signer Management Matters**: Document signing often requires multiple parties (contracts, approvals, witness signatures). Proper access control ensures only authorized parties can sign while maintaining an audit trail.

#### 3.2 Testing Authorization

```move
#[test(admin = @document_management, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_NOT_AUTHORIZED)]
fun test_add_signer_unauthorized(
    admin: &signer,
    user1: &signer,
    user2: &signer
) acquires DocState, ResourceAccountCap {
    setup_test(admin, user1, user2);
    let user2_addr = signer::address_of(user2);

    // Upload document as user1
    let doc_id = string::utf8(b"doc1");
    upload_document(user1, string::utf8(b"Test"), string::utf8(b"Hash"), doc_id);

    // Try to add signer as user2 (not owner) - should fail
    // Why test this? Prevents unauthorized users from manipulating access control
    add_signer(user2, doc_id, user2_addr);
}
```

**Security Principle**: This follows the principle of least privilege - only document owners should control who can sign their documents. Testing this prevents privilege escalation attacks.

### Step 4: Testing Document Signing

#### 4.1 Successful Signing

```move
#[test(admin = @document_management, user1 = @0x456, user2 = @0x789)]
fun test_sign_document(
    admin: &signer,
    user1: &signer,
    user2: &signer
) acquires DocState, ResourceAccountCap {
    setup_test(admin, user1, user2);
    let user2_addr = signer::address_of(user2);

    // Setup document and add signer
    let doc_id = string::utf8(b"doc1");
    upload_document(user1, string::utf8(b"Test"), string::utf8(b"Hash"), doc_id);
    add_signer(user1, doc_id, user2_addr);

    // Sign the document
    // Why test signing? It's the core value proposition of the system
    sign_document(user2, doc_id);

    // Verify signature
    let (_, _, _, _, signatures, _) = get_document(doc_id);
    assert!(vector::length(&signatures) == 1, 0);
    assert!(vector::contains(&signatures, &user2_addr), 1);
    assert!(is_signed_by(doc_id, user2_addr), 2);
}
```

**Why Events Matter**: When users sign documents, the system should emit events for:

- **Transparency**: Off-chain systems can track signing progress
- **Notifications**: Users can be alerted when documents are signed
- **Audit Trails**: Legal compliance often requires immutable signing records
- **Integration**: Other contracts or dApps can react to signing events

#### 4.2 Testing Signing Restrictions

**Unauthorized Signing**:

```move
#[test(admin = @document_management, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_NOT_ALLOWED_TO_SIGN)]
fun test_sign_document_not_allowed(
    admin: &signer,
    user1: &signer,
    user2: &signer
) acquires DocState, ResourceAccountCap {
    setup_test(admin, user1, user2);

    // Upload document (user2 not added as signer)
    let doc_id = string::utf8(b"doc1");
    upload_document(user1, string::utf8(b"Test"), string::utf8(b"Hash"), doc_id);

    // Attempt to sign without permission - should fail
    // Why test this? Prevents unauthorized signatures that could be legally binding
    sign_document(user2, doc_id);
}
```

**Legal and Security Implications**: Unauthorized signatures could create false legal agreements or be used in fraud. This test ensures only explicitly authorized parties can sign.

**Double Signing Prevention**:

```move
#[test(admin = @document_management, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_ALREADY_SIGNED)]
fun test_sign_document_already_signed(
    admin: &signer,
    user1: &signer,
    user2: &signer
) acquires DocState, ResourceAccountCap {
    setup_test(admin, user1, user2);
    let user2_addr = signer::address_of(user2);

    // Setup and first signature
    let doc_id = string::utf8(b"doc1");
    upload_document(user1, string::utf8(b"Test"), string::utf8(b"Hash"), doc_id);
    add_signer(user1, doc_id, user2_addr);
    sign_document(user2, doc_id);

    // Attempt to sign again - should fail
    // Why prevent double signing? Maintains signature integrity and prevents confusion
    sign_document(user2, doc_id);
}
```

**Data Integrity**: Double signing could indicate a replay attack or system malfunction. Prevention ensures each signature represents a single, intentional action.

### Step 5: Testing View Functions

#### 5.1 Document Retrieval

```move
#[test(admin = @document_management, user1 = @0x456, user2 = @0x789)]
fun test_get_documents_for_signer(
    admin: &signer,
    user1: &signer,
    user2: &signer
) acquires DocState, ResourceAccountCap {
    setup_test(admin, user1, user2);
    let user2_addr = signer::address_of(user2);

    // Upload multiple documents
    let doc_id1 = string::utf8(b"doc1");
    let doc_id2 = string::utf8(b"doc2");
    upload_document(user1, string::utf8(b"Test1"), string::utf8(b"Hash1"), doc_id1);
    upload_document(user1, string::utf8(b"Test2"), string::utf8(b"Hash2"), doc_id2);

    // Add user2 as signer to only doc1
    add_signer(user1, doc_id1, user2_addr);

    // Verify user2 can only see doc1
    // Why test view functions? Users should only see documents they're authorized for
    let docs = get_documents_for_signer(user2_addr);
    assert!(vector::length(&docs) == 1, 0);
    let doc = vector::borrow(&docs, 0);
    assert!(doc.id == doc_id1, 1);
}
```

**Privacy and Access Control**: View functions must respect authorization boundaries. Users shouldn't be able to discover or access documents they're not authorized to see, which could leak sensitive business information.

## 5. Advanced Testing Patterns {#advanced-patterns}

### 5.1 Edge Case Testing

Always test boundary conditions that could cause unexpected behavior:

```move
#[test(admin = @document_management, user1 = @0x456, user2 = @0x789)]
fun test_upload_empty_document_id(
    admin: &signer,
    user1: &signer,
    user2: &signer
) acquires DocState, ResourceAccountCap {
    setup_test(admin, user1, user2);

    // Test with empty string ID
    // Why test edge cases? Empty strings might cause hash collisions or storage issues
    let doc_id = string::utf8(b"");
    let doc_name = string::utf8(b"Test Document");
    let ipfs_hash = string::utf8(b"QmTestHash");
    upload_document(user1, doc_name, ipfs_hash, doc_id);

    // Verify it works
    let (name, hash, _, _, _, _) = get_document(doc_id);
    assert!(name == doc_name, 0);
    assert!(hash == ipfs_hash, 1);
}
```

**Why Edge Cases Matter**: Real users will accidentally submit empty fields, very long strings, special characters, or other edge cases. Your contract must handle these gracefully or explicitly reject them with clear error messages.

### 5.2 State Consistency Testing

Test that operations maintain data consistency across multiple operations:

```move
#[test(admin = @document_management, user1 = @0x456, user2 = @0x789)]
fun test_add_same_signer_multiple_times(
    admin: &signer,
    user1: &signer,
    user2: &signer
) acquires DocState, ResourceAccountCap {
    setup_test(admin, user1, user2);
    let user2_addr = signer::address_of(user2);

    let doc_id = string::utf8(b"doc1");
    upload_document(user1, string::utf8(b"Test"), string::utf8(b"Hash"), doc_id);

    // Add the same signer twice
    // Why test this? Users might accidentally submit the same action multiple times
    add_signer(user1, doc_id, user2_addr);
    add_signer(user1, doc_id, user2_addr);

    // Verify signer only appears once
    // Why check this? Duplicate entries waste storage and could cause logic errors
    let (_, _, _, _, _, allowed_signers) = get_document(doc_id);
    assert!(vector::length(&allowed_signers) == 2, 0); // Owner + user2
    assert!(vector::contains(&allowed_signers, &user2_addr), 1);
}
```

**Idempotency**: Many operations should be idempotent (safe to repeat). This prevents issues from network retries, user double-clicks, or automated systems that might retry failed operations.

### 5.3 Testing Error Conditions

Systematically test all error conditions your contract defines:

```move
#[test(admin = @document_management, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_DOCUMENT_NOT_FOUND)]
fun test_get_non_existent_document(
    admin: &signer,
    user1: &signer,
    user2: &signer
) acquires DocState, ResourceAccountCap {
    setup_test(admin, user1, user2);

    let doc_id = string::utf8(b"non_existent_doc");
    get_document(doc_id); // Should abort with specific error code
}
```

**Error Handling Philosophy**: Good smart contracts fail fast with specific error codes rather than returning default values or partially working. This helps users and integrating systems understand exactly what went wrong.

## 6. Running Tests {#running-tests}

### Step 1: Run All Tests

```bash
aptos move test
```

### Step 2: Run Specific Tests

```bash
# Run tests matching a pattern (useful during development)
aptos move test --filter test_upload

# Run a specific test (for debugging specific issues)
aptos move test --filter test_upload_document
```

### Step 3: Verbose Output

```bash
# Get detailed test output (helpful for debugging test failures)
aptos move test --verbose
```

### Step 4: Test Coverage

```bash
# Generate test coverage report (ensures you're testing all code paths)
aptos move test --coverage
```

**Why Coverage Matters**: Untested code paths are potential security vulnerabilities. Aim for high coverage, but remember that 100% coverage doesn't guarantee bug-free code - the quality of your tests matters more than quantity.

## Best Practices Summary

1. **Comprehensive Setup**: Always initialize your test environment to match production conditions
2. **Test Both Success and Failure**: Use `#[expected_failure]` for error cases - security is often about what doesn't happen
3. **Meaningful Assertions**: Use descriptive error codes that make test failures immediately understandable
4. **Edge Cases**: Test boundary conditions, empty inputs, and maximum values
5. **State Verification**: Always verify state changes after operations - don't just test that functions don't crash
6. **Isolation**: Each test should be independent and not rely on others running in a specific order
7. **Clear Naming**: Use descriptive test function names that explain what scenario is being tested
8. **Documentation**: Comment complex test logic, especially the "why" behind unusual test scenarios

## Common Testing Patterns Explained

- **Setup-Execute-Verify (Arrange-Act-Assert)**: Standard test structure that mirrors how users will interact with your contract
- **Happy Path vs Error Path**: Test both successful operations and all the ways they can fail
- **State Consistency**: Verify that operations maintain data integrity across complex interactions
- **Authorization Testing**: Ensure proper access control - this is where many smart contract vulnerabilities occur

## Why This Level of Testing Matters

Smart contract testing isn't just about preventing bugs - it's about:

1. **Financial Security**: Protecting user funds and protocol assets
2. **Legal Compliance**: Ensuring contracts behave as legally specified
3. **User Experience**: Preventing confusion and failed transactions
4. **Protocol Sustainability**: Ensuring fee collection and tokenomics work correctly
5. **Integration Safety**: Allowing other contracts and dApps to interact safely

## Conclusion

Remember: In traditional software, bugs are embarrassing and costly. In smart contracts, bugs can be catastrophic and can be permanent if upgrade is not compatible. The extra effort in comprehensive testing pays for itself many times over through avoided security incidents and user trust.

This testing framework provides a solid foundation for ensuring your smart contract behaves correctly under all conditions, protecting both your users and your protocol's reputation.

Next Tutorial: [Frontend Integration Guide](./building_the_document_management_interface)
