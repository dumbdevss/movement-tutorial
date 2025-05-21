# NFT Marketplace Smart Contract Tutorial - Implementation Guide

This guide walks through each TODO item in the NFT Marketplace smart contract with clear explanations and implementation details. Follow these steps to complete your Move module implementation.

## Introduction

This tutorial will guide you through implementing a full-featured NFT Marketplace on the Aptos blockchain. The marketplace will support:

- Creating and minting NFTs
- Listing NFTs for fixed-price sales
- Auction-style sales with bidding functionality
- Transferring NFT ownership
- Managing collections and NFT metadata

## TODO 1: Set Constants

First, let's define the constants that will be used throughout the contract:

```move
// Constants
const SEED: vector<u8> = b"marketplace_funds";
const MARKETPLACE_FEE_PERCENT: u64 = 5; // represent in 100%
const CANCEL_FEE_PERCENT: u64 = 45; // represented as 45/1000 for precision (4.5%)
```

**Implementation details:**

- `SEED` is used to create a deterministic resource account for the marketplace
- `MARKETPLACE_FEE_PERCENT` defines the marketplace's fee for sales (5%)
- `CANCEL_FEE_PERCENT` defines the fee for canceling auctions with active bids (4.5%, represented as 45/1000 for precision)

## TODO 2: Define Error Codes

Error codes help provide meaningful error messages when operations fail:

```move
// Error codes
const E_NOT_AUTHORIZED: u64 = 301;
const E_NOT_OWNER: u64 = 100;
const E_ALREADY_LISTED: u64 = 101;
const E_INVALID_PRICE: u64 = 102;
const E_NOT_FOR_SALE: u64 = 200;
const E_INSUFFICIENT_BALANCE: u64 = 201;
const E_SAME_OWNER: u64 = 302;
const E_NFT_NOT_FOUND: u64 = 303;
const E_INVALID_DEADLINE: u64 = 304;
const E_AUCTION_NOT_ENDED: u64 = 305;
const E_AUCTION_ENDED: u64 = 306;
const E_NOT_AUCTION: u64 = 307;
const E_INVALID_OFFER: u64 = 308;
const E_INVALID_SALE_TYPE: u64 = 309;
const E_NO_SIGNER_CAP: u64 = 310;
```

**Implementation details:**

- These error codes help identify specific failure scenarios
- Numbered ranges help categorize errors by type (e.g., ownership, sale, auction)

## TODO 3: Define the Offer Data Structure

The Offer structure represents a bid in an auction:

```move
struct Offer has store, drop, copy {
    bidder: address,
    amount: u64,
    timestamp: u64,
}
```

**Implementation details:**

- `bidder`: Address of the user making the offer
- `amount`: The bid amount in AptosCoin
- `timestamp`: When the offer was placed
- The structure has `store`, `drop`, and `copy` abilities for flexibility

## TODO 4: Define the Auction Data Structure

The Auction structure tracks auction-specific data:

```move
struct Auction has store, drop {
    deadline: option::Option<u64>,
    offers: vector<Offer>,
    highest_bid: u64,
    highest_bidder: option::Option<address>,
}
```

**Implementation details:**

- `deadline`: Optional timestamp when the auction ends
- `offers`: Vector containing all offers made for the auction
- `highest_bid`: Current highest bid amount
- `highest_bidder`: Optional address of the current highest bidder

## TODO 5: Define the Collection Data Structure

Collections group related NFTs:

```move
struct Collection has copy, drop, store {
    name: String,
    description: String,
    uri: String,
    creator: address,
}
```

**Implementation details:**

- `name`: Name of the collection
- `description`: Description of the collection
- `uri`: URI for collection metadata
- `creator`: Address of the collection creator

## TODO 6: Define the NFT Data Structure

The NFT structure contains all NFT-related data:

```move
#[resource_group_member(group = aptos_framework::object::ObjectGroup)]
struct NFT has store, key {
    id: u64,
    owner: address,
    creator: address,
    created_at: u64,
    category: String,
    collection_name: String,
    name: String,
    description: String,
    uri: String,
    price: u64,
    for_sale: bool,
    sale_type: u8,
    auction: option::Option<Auction>,
    token: Object<Token>,
}
```

**Implementation details:**

- `id`: Unique identifier for the NFT
- `owner`: Current owner's address
- `creator`: Original creator's address
- `created_at`: Timestamp of creation
- `category`: Category of the NFT
- `collection_name`: Name of the collection it belongs to
- `name`: Name of the NFT
- `description`: Description of the NFT
- `uri`: URI for NFT metadata
- `price`: Current price if for sale
- `for_sale`: Whether the NFT is listed for sale
- `sale_type`: Type of sale (instant or auction)
- `auction`: Optional auction details
- `token`: Reference to the underlying token object

## TODO 7: Define the History Data Structure

The History structure tracks ownership transfers:

```move
struct History has store, drop, copy {
    new_owner: address,
    seller: address,
    amount: u64,
    timestamp: u64,
}
```

**Implementation details:**

- `new_owner`: Address of the buyer/new owner
- `seller`: Address of the seller
- `amount`: Sale amount
- `timestamp`: When the transfer occurred

## TODO 8: Initialize the Marketplace

Now let's initialize the marketplace with the necessary resources:

```move
fun init_module(account: &signer) {
    let (resource_signer, signer_cap) = account::create_resource_account(account, SEED);
    coin::register<AptosCoin>(&resource_signer);
    move_to(&resource_signer, Marketplace {
        nfts: vector::empty<NFT>()
    });
    move_to(account, MarketplaceFunds {
        signer_cap
    });
    move_to(&resource_signer, Collections {
        collections: vector::empty<Collection>()
    });
}
```

**Implementation details:**

- Creates a resource account with the provided SEED
- Registers AptosCoin for the resource account
- Initializes the Marketplace with an empty NFTs vector
- Stores MarketplaceFunds with the signer capability
- Initializes Collections with an empty collections vector

## TODO 9: Initialize a Collection

Let's implement a function to initialize a new collection:

```move
public entry fun initialize_collection(
    creator: &signer,
    name: String,
    description: String,
    uri: String
) acquires Collections {
    let resources_address = account::create_resource_address(&@marketplace, SEED);
    let collections = borrow_global_mut<Collections>(resources_address);
    
    let creator_addr = signer::address_of(creator);
    let collection_data = Collection {
        name,
        description,
        uri,
        creator: creator_addr,
    };
    
    vector::push_back(&mut collections.collections, collection_data);
    
    // Create an unlimited collection
    collection::create_unlimited_collection(
        creator,
        description,
        name,
        option::none(),
        uri,
    );
}
```

**Implementation details:**

- Gets the resource account address
- Retrieves a mutable reference to the Collections resource
- Creates a new Collection struct with the provided details
- Adds the collection to the collections vector
- Creates an unlimited collection using the Aptos token framework

## TODO 10: Mint an NFT

Now let's implement the NFT minting function:

```move
public entry fun mint_nft(
    creator: &signer,
    collection_name: String,
    collection_description: option::Option<String>,
    collection_uri: option::Option<String>,
    name: String,
    description: String,
    category: String,
    uri: String
) acquires Marketplace, Collections {
    let resources_address = account::create_resource_address(&@marketplace, SEED);
    let marketplace = borrow_global_mut<Marketplace>(resources_address);
    let collections = borrow_global_mut<Collections>(resources_address);
    
    let creator_addr = signer::address_of(creator);
    
    // Check if collection exists; if not, initialize it
    let collection_exists = false;
    let i = 0;
    let len = vector::length(&collections.collections);
    
    while (i < len) {
        let collection = vector::borrow(&collections.collections, i);
        if (collection.name == collection_name && collection.creator == creator_addr) {
            collection_exists = true;
            break;
        };
        i = i + 1;
    };
    
    if (!collection_exists) {
        let desc = if (option::is_some(&collection_description)) {
            *option::borrow(&collection_description)
        } else {
            string::utf8(b"")
        };
        
        let coll_uri = if (option::is_some(&collection_uri)) {
            *option::borrow(&collection_uri)
        } else {
            string::utf8(b"")
        };
        
        initialize_collection(creator, collection_name, desc, coll_uri);
    };
    
    // Create a named token
    let token_constructor_ref = token::create_named_token(
        creator,
        collection_name,
        description,
        name,
        option::none(),
        uri,
    );
    
    let token_signer = object::generate_signer(&token_constructor_ref);
    let transfer_ref = object::generate_transfer_ref(&token_constructor_ref);
    let token_object = object::object_from_constructor_ref<Token>(&token_constructor_ref);
    
    let nft = NFT {
        id: vector::length(&marketplace.nfts),
        owner: creator_addr,
        creator: creator_addr,
        created_at: timestamp::now_seconds(),
        category,
        collection_name,
        name,
        description,
        uri,
        price: 0,
        for_sale: false,
        sale_type: SALE_TYPE_INSTANT,
        auction: option::none(),
        token: token_object,
    };
    
    vector::push_back(&mut marketplace.nfts, nft);
    move_to(&token_signer, PermissionRef { transfer_ref });
}
```

**Implementation details:**

- Gets the resource account address
- Retrieves mutable references to Marketplace and Collections resources
- Checks if the collection exists; if not, initializes it
- Creates a named token using the Aptos token objects framework
- Generates token signer and transfer reference
- Creates an NFT struct with provided details
- Adds the NFT to the marketplace's NFTs vector
- Stores the PermissionRef with the transfer reference

## TODO 11: List an NFT for Sale

Let's implement the function to list an NFT for sale:

```move
public entry fun list_nft_for_sale(
    owner: &signer,
    nft_id: u64,
    price: u64,
    sale_type: u8,
    auction_deadline: option::Option<u64>
) acquires Marketplace {
    let resources_address = account::create_resource_address(&@marketplace, SEED);
    let marketplace = borrow_global_mut<Marketplace>(resources_address);
    assert!(nft_id < vector::length(&marketplace.nfts), E_NFT_NOT_FOUND);
    let nft = vector::borrow_mut(&mut marketplace.nfts, nft_id);

    assert!(nft.owner == signer::address_of(owner), E_NOT_OWNER);
    assert!(!nft.for_sale, E_ALREADY_LISTED);
    assert!(sale_type == SALE_TYPE_INSTANT || sale_type == SALE_TYPE_AUCTION, E_INVALID_SALE_TYPE);
    if (option::is_some(&auction_deadline)) {
        assert!(*option::borrow(&auction_deadline) > timestamp::now_seconds(), E_INVALID_DEADLINE);
    };

    nft.for_sale = true;
    nft.sale_type = sale_type;
    nft.price = price;

    if (sale_type == SALE_TYPE_AUCTION) {
        nft.auction = option::some(Auction {
            deadline: auction_deadline,
            offers: vector::empty<Offer>(),
            highest_bid: 0,
            highest_bidder: option::none(),
        });
    };
}
```

**Implementation details:**

- Gets the resource account address
- Retrieves the NFT by ID from the marketplace
- Verifies the caller is the NFT owner
- Ensures the NFT is not already listed
- Validates the sale type and auction deadline
- Updates NFT fields: for_sale, sale_type, price
- If listing as an auction, initializes the Auction struct

## TODO 12: Place an Offer for an Auction

Now let's implement the function to place an offer in an auction:

```move
public entry fun place_offer(
    bidder: &signer,
    nft_id: u64,
    amount: u64
) acquires Marketplace, MarketplaceFunds {
    let resources_address = account::create_resource_address(&@marketplace, SEED);
    let marketplace = borrow_global_mut<Marketplace>(resources_address);
    assert!(nft_id < vector::length(&marketplace.nfts), E_NFT_NOT_FOUND);
    let nft = vector::borrow_mut(&mut marketplace.nfts, nft_id);

    assert!(nft.for_sale, E_NOT_FOR_SALE);
    assert!(nft.sale_type == SALE_TYPE_AUCTION, E_NOT_AUCTION);
    let auction = option::borrow_mut(&mut nft.auction);
    if (option::is_some(&auction.deadline)) {
        assert!(timestamp::now_seconds() < *option::borrow(&auction.deadline), E_AUCTION_ENDED);
    };
    assert!(amount > nft.price && amount > auction.highest_bid, E_INVALID_OFFER);

    let amount_to_pay = amount + ((amount * MARKETPLACE_FEE_PERCENT) / 100);

    let bidder_addr = signer::address_of(bidder);
    assert!(coin::balance<AptosCoin>(bidder_addr) >= amount_to_pay, E_INSUFFICIENT_BALANCE);

    // Get resource account
    let funds = borrow_global<MarketplaceFunds>(@marketplace);
    let resource_signer = account::create_signer_with_capability(&funds.signer_cap);
    let resource_addr = account::get_signer_capability_address(&funds.signer_cap);

    // Refund previous highest bidder if exists
    if (option::is_some(&auction.highest_bidder) && *option::borrow(&auction.highest_bidder) != bidder_addr) {
        let previous_bidder = *option::borrow(&auction.highest_bidder);
        let previous_bid = auction.highest_bid;
        let refund_amount = previous_bid + ((previous_bid * CANCEL_FEE_PERCENT) / 1000);
        assert!(coin::balance<AptosCoin>(resource_addr) >= refund_amount, E_INSUFFICIENT_BALANCE);
        coin::transfer<AptosCoin>(&resource_signer, previous_bidder, refund_amount);
    };

    // Lock bidder's coins in resource account
    coin::transfer<AptosCoin>(bidder, resource_addr, amount_to_pay);

    let offer = Offer {
        bidder: bidder_addr,
        amount,
        timestamp: timestamp::now_seconds(),
    };
    vector::push_back(&mut auction.offers, offer);

    auction.highest_bid = amount;
    auction.highest_bidder = option::some(bidder_addr);
}
```

**Implementation details:**

- Verifies the NFT exists, is for sale, and is an auction
- Checks that the auction hasn't ended
- Ensures the offer amount is higher than the minimum price and current highest bid
- Verifies the bidder has sufficient balance including fees
- Gets the resource account signer
- Refunds the previous highest bidder if exists
- Transfers the bid amount to the resource account
- Creates and stores the Offer struct
- Updates the auction's highest bid and bidder

## TODO 13: Finalize an Auction

Let's implement the function to finalize an auction:

```move
public entry fun finalize_auction(
    owner: &signer,
    nft_id: u64
) acquires Marketplace, PermissionRef, MarketplaceFunds {
    let resources_address = account::create_resource_address(&@marketplace, SEED);
    let marketplace = borrow_global_mut<Marketplace>(resources_address);
    assert!(nft_id < vector::length(&marketplace.nfts), E_NFT_NOT_FOUND);
    let nft = vector::borrow_mut(&mut marketplace.nfts, nft_id);

    assert!(nft.owner == signer::address_of(owner), E_NOT_OWNER);
    assert!(nft.for_sale, E_NOT_FOR_SALE);
    assert!(nft.sale_type == SALE_TYPE_AUCTION, E_NOT_AUCTION);
    let auction = option::borrow(&nft.auction);
    if (option::is_some(&auction.deadline)) {
        assert!(timestamp::now_seconds() >= *option::borrow(&auction.deadline), E_AUCTION_NOT_ENDED);
    };

    let funds = borrow_global<MarketplaceFunds>(@marketplace);
    let resource_signer = account::create_signer_with_capability(&funds.signer_cap);
    let resource_addr = account::get_signer_capability_address(&funds.signer_cap);

    if (option::is_some(&auction.highest_bidder)) {
        let winner = *option::borrow(&auction.highest_bidder);
        let amount = auction.highest_bid;

        // Transfer payment to seller and fee to marketplace
        coin::transfer<AptosCoin>(&resource_signer, nft.owner, amount);

        // Transfer NFT to winner
        let token = nft.token;
        let permission_ref = borrow_global<PermissionRef>(object::object_address(&token));
        let linear_transfer_ref = object::generate_linear_transfer_ref(&permission_ref.transfer_ref);
        object::transfer_with_ref(linear_transfer_ref, winner);

        nft.owner = winner;
    };

    nft.for_sale = false;
    nft.price = 0;
    nft.sale_type = SALE_TYPE_INSTANT;
    nft.auction = option::none();
}
```

**Implementation details:**

- Verifies the NFT exists, owner is caller, is for sale, and is an auction
- Checks the auction has ended if deadline exists
- Gets the resource account signer
- If there's a highest bidder:
  - Transfers payment to seller
  - Transfers NFT to winner
  - Updates NFT owner
- Updates NFT: clears sale status, price, and auction

## TODO 14: Purchase an NFT (Instant Sale)

Let's implement the function to purchase an NFT listed for instant sale:

```move
public entry fun purchase_nft(
    buyer: &signer,
    nft_id: u64
) acquires Marketplace, PermissionRef, MarketplaceFunds {
    let resources_address = account::create_resource_address(&@marketplace, SEED);
    let marketplace = borrow_global_mut<Marketplace>(resources_address);
    assert!(nft_id < vector::length(&marketplace.nfts), E_NFT_NOT_FOUND);
    let nft = vector::borrow_mut(&mut marketplace.nfts, nft_id);

    assert!(nft.for_sale, E_NOT_FOR_SALE);
    assert!(nft.sale_type == SALE_TYPE_INSTANT, E_NOT_AUCTION);
    let price = nft.price;
    let fee = (price * MARKETPLACE_FEE_PERCENT) / 100;
    let seller_amount = price - fee;

    let buyer_addr = signer::address_of(buyer);
    let seller_addr = nft.owner;

    assert!(coin::balance<AptosCoin>(buyer_addr) >= price, E_INSUFFICIENT_BALANCE);

    let funds = borrow_global<MarketplaceFunds>(@marketplace);
    let resource_addr = account::get_signer_capability_address(&funds.signer_cap);

    // Transfer payment to seller and resource account for fee
    coin::transfer<AptosCoin>(buyer, seller_addr, seller_amount);
    coin::transfer<AptosCoin>(buyer, resource_addr, fee);

    // Transfer NFT to buyer
    let token = nft.token;
    let permission_ref = borrow_global<PermissionRef>(object::object_address(&token));
    let linear_transfer_ref = object::generate_linear_transfer_ref(&permission_ref.transfer_ref);
    object::transfer_with_ref(linear_transfer_ref, buyer_addr);

    nft.owner = buyer_addr;
    nft.for_sale = false;
    nft.price = 0;
    nft.sale_type = SALE_TYPE_INSTANT;
    nft.auction = option::none();
}
```

**Implementation details:**

- Verifies the NFT exists, is for sale, and is instant sale
- Calculates fees and seller amount
- Verifies buyer has sufficient balance
- Gets the resource account address
- Transfers payment to seller and fee to resource account
- Transfers NFT to buyer
- Updates NFT: owner, clears sale status, price, and auction

## TODO 15: Transfer an NFT

Let's implement the function to transfer an NFT:

```move
public entry fun transfer_nft(
    owner: &signer,
    nft_id: u64,
    new_owner: address
) acquires Marketplace, PermissionRef {
    let resources_address = account::create_resource_address(&@marketplace, SEED);
    let marketplace = borrow_global_mut<Marketplace>(resources_address);
    assert!(nft_id < vector::length(&marketplace.nfts), E_NFT_NOT_FOUND);
    let nft = vector::borrow_mut(&mut marketplace.nfts, nft_id);

    assert!(nft.owner == signer::address_of(owner), E_NOT_OWNER);
    assert!(nft.owner != new_owner, E_SAME_OWNER);

    let token = nft.token;
    let permission_ref = borrow_global<PermissionRef>(object::object_address(&token));
    let linear_transfer_ref = object::generate_linear_transfer_ref(&permission_ref.transfer_ref);
    object::transfer_with_ref(linear_transfer_ref, new_owner);

    nft.owner = new_owner;
    nft.for_sale = false;
    nft.price = 0;
    nft.sale_type = SALE_TYPE_INSTANT;
    nft.auction = option::none();
}
```

**Implementation details:**

- Verifies the NFT exists and owner is caller
- Ensures new owner is different from current owner
- Transfers NFT to new owner
- Updates NFT: owner, clears sale status, price, and auction

## TODO 16: Cancel an NFT Listing

Let's implement the function to cancel an NFT listing:

```move
public entry fun cancel_listing(owner: &signer, nft_id: u64) acquires Marketplace, MarketplaceFunds {
    let resources_address = account::create_resource_address(&@marketplace, SEED);
    let marketplace = borrow_global_mut<Marketplace>(resources_address);
    assert!(nft_id < vector::length(&marketplace.nfts), E_NFT_NOT_FOUND);
    let nft = vector::borrow_mut(&mut marketplace.nfts, nft_id);
    assert!(nft.owner == signer::address_of(owner), E_NOT_OWNER);
    assert!(nft.for_sale, E_NOT_FOR_SALE);

    // If auction, refund the highest bidder with 4.5% fee
    if (nft.sale_type == SALE_TYPE_AUCTION && option::is_some(&nft.auction)) {
        let auction = option::borrow(&nft.auction);
        if (option::is_some(&auction.highest_bidder)) {
            let bidder = *option::borrow(&auction.highest_bidder);
            let bid_amount = auction.highest_bid;
            let refund_fee = (bid_amount * CANCEL_FEE_PERCENT) / 1000; // 4.5% fee
            let refund_amount = bid_amount + refund_fee;

            let funds = borrow_global<MarketplaceFunds>(@marketplace);
            let resource_signer = account::create_signer_with_capability(&funds.signer_cap);
            let resource_addr = account::get_signer_capability_address(&funds.signer_cap);

            assert!(coin::balance<AptosCoin>(resource_addr) >= refund_amount, E_INSUFFICIENT_BALANCE);
            coin::transfer<AptosCoin>(&resource_signer, bidder, refund_amount);
        };
    };

    nft.for_sale = false;
    nft.price = 0;
    nft.sale_type = SALE_TYPE_INSTANT;
    nft.auction = option::none();
}
```

**Implementation details:**

- Verifies the NFT exists, owner is caller, and is for sale
- If auction with a highest bidder:
  - Calculates refund amount with cancellation fee
  - Gets resource account signer
  - Transfers refund to bidder
- Updates NFT: clears sale status, price, and auction

## TODO 17: View Function - Get User Collections

```move
#[view]
public fun get_all_collections_by_user(account: address, limit: u64, offset: u64): vector<Collection> acquires Collections {
    let resources_address = account::create_resource_address(&@marketplace, SEED);
    let collections = borrow_global<Collections>(resources_address);
    let result = vector::empty<Collection>();
    let len = vector::length(&collections.collections);
    let i = offset;
    let count = 0;
    
    while (i < len && count < limit) {
        let collection = vector::borrow(&collections.collections, i);
        if (collection.creator == account) {
            vector::push_back(&mut result, *collection);
            count = count + 1;
        };
        i = i + 1;
    };
    
    result
}
```

**Implementation details:**

- Gets the Collections resource
- Creates an empty result vector
- Iterates through collections within limit and offset
- Filters collections by creator address
- Returns the filtered collections

## TODO 18: View Function - Get All Collections

```move
#[view]
public fun get_all_collections(limit: u64, offset: u64): vector<Collection> acquires Collections {
    let resources_address = account::create_resource_address(&@marketplace, SEED);
    let collections = borrow_global<Collections>(resources_address);
    let result = vector::empty<Collection>();
    let len = vector::length(&collections.collections);
    let i = offset;
    let count = 0;
    
    while (i < len && count < limit) {
        vector::push_back(&mut result, *vector::borrow(&collections.collections, i));
        count = count + 1;
        i = i + 1;
    };
    
    result
}
```

**Implementation details:**

- Gets the Collections resource
- Creates an empty result vector
- Iterates through collections within limit and offset
- Adds each collection to the result
- Returns the collections

## TODO 19: Get NFT By Collection and Token Name

```move
#[view]
public fun get_nft_by_collection_name_and_token_name(
    collection_name: String,
    token_name: String,
    user_address: address
): option::Option<NFT> acquires Marketplace {
    let resources_address = account::create_resource_address(&@marketplace, SEED);
    let marketplace = borrow_global<Marketplace>(resources_address);
    let len = vector::length(&marketplace.nfts);
    let i = 0;
    
    while (i < len) {
        let nft = vector::borrow(&marketplace.nfts, i);
        if (nft.collection_name == collection_name && nft.name == token_name && nft.owner == user_address) {
            return option::some(*nft)
        };
        i = i + 1;
    };
    
    option::none<NFT>()
}
```

**Implementation details:**

- Gets the Marketplace resource
- Iterates through all NFTs
- Finds NFT matching collection name, token name, and owner
- Returns the NFT if found, none otherwise

## TODO 20: Get User's NFTs

```move
#[view]
public fun get_user_nfts(
    owner: address
): vector<NFT> acquires Marketplace {
    let resources_address = account::create_resource_address(&@marketplace, SEED);
    let marketplace = borrow_global<Marketplace>(resources_address);
    let result = vector::empty<NFT>();
    let i = 0;
    
    while (i < vector::length(&marketplace.nfts)) {
        let nft = vector::borrow(&marketplace.nfts, i);
        if (nft.owner == owner) {
            vector::push_back(&mut result, *nft);
        };
        i = i + 1;
    };
    
    result
}
```

**Implementation details:**

- Gets the Marketplace resource
- Creates an empty result vector
- Iterates through all NFTs
- Filters NFTs by owner address
- Returns the filtered NFTs

## TODO 21: Get NFTs For Sale

```move
#[view]
public fun get_nfts_for_sale(): vector<NFT> acquires Marketplace {
    let resources_address = account::create_resource_address(&@marketplace, SEED);
    let marketplace = borrow_global<Marketplace>(resources_address);
    let result = vector::empty<NFT>();
    let i = 0;
    
    while (i < vector::length(&marketplace.nfts)) {
        let nft = vector::borrow(&marketplace.nfts, i);
        if (nft.for_sale) {
            vector::push_back(&mut result, *nft);
        };
        i = i + 1;
    };
    
    result
}
```

**Implementation details:**

- Gets the Marketplace resource
- Creates an empty result vector
- Iterates through all NFTs
- Filters NFTs that are for sale
- Returns the filtered NFTs

## Key Implementation Considerations

1. **Resource Account Pattern**: The marketplace uses a resource account to manage shared state and handle payments, which enhances security and simplifies fund management.

2. **Dual Sale Types**: The contract supports both instant sales and auctions with a unified listing system.

3. **Fee Management**: Marketplace fees are collected automatically for both sale types, and cancellation fees protect bidders from auction manipulation.

4. **Object Model Integration**: The contract leverages Aptos's object model for token management, ensuring compatibility with the broader Aptos ecosystem.

5. **Collection Management**: Collections provide a way to organize related NFTs and establish provenance.

6. **Transfer Controls**: The PermissionRef pattern ensures that only authorized parties can transfer NFTs.

## Testing Recommendations

To ensure your NFT marketplace implementation works correctly and securely, follow these testing guidelines:

1. **Test NFT Minting Process**
   - Verify NFT creation
   - Confirm proper token ownership assignment
   - Check collection association

2. **Verify Listing Functionality**
   - Test listing NFTs with different price points
   - Validate both instant sale and auction listing mechanisms
   - Ensure only NFT owners can create listings

3. **Test Auction Mechanics**
   - Verify bid placement and proper recording of highest bids
   - Test auction deadline enforcement
   - Confirm automatic refunds to outbid participants
   - Validate auction finalization with and without bids

4. **Test Purchase Flows**
   - Verify instant purchase works correctly
   - Confirm proper fee collection (marketplace commission)
   - Test ownership transfer upon successful purchase
   - Ensure listing state is properly updated after purchase

5. **Validate Cancellation Logic**
   - Test listing cancellation for both sale types
   - Verify refund mechanisms for auction participants
   - Confirm proper state reset after cancellation

6. **Security Testing**
   - Attempt unauthorized operations (purchases, transfers, updates)
   - Test with insufficient balance scenarios
   - Verify deadline and timestamp-related security measures

7. **View Function Accuracy**
   - Test all view functions with various states of NFTs
   - Verify correct filtering of user-owned and for-sale NFTs
   - Confirm NFT details are accurately reported

8. **Event Verification**
   - Verify that all relevant events are emitted correctly
   - Check event data for accuracy and completeness

## Conclusion

Congratulations! You've successfully built a complete NFT marketplace on the Movement blockchain by:

1. Implementing a robust Move smart contract with:
   - NFT minting and management
   - Dual-mode selling mechanisms (instant purchase and auctions)
   - Secure ownership transfers and metadata management
   - Marketplace fee collection for sustainability

## Next Steps

Next Tutorial: [Building the NFT Marketplace Frontend](./building_the_nft_marketplace_interface)
