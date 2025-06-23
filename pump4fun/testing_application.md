# Testing the NFT Marketplace Smart Contract

This section outlines a comprehensive testing strategy for the NFT Marketplace smart contract written in Move. The tests cover all critical functionalities, including module initialization, collection creation, NFT minting, listing, purchasing, auction management, transfers, and view functions. Each test ensures the contract behaves as expected under both success and failure conditions, adhering to the principles outlined in the provided document management system testing guide.

## 1. Test Environment Setup

The test environment setup initializes the blockchain state, including timestamps, coin systems, and account balances, to simulate a realistic economic environment.

```move
#[test_only]
fun setup_test(
    admin: &signer,
    user1: &signer,
    user2: &signer,
) {
    // Initialize timestamp system for auction deadlines
    let aptos_framework = account::create_account_for_test(@aptos_framework);
    timestamp::set_time_has_started_for_testing(&aptos_framework);

    // Initialize APT coin system for payments
    let (burn_cap, mint_cap) = aptos_coin::initialize_for_test(&aptos_framework);
    
    // Fund test accounts with 100 APT (8 decimal places)
    give_coins(&mint_cap, admin, 10000000000);
    give_coins(&mint_cap, user1, 10000000000);
    give_coins(&mint_cap, user2, 10000000000);

    // Clean up capabilities
    coin::destroy_burn_cap(burn_cap);
    coin::destroy_mint_cap(mint_cap);

    // Initialize the marketplace module
    init_module(admin);
}
```

**Why This Setup?**

- **Timestamps**: Required for auction deadlines and history tracking.
- **Coin System**: Ensures users can pay for NFTs and fees.
- **Resource Account**: The marketplace uses a resource account to manage fees, which must be initialized and funded.
- **Module Initialization**: Sets up the marketplace's global state, including empty NFT and collection vectors.

## 2. Testing Module Initialization

**Purpose**: Verify that the marketplace initializes correctly, creating necessary resources and registering for AptosCoin.

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
fun test_init_module(admin: &signer, user1: &signer, user2: &signer) {
    setup_test(admin, user1, user2);
    let resource_addr = account::create_resource_address(&@marketplace, SEED);
    
    // Verify resource account and structures
    assert!(exists<Marketplace>(resource_addr), E_NOT_INITIALIZED);
    assert!(exists<MarketplaceFunds>(@marketplace), E_NO_SIGNER_CAP);
    assert!(exists<Collections>(@marketplace), E_NOT_INITIALIZED);
    
    // Verify resource account is registered for APT
    assert!(coin::is_account_registered<AptosCoin>(resource_addr), E_NOT_INITIALIZED);
    
    // Verify initial state
    let marketplace = borrow_global<Marketplace>(resource_addr);
    assert!(vector::length(&marketplace.nfts) == 0, E_INVALID_STATE);
}
```

**Why This Matters**: Initialization bugs could prevent the marketplace from functioning, as all operations rely on the resource account and global state.

## 3. Testing Collection Initialization

**Purpose**: Ensure collections are created correctly and stored in the global state.

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
fun test_initialize_collection(admin: &signer, user1: &signer, user2: &signer) acquires Collections {
    setup_test(admin, user1, user2);
    let creator_addr = signer::address_of(user1);
    initialize_collection(
        user1,
        string::utf8(b"Test Collection"),
        string::utf8(b"Test Description"),
        string::utf8(b"https://test.com")
    );

    // Verify collection details
    let collections = get_all_collections(10, 0);
    assert!(vector::length(&collections) == 1, E_INVALID_STATE);
    let collection = vector::borrow(&collections, 0);
    assert!(collection.name == string::utf8(b"Test Collection"), E_INVALID_DATA);
    assert!(collection.description == string::utf8(b"Test Description"), E_INVALID_DATA);
    assert!(collection.uri == string::utf8(b"https://test.com"), E_INVALID_DATA);
    assert!(collection.creator == creator_addr, E_NOT_AUTHORIZED);
}
```

**Why This Matters**: Collections are the foundation for NFT organization. Incorrect initialization could lead to metadata errors or unauthorized access.

## 4. Testing NFT Minting

**Purpose**: Verify that NFTs are minted correctly, stored in the marketplace, and have proper initial state.

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
fun test_mint_nft(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections {
    setup_test(admin, user1, user2);
    let creator_addr = signer::address_of(user1);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );

    // Verify NFT details
    let nft = get_nft_details(0);
    assert!(nft.id == 0, E_INVALID_DATA);
    assert!(nft.owner == creator_addr, E_NOT_OWNER);
    assert!(nft.creator == creator_addr, E_NOT_AUTHORIZED);
    assert!(nft.collection_name == string::utf8(b"Test Collection"), E_INVALID_DATA);
    assert!(nft.name == string::utf8(b"Test NFT"), E_INVALID_DATA);
    assert!(nft.description == string::utf8(b"NFT Description"), E_INVALID_DATA);
    assert!(nft.uri == string::utf8(b"https://nft.com"), E_INVALID_DATA);
    assert!(nft.category == string::utf8(b"Art"), E_INVALID_DATA);
    assert!(nft.price == 0, E_INVALID_STATE);
    assert!(!nft.for_sale, E_INVALID_STATE);
    assert!(nft.sale_type == SALE_TYPE_INSTANT, E_INVALID_SALE_TYPE);
    assert!(option::is_none(&nft.auction), E_INVALID_STATE);
    assert!(vector::length(&nft.history) == 0, E_INVALID_STATE);
}
```

**Why This Matters**: Minting is the entry point for NFTs into the marketplace. Errors here could result in invalid tokens or ownership issues, breaking the entire system.

## 5. Testing NFT Listing

### 5.1 Instant Sale Listing

**Purpose**: Verify that NFTs can be listed for instant sale with correct state updates.

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
fun test_list_nft_instant_sale(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections {
    setup_test(admin, user1, user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );

    list_nft_for_sale(user1, 0, 1000, SALE_TYPE_INSTANT, option::none());
    let nft = get_nft_details(0);
    assert!(nft.for_sale, E_NOT_FOR_SALE);
    assert!(nft.sale_type == SALE_TYPE_INSTANT, E_INVALID_SALE_TYPE);
    assert!(nft.price == 1000, E_INVALID_DATA);
    assert!(option::is_none(&nft.auction), E_INVALID_STATE);
}
```

### 5.2 Auction Listing

**Purpose**: Ensure NFTs can be listed for auction with proper auction state initialization.

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
fun test_list_nft_auction(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections {
    setup_test(admin, user1, user2);
    timestamp::fast_forward_seconds(1000);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );

    list_nft_for_sale(user1, 0, 1000, SALE_TYPE_AUCTION, option::some(2000));
    let nft = get_nft_details(0);
    assert!(nft.for_sale, E_NOT_FOR_SALE);
    assert!(nft.sale_type == SALE_TYPE_AUCTION, E_INVALID_SALE_TYPE);
    assert!(nft.price == 1000, E_INVALID_DATA);
    assert!(option::is_some(&nft.auction), E_INVALID_STATE);
    let auction = option::borrow(&nft.auction);
    assert!(option::is_some(&auction.deadline), E_INVALID_STATE);
    assert!(*option::borrow(&auction.deadline) == 2000, E_INVALID_DATA);
    assert!(vector::length(&auction.offers) == 0, E_INVALID_STATE);
    assert!(auction.highest_bid == 0, E_INVALID_STATE);
    assert!(option::is_none(&auction.highest_bidder), E_INVALID_STATE);
}
```

### 5.3 Error Cases for Listing

**Non-Owner Listing**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_NOT_OWNER)]
fun test_list_nft_not_owner(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections {
    setup_test(admin, user1, user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    list_nft_for_sale(user2, 0, 1000, SALE_TYPE_INSTANT, option::none());
}
```

**Already Listed**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_ALREADY_LISTED)]
fun test_list_nft_already_listed(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections {
    setup_test(admin, user1, user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    list_nft_for_sale(user1, 0, 1000, SALE_TYPE_INSTANT, option::none());
    list_nft_for_sale(user1, 0, 2000, SALE_TYPE_INSTANT, option::none());
}
```

**Invalid Sale Type**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_INVALID_SALE_TYPE)]
fun test_list_nft_invalid_sale_type(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections {
    setup_test(admin, user1, user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    list_nft_for_sale(user1, 0, 1000, 99, option::none());
}
```

**Invalid Auction Deadline**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_INVALID_DEADLINE)]
fun test_list_nft_invalid_deadline(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections {
    setup_test(admin, user1, user2);
    timestamp::fast_forward_seconds(1000);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    list_nft_for_sale(user1, 0, 1000, SALE_TYPE_AUCTION, option::some(500));
}
```

**Why These Tests?**: Listing is a critical function that sets the stage for sales and auctions. Testing ownership, state transitions, and input validation prevents unauthorized listings and ensures economic security.

## 6. Testing Offer Placement

**Purpose**: Verify that offers can be placed on auctions, updating the highest bid and handling refunds correctly.

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
fun test_place_offer(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, MarketplaceFunds {
    setup_test(admin, user1, user2);
    timestamp::fast_forward_seconds(1000);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    list_nft_for_sale(user1, 0, 1000, SALE_TYPE_AUCTION, option::some(2000));
    let user2_addr = signer::address_of(user2);
    place_offer(user2, 0, 1500);

    let nft = get_nft_details(0);
    let auction = option::borrow(&nft.auction);
    assert!(vector::length(&auction.offers) == 1, E_INVALID_STATE);
    assert!(auction.highest_bid == 1500, E_INVALID_DATA);
    assert!(*option::borrow(&auction.highest_bidder) == user2_addr, E_NOT_AUTHORIZED);
    let offer = vector::borrow(&auction.offers, 0);
    assert!(offer.amount == 1500, E_INVALID_DATA);
    assert!(offer.bidder == user2_addr, E_NOT_AUTHORIZED);
}
```

### Error Cases for Offer Placement

**NFT Not Found**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_NFT_NOT_FOUND)]
fun test_place_offer_nft_not_found(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, MarketplaceFunds {
    setup_test(admin, user1, user2);
    place_offer(user2, 999, 1500);
}
```

**Not For Sale**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_NOT_FOR_SALE)]
fun test_place_offer_not_for_sale(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, MarketplaceFunds {
    setup_test(admin, user1, user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    place_offer(user2, 0, 1500);
}
```

**Not an Auction**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_NOT_AUCTION)]
fun test_place_offer_not_auction(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, MarketplaceFunds {
    setup_test(admin, user1, user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    list_nft_for_sale(user1, 0, 1000, SALE_TYPE_INSTANT, option::none());
    place_offer(user2, 0, 1500);
}
```

**Auction Ended**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_AUCTION_ENDED)]
fun test_place_offer_auction_ended(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, MarketplaceFunds {
    setup_test(admin, user1, user2);
    timestamp::fast_forward_seconds(1000);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    list_nft_for_sale(user1, 0, 1000, SALE_TYPE_AUCTION, option::some(1500));
    timestamp::fast_forward_seconds(1000);
    place_offer(user2, 0, 1500);
}
```

**Invalid Offer Amount**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_INVALID_OFFER)]
fun test_place_offer_invalid_amount(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, MarketplaceFunds {
    setup_test(admin, user1, user2);
    timestamp::fast_forward_seconds(1000);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    list_nft_for_sale(user1, 0, 1000, SALE_TYPE_AUCTION, option::some(2000));
    place_offer(user2, 0, 500);
}
```

**Why These Tests?**: Offer placement is critical for auctions. These tests ensure bids are valid, funds are locked correctly, and previous bidders are refunded, maintaining economic fairness.

## 7. Testing NFT Purchase (Instant Sale)

**Purpose**: Verify that instant sale purchases transfer ownership, process payments, and update state correctly.

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
fun test_purchase_nft(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, PermissionRef, MarketplaceFunds {
    setup_test(admin, user1, user2);
    let user1_addr = signer::address_of(user1);
    let user2_addr = signer::address_of(user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    list_nft_for_sale(user1, 0, 1000, SALE_TYPE_INSTANT, option::none());
    let initial_balance_user1 = coin::balance<AptosCoin>(user1_addr);
    let initial_balance_user2 = coin::balance<AptosCoin>(user2_addr);
    purchase_nft(user2, 0);

    let nft = get_nft_details(0);
    assert!(nft.owner == user2_addr, E_NOT_OWNER);
    assert!(!nft.for_sale, E_INVALID_STATE);
    assert!(nft.price == 0, E_INVALID_STATE);
    assert!(nft.sale_type == SALE_TYPE_INSTANT, E_INVALID_SALE_TYPE);
    assert!(option::is_none(&nft.auction), E_INVALID_STATE);
    assert!(vector::length(&nft.history) == 1, E_INVALID_STATE);
    let history = vector::borrow(&nft.history, 0);
    assert!(history.new_owner == user2_addr, E_NOT_OWNER);
    assert!(history.seller == user1_addr, E_NOT_AUTHORIZED);
    assert!(history.amount == 1000, E_INVALID_DATA);
    
    // Verify payment (5% fee)
    let fee = (1000 * MARKETPLACE_FEE_PERCENT) / 100;
    let seller_amount = 1000 - fee;
    assert!(coin::balance<AptosCoin>(user1_addr) == initial_balance_user1 + seller_amount, E_INVALID_BALANCE);
    assert!(coin::balance<AptosCoin>(user2_addr) == initial_balance_user2 - 1000, E_INVALID_BALANCE);
}
```

### Error Cases for Purchase

**NFT Not Found**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_NFT_NOT_FOUND)]
fun test_purchase_nft_not_found(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, PermissionRef, MarketplaceFunds {
    setup_test(admin, user1, user2);
    purchase_nft(user2, 999);
}
```

**Not For Sale**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_NOT_FOR_SALE)]
fun test_purchase_nft_not_for_sale(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, PermissionRef, MarketplaceFunds {
    setup_test(admin, user1, user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    purchase_nft(user2, 0);
}
```

**Not Instant Sale**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_NOT_AUCTION)]
fun test_purchase_nft_not_instant_sale(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, PermissionRef, MarketplaceFunds {
    setup_test(admin, user1, user2);
    timestamp::fast_forward_seconds(1000);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    list_nft_for_sale(user1, 0, 1000, SALE_TYPE_AUCTION, option::some(2000));
    purchase_nft(user2, 0);
}
```

**Insufficient Balance**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_INSUFFICIENT_BALANCE)]
fun test_purchase_nft_insufficient_balance(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, PermissionRef, MarketplaceFunds {
    setup_test(admin, user1, user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    list_nft_for_sale(user1, 0, 100000000000, SALE_TYPE_INSTANT, option::none());
    purchase_nft(user2, 0);
}
```

**Why These Tests?**: Purchasing is a core economic function. These tests ensure correct ownership transfer, fee calculation, and payment distribution, preventing financial losses or exploits.

## 8. Testing Auction Finalization

**Purpose**: Verify that auctions can be finalized, transferring NFTs to the highest bidder and funds to the seller.

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
fun test_finalize_auction(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, PermissionRef, MarketplaceFunds {
    setup_test(admin, user1, user2);
    timestamp::fast_forward_seconds(1000);
    let user1_addr = signer::address_of(user1);
    let user2_addr = signer::address_of(user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    list_nft_for_sale(user1, 0, 1000, SALE_TYPE_AUCTION, option::some(2000));
    place_offer(user2, 0, 1500);
    let initial_balance_user1 = coin::balance<AptosCoin>(user1_addr);
    timestamp::fast_forward_seconds(1000);
    finalize_auction(user1, 0);

    let nft = get_nft_details(0);
    assert!(nft.owner == user2_addr, E_NOT_OWNER);
    assert!(!nft.for_sale, E_INVALID_STATE);
    assert!(nft.price == 0, E_INVALID_STATE);
    assert!(nft.sale_type == SALE_TYPE_INSTANT, E_INVALID_SALE_TYPE);
    assert!(option::is_none(&nft.auction), E_INVALID_STATE);
    assert!(vector::length(&nft.history) == 1, E_INVALID_STATE);
    let history = vector::borrow(&nft.history, 0);
    assert!(history.new_owner == user2_addr, E_NOT_OWNER);
    assert!(history.seller == user1_addr, E_NOT_AUTHORIZED);
    assert!(history.amount == 1500, E_INVALID_DATA);
    assert!(coin::balance<AptosCoin>(user1_addr) == initial_balance_user1 + 1500, E_INVALID_BALANCE);
}
```

### Error Cases for Auction Finalization

**Not Owner**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_NOT_OWNER)]
fun test_finalize_auction_not_owner(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, PermissionRef, MarketplaceFunds {
    setup_test(admin, user1, user2);
    timestamp::fast_forward_seconds(1000);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    list_nft_for_sale(user1, 0, 1000, SALE_TYPE_AUCTION, option::some(2000));
    timestamp::fast_forward_seconds(1000);
    finalize_auction(user2, 0);
}
```

**Not For Sale**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_NOT_FOR_SALE)]
fun test_finalize_auction_not_for_sale(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, PermissionRef, MarketplaceFunds {
    setup_test(admin, user1, user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    finalize_auction(user1, 0);
}
```

**Not an Auction**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_NOT_AUCTION)]
fun test_finalize_auction_not_auction(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, PermissionRef, MarketplaceFunds {
    setup_test(admin, user1, user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    list_nft_for_sale(user1, 0, 1000, SALE_TYPE_INSTANT, option::none());
    finalize_auction(user1, 0);
}
```

**Auction Not Ended**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_AUCTION_NOT_ENDED)]
fun test_finalize_auction_not_ended(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, PermissionRef, MarketplaceFunds {
    setup_test(admin, user1, user2);
    timestamp::fast_forward_seconds(1000);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    list_nft_for_sale(user1, 0, 1000, SALE_TYPE_AUCTION, option::some(2000));
    finalize_auction(user1, 0);
}
```

**Why These Tests?**: Auction finalization transfers valuable assets and funds. These tests ensure only authorized users can finalize auctions, and only after the deadline, protecting both buyers and sellers.

## 9. Testing NFT Transfer

**Purpose**: Verify that NFTs can be transferred to new owners, clearing sale status.

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
fun test_transfer_nft(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, PermissionRef {
    setup_test(admin, user1, user2);
    let user2_addr = signer::address_of(user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    transfer_nft(user1, 0, user2_addr);

    let nft = get_nft_details(0);
    assert!(nft.owner == user2_addr, E_NOT_OWNER);
    assert!(!nft.for_sale, E_INVALID_STATE);
    assert!(nft.price == 0, E_INVALID_STATE);
    assert!(nft.sale_type == SALE_TYPE_INSTANT, E_INVALID_SALE_TYPE);
    assert!(option::is_none(&nft.auction), E_INVALID_STATE);
}
```

### Error Cases for Transfer

**Not Owner**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_NOT_OWNER)]
fun test_transfer_nft_not_owner(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, PermissionRef {
    setup_test(admin, user1, user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    transfer_nft(user2, 0, @0x789);
}
```

**Same Owner**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_SAME_OWNER)]
fun test_transfer_nft_same_owner(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, PermissionRef {
    setup_test(admin, user1, user2);
    let user1_addr = signer::address_of(user1);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    transfer_nft(user1, 0, user1_addr);
}
```

**Why These Tests?**: Transfers are a fundamental NFT operation. These tests ensure only owners can transfer NFTs and prevent redundant transfers, maintaining data integrity.

## 10. Testing Metadata Updates

**Purpose**: Verify that only creators can update NFT metadata.

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
fun test_update_nft_metadata(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections {
    setup_test(admin, user1, user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    update_nft_metadata(
        user1,
        0,
        string::utf8(b"Updated NFT"),
        string::utf8(b"Updated Description"),
        string::utf8(b"https://updated.com")
    );

    let nft = get_nft_details(0);
    assert!(nft.name == string::utf8(b"Updated NFT"), E_INVALID_DATA);
    assert!(nft.description == string::utf8(b"Updated Description"), E_INVALID_DATA);
    assert!(nft.uri == string::utf8(b"https://updated.com"), E_INVALID_DATA);
}
```

**Non-Creator Update**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_NOT_AUTHORIZED)]
fun test_update_nft_metadata_not_creator(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections {
    setup_test(admin, user1, user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    update_nft_metadata(
        user2,
        0,
        string::utf8(b"Updated NFT"),
        string::utf8(b"Updated Description"),
        string::utf8(b"https://updated.com")
    );
}
```

**Why These Tests?**: Metadata updates affect NFT value and representation. Restricting updates to creators prevents unauthorized changes that could mislead buyers.

## 11. Testing Listing Cancellation

**Purpose**: Verify that listings can be canceled, refunding bidders in auctions with appropriate fees.

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
fun test_cancel_listing_auction(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, MarketplaceFunds {
    setup_test(admin, user1, user2);
    timestamp::fast_forward_seconds(1000);
    let user2_addr = signer::address_of(user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    list_nft_for_sale(user1, 0, 1000, SALE_TYPE_AUCTION, option::some(2000));
    place_offer(user2, 0, 1500);
    let initial_balance_user2 = coin::balance<AptosCoin>(user2_addr);
    cancel_listing(user1, 0);

    let nft = get_nft_details(0);
    assert!(!nft.for_sale, E_INVALID_STATE);
    assert!(nft.price == 0, E_INVALID_STATE);
    assert!(nft.sale_type == SALE_TYPE_INSTANT, E_INVALID_STATE);
    assert!(option::is_none(&nft.auction), E_INVALID_STATE);
    
    // Verify refund with 4.5% fee
    let refund_fee = (1500 * CANCEL_FEE_PERCENT) / 1000;
    let refund_amount = 1500 + refund_fee;
    assert!(coin::balance<AptosCoin>(user2_addr) == initial_balance_user2 - 1500 + refund_amount, E_INVALID_BALANCE);
}
```

**Not For Sale**:

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
#[expected_failure(abort_code = E_NOT_FOR_SALE)]
fun test_cancel_listing_not_for_sale(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections, MarketplaceFunds {
    setup_test(admin, user1, user2);
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    cancel_listing(user1, 0);
}
```

**Why These Tests?**: Canceling listings must properly handle refunds to prevent financial loss for bidders and maintain marketplace integrity.

## 12. Testing View Functions

**Purpose**: Ensure view functions return accurate data for collections, NFTs, and user-owned assets.

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
fun test_view_functions(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections {
    setup_test(admin, user1, user2);
    let user1_addr = signer::address_of(user1);
    initialize_collection(
        user1,
        string::utf8(b"Test Collection"),
        string::utf8(b"Test Description"),
        string::utf8(b"https://test.com")
    );
    mint_nft(
        user1,
        string::utf8(b"Test Collection"),
        option::some(string::utf8(b"Test Description")),
        option::some(string::utf8(b"https://test.com")),
        string::utf8(b"Test NFT"),
        string::utf8(b"NFT Description"),
        string::utf8(b"Art"),
        string::utf8(b"https://nft.com")
    );
    list_nft_for_sale(user1, 0, 1000, SALE_TYPE_INSTANT, option::none());

    // Test get_all_collections_by_user
    let collections = get_all_collections_by_user(user1_addr, 10, 0);
    assert!(vector::length(&collections) == 1, E_INVALID_STATE);
    assert!(vector::borrow(&collections, 0).name == string::utf8(b"Test Collection"), E_INVALID_DATA);

    // Test get_all_collections
    let all_collections = get_all_collections(10, 0);
    assert!(vector::length(&all_collections) == 1, E_INVALID_STATE);

    // Test get_nft_by_collection_name_and_token_name
    let nft_opt = get_nft_by_collection_name_and_token_name(
        string::utf8(b"Test Collection"),
        string::utf8(b"Test NFT"),
        user1_addr
    );
    assert!(option::is_some(&nft_opt), E_INVALID_STATE);
    assert!(option::borrow(&nft_opt).name == string::utf8(b"Test NFT"), E_INVALID_DATA);

    // Test get_user_nfts
    let user_nfts = get_user_nfts(user1_addr);
    assert!(vector::length(&user_nfts) == 1, E_INVALID_STATE);
    assert!(vector::borrow(&user_nfts, 0).name == string::utf8(b"Test NFT"), E_INVALID_DATA);

    // Test get_nfts_for_sale
    let sale_nfts = get_nfts_for_sale();
    assert!(vector::length(&sale_nfts) == 1, E_INVALID_STATE);
    assert!(vector::borrow(&sale_nfts, 0).price == 1000, E_INVALID_DATA);
}
```

**Why These Tests?**: View functions are critical for front-end integration and user experience. They must return accurate data without leaking unauthorized information.

## 13. Edge Case Testing

**Purpose**: Test boundary conditions like empty strings and maximum values.

```move
#[test(admin = @marketplace, user1 = @0x456, user2 = @0x789)]
fun test_mint_nft_empty_strings(admin: &signer, user1: &signer, user2: &signer) acquires Marketplace, Collections {
    setup_test(admin, user1, user2);
    mint_nft(
        user1,
        string::utf8(b""),
        option::some(string::utf8(b"")),
        option::some(string::utf8(b"")),
        string::utf8(b""),
        string::utf8(b""),
        string::utf8(b""),
        string::utf8(b"")
    );

    let nft = get_nft_details(0);
    assert!(nft.collection_name == string::utf8(b""), E_INVALID_DATA);
    assert!(nft.name == string::utf8(b""), E_INVALID_DATA);
    assert!(nft.description == string::utf8(b""), E_INVALID_DATA);
    assert!(nft.uri == string::utf8(b""), E_INVALID_DATA);
    assert!(nft.category == string::utf8(b""), E_INVALID_DATA);
}
```

**Why This Matters**: Users may submit empty or malformed inputs. The contract should handle these gracefully to avoid state corruption.

## 14. Running Tests

To execute the tests, use the Aptos CLI:

```bash
# Run all tests
aptos move test

# Run specific tests
aptos move test --filter test_mint_nft

# Verbose output for debugging
aptos move test --verbose

# Generate coverage report
aptos move test --coverage
```

**Coverage Importance**: High test coverage ensures all code paths are exercised, reducing the risk of untested vulnerabilities.

## 15. Best Practices Summary

- **Comprehensive Setup**: Initialize a realistic blockchain environment with timestamps and funds.
- **Success and Failure Testing**: Use `#[expected_failure]` to verify error handling.
- **Descriptive Error Codes**: Use constants like `E_NOT_OWNER` for clear failure messages.
- **Edge Cases**: Test empty inputs and boundary conditions.
- **State Verification**: Check all state changes after operations.
- **Isolation**: Ensure tests are independent and deterministic.
- **Clear Naming**: Use descriptive test names like `test_mint_nft_empty_strings`.
- **Documentation**: Comment tests to explain the rationale behind scenarios.

## 16. Why This Testing Matters

Testing the NFT Marketplace contract ensures:

- **Financial Security**: Protects user funds during purchases and auctions.
- **Data Integrity**: Maintains accurate NFT ownership and metadata.
- **User Trust**: Prevents unauthorized actions that could erode confidence.
- **Economic Fairness**: Ensures fees and refunds are processed correctly.
- **Integration Safety**: Provides reliable view functions for front-end applications.

By thoroughly testing all scenarios, you mitigate the risk of costly bugs in an immutable blockchain environment, safeguarding both users and the protocol's reputation.
