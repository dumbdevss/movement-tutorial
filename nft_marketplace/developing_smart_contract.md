# NFT Marketplace Smart Contract Tutorial - Implementation Guide

This guide provides a detailed walkthrough of implementing an NFT Marketplace smart contract on the Aptos blockchain, with explanations of why each component is necessary and how it contributes to the system.

## Introduction

This tutorial guides you through building a feature-rich NFT Marketplace on the Movement blockchain, enabling:

- Creating and minting NFTs
- Listing NFTs for fixed-price sales
- Conducting auction-style sales with bidding
- Transferring NFT ownership
- Managing collections and NFT metadata

The marketplace leverages resource accounts and object model for secure and scalable operations. Resource accounts and signer capabilities ensure secure management of shared state, while tables and vectors optimize data storage and retrieval.

## TODO 1: Set Constants

Define constants to ensure consistent configuration across the contract:

```move
// Constants
const SEED: vector<u8> = b"marketplace_funds";
const MARKETPLACE_FEE_PERCENT: u64 = 5; // 5% fee
const CANCEL_FEE_PERCENT: u64 = 45; // 4.5% fee (45/1000 for precision)
```

**Why we need constants**: Constants centralize configuration values, making the contract easier to maintain and modify. They prevent hardcoding values throughout the code, reducing errors and improving readability.

**Why these specific constants**:

- `SEED`: A unique byte string used to generate a deterministic address for the resource account, ensuring predictable account creation for storing funds and data securely.
- `MARKETPLACE_FEE_PERCENT`: Sets a 5% fee on sales to sustain the marketplace's operations, ensuring economic viability.
- `CANCEL_FEE_PERCENT`: Applies a 4.5% fee (represented as 45/1000 for precision) when canceling auctions with active bids to discourage malicious cancellations and protect bidders.

## TODO 2: Define Error Codes

Error codes provide clear feedback for failed operations:

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

**Why we need error codes**: In Move, error codes are essential for secure and user-friendly error handling. They allow the contract to specify why a transaction failed, aiding debugging and providing clear feedback to users.

**Why specific error codes**: Each code corresponds to a unique failure scenario (e.g., `E_NOT_OWNER` for unauthorized access). Organized in ranges (e.g., 100s for ownership issues, 300s for general errors), they make error categorization intuitive and help developers quickly identify issues.

## TODO 3: Define the Offer Data Structure

The `Offer` structure represents a bid in an auction:

```move
struct Offer has store, drop, copy {
    bidder: address,
    amount: u64,
    timestamp: u64,
}
```

**Why we need the Offer structure**: It tracks bids in auctions, enabling the marketplace to record and compare offers efficiently. Using a vector to store offers allows iteration over all bids to find the highest or refund bidders.

**Why these fields and abilities**:

- `bidder`: Stores the bidder’s address for identification and fund transfers.
- `amount`: Records the bid amount in AptosCoin for comparison.
- `timestamp`: Captures when the bid was placed, useful for auditing or resolving disputes.
- **Abilities**: `store` allows storage in global state, `drop` permits discarding unused offers, and `copy` enables duplication for processing without modifying the original.

## TODO 4: Define the Auction Data Structure

The `Auction` structure manages auction-specific data:

```move
struct Auction has store, drop {
    deadline: option::Option<u64>,
    offers: vector<Offer>,
    highest_bid: u64,
    highest_bidder: option::Option<address>,
}
```

**Why we need the Auction structure**: It encapsulates all data needed to run an auction, ensuring organized tracking of bids and deadlines. Storing offers in a vector allows efficient iteration to process bids or refund them when needed.

**Why these fields**:

- `deadline`: Optional timestamp for auction end, supporting timed or open-ended auctions.
- `offers`: A vector of all bids for transparency and record-keeping, enabling iteration for processing.
- `highest_bid`: Tracks the current highest bid for quick reference, avoiding repeated vector iteration.
- `highest_bidder`: Stores the address of the highest bidder, if any, for efficient winner identification.

## TODO 5: Define the Collection Data Structure

The `Collection` structure groups related NFTs:

```move
struct Collection has copy, drop, store {
    name: String,
    description: String,
    uri: String,
    creator: address,
}
```

**Why we need Collections**: Collections organize NFTs, establishing provenance and enabling creators to manage related assets under a unified brand, which is critical for user experience and discoverability.

**Why these fields**:

- `name`: Identifies the collection for user display.
- `description`: Provides context or branding for the collection.
- `uri`: Points to off-chain metadata (e.g., images or details) for rich presentation.
- `creator`: Tracks the collection’s creator for attribution and access control.

## TODO 6: Define the NFT Data Structure

The `NFT` structure holds all NFT-related data:

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

**Why we need the NFT structure**: It centralizes all data for an NFT, enabling the marketplace to manage its state and interactions. Storing NFTs in a vector allows iteration for listing or filtering.

**Why these fields and abilities**:

- `id`: Unique identifier for tracking within the marketplace.
- `owner` and `creator`: Track ownership and origin for access control and provenance.
- `created_at`: Records creation time for auditing.
- `category`, `collection_name`, `name`, `description`, `uri`: Store metadata for user display and organization.
- `price`, `for_sale`, `sale_type`, `auction`: Manage sale status and type (instant or auction).
- `token`: Links to the Aptos token object for ecosystem compatibility.
- **Abilities**: `store` enables global storage, `key` ensures unique identification for resource management.

## TODO 7: Define the History Data Structure

The `History` structure tracks ownership transfers:

```move
struct History has store, drop, copy {
    new_owner: address,
    seller: address,
    amount: u64,
    timestamp: u64,
}
```

**Why we need History**: It provides a transparent record of NFT transfers, crucial for auditing and proving provenance, which builds trust in the marketplace.

**Why these fields**:

- `new_owner` and `seller`: Identify parties for clear transfer records.
- `amount`: Records the sale amount for transparency.
- `timestamp`: Marks when the transfer occurred for chronological tracking.

## TODO 8: Initialize the Marketplace

Initialize the marketplace with necessary resources:

```move
fun init_module(account: &signer) {
    // This creates a resource signer and a signer cap
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

**Why we need this function**: It sets up the marketplace’s core infrastructure, including a resource account to manage funds and state securely.

**Why resource accounts and signer capability**:

- **Resource Account**: A separate account created with `SEED` to hold shared state (e.g., NFTs, funds) and isolate marketplace operations from the deployer’s account, enhancing security.
- **Signer Capability**: A secure token (`signer_cap`) that allows the marketplace to perform actions on behalf of the resource account, such as transferring funds, without exposing the resource account’s private key.
- **Vectors**: Used for `nfts` and `collections` to enable dynamic storage and iteration over NFTs and collections, supporting scalable growth.

## TODO 9: Initialize a Collection

Implement a function to create a new collection:

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

**Why we need this function**: It allows creators to define collections, which organize NFTs and establish their provenance, enhancing discoverability and branding.

**Why these components**:

- **Resource Address**: Uses the deterministic address from `SEED` to access the `Collections` resource, ensuring consistent state management.
- **Vector for Collections**: Stores collections in a vector for efficient iteration when displaying or filtering collections.
- **Aptos Collection Framework**: Integrates with `collection::create_unlimited_collection` to ensure compatibility with Aptos’s token ecosystem.

## TODO 10: Mint an NFT

Implement the NFT minting function:

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

**Why we need this function**: It enables creators to mint new NFTs, the core asset of the marketplace, and ensures they are properly integrated into collections and the Aptos token framework.

**Token Functions Used**:

1. `token::create_named_token`: Creates a new NFT token within an existing collection
    - Parameters: creator signer, collection name, description, token name, property version (optional), URI
    - Returns: `ConstructorRef` for token setup

2. `object::generate_signer`: Creates a token signer from a constructor reference and returns the signer
    - Parameters: constructor_ref returned when from creating token
  
3. `object::generate_transfer_ref`: Create a transfer reference from a constructor reference and returns the transfer reference
    - Parameters: constructor_ref returned when from creating token

**Note:** Transfer reference is used to transfer token. It can also be used when try to disabe transfer of token

**Why these components**:

- **Collection Check**: Uses vector iteration to verify if the collection exists, ensuring NFTs are linked to valid collections.
- **Resource Account**: Accesses `Marketplace` and `Collections` via the resource address for secure state management.
- **Token Creation**: Leverages `token::create_named_token` for compatibility with Aptos’s object model.
- **Signer and Transfer References**: `token_signer` and `transfer_ref` enable secure token management, with `PermissionRef` stored to control future transfers.

## TODO 11: List an NFT for Sale

Implement the function to list an NFT for sale:

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

**Why we need this function**: It allows owners to list NFTs for sale, supporting both instant and auction sales, which are core marketplace functionalities.

**Why these components**:

- **Resource Account**: Accesses the `Marketplace` resource to update NFT state securely.
- **Vector Access**: Uses `vector::borrow_mut` to retrieve and modify the specific NFT by ID, leveraging vector’s indexing for efficiency.
- **Auction Initialization**: Creates an `Auction` struct with an empty offers vector for new auctions, enabling bid tracking.

## TODO 12: Place an Offer for an Auction

Implement the function to place a bid in an auction:

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

**Why we need this function**: It enables users to place bids in auctions, a key feature for dynamic pricing and competitive sales.

**Why these components**:

- **Resource Account and Signer Capability**: Uses `signer_cap` to securely transfer funds to the resource account, isolating bidder funds until the auction concludes.
- **Vector for Offers**: Stores bids in the auction’s offers vector for iteration and tracking, ensuring all bids are recorded.
- **Refund Logic**: Refunds the previous highest bidder with a cancellation fee to maintain fairness and deter manipulation.

## TODO 13: Finalize an Auction

Implement the function to finalize an auction:

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

**Why we need this function**: It finalizes auctions by transferring funds and NFTs to the appropriate parties, ensuring a secure and fair conclusion.

**Why these components**:

- **Resource Account and Signer Capability**: Uses `signer_cap` to securely transfer funds from the resource account to the seller.
- **PermissionRef**: Ensures only authorized transfers of the NFT using the stored transfer reference.
- **Vector Access**: Retrieves the NFT via vector indexing for efficient state updates.

## TODO 14: Purchase an NFT (Instant Sale)

Implement the function to purchase an NFT listed for instant sale:

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

**Why we need this function**: It enables instant purchases, providing a straightforward buying option for users.

**Why these components**:

- **Resource Account**: Transfers fees to the resource account for secure fund management.
- **PermissionRef**: Ensures secure NFT transfer using the stored transfer reference.
- **Vector Access**: Retrieves the NFT efficiently via vector indexing.

## TODO 15: Transfer an NFT

Implement the function to transfer an NFT:

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

**Why we need this function**: It allows owners to transfer NFTs without a sale, supporting gifting or private transfers.

**Why these components**:

- **PermissionRef**: Ensures only the owner can authorize the transfer, maintaining security.
- **Vector Access**: Retrieves the NFT efficiently for state updates.

## TODO 16: Cancel an NFT Listing

Implement the function to cancel an NFT listing:

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

**Why we need this function**: It allows owners to delist NFTs, providing flexibility while protecting bidders in auctions.

**Why these components**:

- **Resource Account and Signer Capability**: Refunds bidders securely using the resource account’s funds.
- **Vector Access**: Retrieves the NFT for state updates.
- **Refund Logic**: Applies a cancellation fee to deter abuse while ensuring fairness.

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

**Why we need this function**: It allows users to view their collections, enhancing discoverability and user experience.

**Why vector iteration**: Iterates over the `collections` vector to filter by creator, supporting pagination with `limit` and `offset` for scalability.

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

**Why we need this function**: It provides a view of all collections, supporting marketplace browsing.

**Why vector iteration**: Uses vector iteration for efficient retrieval, with `limit` and `offset` for pagination.

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

**Why we need this function**: It enables precise lookup of an NFT by collection, name, and owner, useful for user interfaces and verification.

**Why vector iteration**: Iterates over the `nfts` vector to find a matching NFT, as vectors are suitable for sequential searches in Move.

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

**Why we need this function**: It allows users to view their owned NFTs, a core feature for wallet functionality.

**Why vector iteration**: Filters NFTs by owner through vector iteration, ensuring all relevant NFTs are returned.

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

**Why we need this function**: It displays all NFTs available for sale, enabling marketplace browsing.

**Why vector iteration**: Filters for-sale NFTs through vector iteration, suitable for dynamic listings.

## Key Implementation Considerations

1. **Resource Account Pattern**: A separate account manages shared state and funds, using `signer_cap` to perform actions securely without exposing private keys, enhancing security.
2. **Vectors for Iteration**: Used for `nfts`, `collections`, and `offers` to support dynamic storage and efficient iteration for filtering and processing.
3. **Dual Sale Types**: Supports instant sales and auctions with a unified listing system for flexibility.
4. **Fee Management**: Automatically collects fees for sustainability and applies cancellation fees to protect bidders.
5. **Object Model Integration**: Uses Aptos’s object model for token management, ensuring ecosystem compatibility.
6. **Collection Management**: Organizes NFTs for provenance and discoverability.
7. **Transfer Controls**: `PermissionRef` ensures only authorized transfers, maintaining security.

## Conclusion

You’ve built a robust NFT marketplace on the Aptos blockchain, featuring:

- Secure NFT minting and management
- Dual-mode selling (instant and auctions)
- Safe ownership transfers and metadata management
- Sustainable fee collection

## Next Steps

Next Tutorial: [Building the NFT Marketplace Frontend](./building_the_nft_marketplace_interface)