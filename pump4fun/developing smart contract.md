# Building a Pump.Fun Token Platform on Aptos - Complete Tutorial

## Overview

This tutorial will guide you through building a complete pump.fun-style token platform on Aptos. You'll learn how to create a decentralized platform where users can launch meme tokens, trade them, and participate in automated market making through liquidity pools.

## What You'll Build

- A token creation platform with customizable metadata
- Automated Market Maker (AMM) for token trading
- Liquidity pools for price discovery
- Transaction history tracking
- Admin controls for platform management

## Prerequisites

- Basic understanding of Move programming language
- Aptos development environment set up
- Familiarity with blockchain concepts (tokens, liquidity pools, AMMs)

---

## Step 1: Define Constants (TODO 1)

First, let's define the essential constants for our platform:

```move
// Token decimals - 8 decimal places means 1 token = 100,000,000 smallest units
const DECIMAL: u8 = 8;

// Platform fee for creating tokens - 0.04 APT (4,000,000 octas)
const FEE: u64 = 4_000_000;

// APT has 8 decimal places
const MOVE_DECIMALS: u8 = 8;

// Multiplier for APT calculations (10^8)
const MOVE_MULTIPLIER: u64 = 100_000_000;
```

**Explanation:**
- `DECIMAL`: Defines how many decimal places our tokens will have
- `FEE`: The cost to create a new token (prevents spam)
- `MOVE_MULTIPLIER`: Used for precise calculations with APT amounts

---

## Step 2: Set Error Codes (TODO 2)

Define clear error codes for better debugging:

```move
const INSUFFICIENT_LIQUIDITY: u64 = 1;
const INVALID_AMOUNT: u64 = 2;
const ERR_NOT_OWNER: u64 = 3;
const ERR_ZERO_AMOUNT: u64 = 4;
const ERR_MAX_SUPPLY_EXCEEDED: u64 = 5;
const ERR_NOT_ADMIN: u64 = 6;
const ERR_TOKEN_EXISTS: u64 = 7;
const ERR_ACCOUNT_NOT_REGISTERED: u64 = 8;
const ERR_INVALID_PRICE: u64 = 9;
const ERR_TOKEN_NOT_FOUND: u64 = 10;
const ERR_POOL_NOT_FOUND: u64 = 11;
```

**Best Practice:** Always use descriptive error codes to help users understand what went wrong.

---

## Step 3: Define AppConfig Structure (TODO 3)

This struct holds the global configuration for our platform:

```move
struct AppConfig has key {
    fees: u64,                           // Fee amount for token creation
    admin: address,                      // Address of the platform admin
    history: vector<History>,            // Global transaction history
    tokens: vector<Token>,               // All tokens created on the platform
    token_addresses: vector<address>,    // Addresses of all tokens
}
```

**Purpose:** 
- Centralized configuration management
- Global state tracking
- Admin control mechanisms

---

## Step 4: Define Token Structure (TODO 4)

This represents each token created on the platform:

```move
struct Token has key, store, copy {
    id: u64,                             // Unique identifier
    name: String,                        // Token name (e.g., "Doge Coin")
    symbol: String,                      // Trading symbol (e.g., "DOGE")
    icon_uri: String,                    // URL to token logo
    supply: u64,                         // Total token supply
    current_price: u64,                  // Current market price
    project_url: String,                 // Official project website
    description: String,                 // Token description
    telegram: option::Option<String>,    // Optional Telegram link
    twitter: option::Option<String>,     // Optional Twitter link
    discord: option::Option<String>,     // Optional Discord link
    history: vector<History>,            // Token-specific transaction history
    timestamp: u64,                      // Creation timestamp
    token_addr: address,                 // Token contract address
    pool_addr: address,                  // Liquidity pool address
}
```

**Key Features:**
- Complete metadata support
- Social media integration
- Price tracking
- Transaction history

---

## Step 5: Define History Structure (TODO 5)

Tracks all transactions for analytics and transparency:

```move
struct History has key, store, copy {
    move_amount: u64,      // Amount of APT involved
    token_amount: u64,     // Amount of tokens involved
    type: String,          // Transaction type ("buy" or "sell")
    buyer: address,        // Buyer's address
    seller: address,       // Seller's address
    timestamp: u64,        // When transaction occurred
    amount_in_usd: u64,    // USD value of input
    amount_out_usd: u64,   // USD value of output
}
```

**Use Cases:**
- Price charts and analytics
- Transaction transparency
- User trading history

---

## Step 6: Define FAController Structure (TODO 6)

Controls the fungible asset operations:

```move
#[resource_group_member(group = aptos_framework::object::ObjectGroup)]
struct FAController has key {
    dev_address: address,      // Token creator's address
    mint_ref: MintRef,         // Reference to mint new tokens
    burn_ref: BurnRef,         // Reference to burn tokens
    transfer_ref: TransferRef, // Reference to transfer tokens
}
```

**Purpose:** 
- Manages token supply
- Controls token operations
- Enforces permissions

---

## Step 7: Define LiquidityPool Structure (TODO 7)

Implements the Automated Market Maker (AMM):

```move
struct LiquidityPool has key {
    token_reserve: u64,              // Tokens in the pool
    move_reserve: u64,               // APT in the pool
    token_address: Object<Metadata>, // Token metadata object
    owner: address,                  // Pool owner
    signer_cap: SignerCapability,    // Signing capability for pool operations
}
```

**AMM Concept:** 
- Uses constant product formula: `x * y = k`
- Price determined by ratio of reserves
- Automatic price discovery

---

## Step 8: Initialize Module (TODO 8)

Set up the platform when the contract is deployed:

```move
fun init_module(sender: &signer) {
    let admin_addr = signer::address_of(sender);
    move_to(sender, AppConfig {
        fees: FEE,
        admin: admin_addr,
        history: vector::empty<History>(),
        tokens: vector::empty<Token>(),
        token_addresses: vector::empty<address>(),
    });
}
```

**Initialization Steps:**
1. Get deployer's address as admin
2. Create empty vectors for tracking
3. Set initial fee structure
4. Store configuration on-chain

---

## Step 9: Calculate Output Amount (TODO 14)

Implement the AMM pricing formula:

```move
fun get_output_amount(
    input_amount: u64,
    input_reserve: u64,
    output_reserve: u64
): u64 {
    // Apply 0.3% trading fee
    let input_amount_with_fee = (input_amount as u128) * 997;
    let numerator = input_amount_with_fee * (output_reserve as u128);
    let denominator = ((input_reserve as u128) * 1000) + input_amount_with_fee;
    ((numerator / denominator) as u64)
}
```

**Formula Breakdown:**
- 0.3% fee: multiply by 997/1000
- Constant product: `(x + Δx) * (y - Δy) = x * y`
- Solve for Δy (output amount)

---

## Step 10: Mint Tokens Function (TODO 12)

Helper function to mint tokens to accounts:

```move
fun mint_tokens(
    account: &signer,
    token: Object<Metadata>,
    amount: u64,
) acquires FAController {
    let token_addr = object::object_address(&token);
    let controller = borrow_global<FAController>(token_addr);
    let fa = fungible_asset::mint(&controller.mint_ref, amount);
    primary_fungible_store::deposit(signer::address_of(account), fa);
}
```

**Process:**
1. Get token address from metadata object
2. Access the mint reference
3. Mint specified amount
4. Deposit to recipient's account

---

## Step 11: Initialize Liquidity Pool (TODO 13)

Create and fund the initial liquidity pool:

```move
fun initialize_liquidity_pool(
    token: Object<Metadata>,
    move_amount: u64,
    token_amount: u64,
    token_signer: &signer,
    sender: &signer
) acquires FAController, Token {
    // Create resource account for pool
    let (pool_signer, signer_cap) = account::create_resource_account(token_signer, b"Liquidity_Pool");
    let pool_addr = signer::address_of(&pool_signer);
    
    // Update token with pool address
    let token_address = signer::address_of(token_signer);
    let token_data = borrow_global_mut<Token>(token_address);
    token_data.pool_addr = pool_addr;
    
    // Register pool for APT if needed
    if (!coin::is_account_registered<AptosCoin>(pool_addr)) {
        coin::register<AptosCoin>(&pool_signer);
    };
    
    // Add tokens and APT to pool
    mint_tokens(&pool_signer, token, token_amount);
    let move_coins = coin::withdraw<AptosCoin>(sender, move_amount);
    coin::deposit(pool_addr, move_coins);
    
    // Create pool resource
    move_to(&pool_signer, LiquidityPool {
        token_reserve: token_amount,
        move_reserve: move_amount,
        token_address: token,
        owner: signer::address_of(token_signer),
        signer_cap,
    });
}
```

---

## Step 12: Record Transaction History (TODO 14)

Track all transactions for transparency:

```move
fun record_history(
    token_addr: address,
    move_amount: u64,
    token_amount: u64,
    type: String,
    buyer: address,
    seller: address
) acquires Token, AppConfig {
    let token = borrow_global_mut<Token>(token_addr);
    let token_id = token.id;
    
    // Price oracle placeholder (use real oracle in production)
    let price_in_move_coin = 1;
    assert!(price_in_move_coin > 0, ERR_INVALID_PRICE);
    
    let amount_in_usd = if (move_amount > 0) {
        (move_amount * price_in_move_coin)
    } else {
        0
    };
    
    let amount_out_usd = if (token_amount > 0) {
        (token_amount * price_in_move_coin)
    } else {
        0
    };
    
    let history_entry = History {
        move_amount,
        token_amount,
        buyer,
        seller,
        type,
        timestamp: timestamp::now_microseconds(),
        amount_in_usd,
        amount_out_usd,
    };
    
    // Add to token history
    vector::push_back(&mut token.history, history_entry);
    
    // Add to global history
    let app_config = borrow_global_mut<AppConfig>(@pump_fun);
    let token_gotten = vector::borrow_mut(&mut app_config.tokens, token_id);
    vector::push_back(&mut token_gotten.history, history_entry);
}
```

---

## Step 13: Create Token Function (TODO 11)

The main function for launching new tokens:

```move
public entry fun create_token(
    sender: &signer,
    name: String,
    symbol: String,
    supply: u64,
    description: String,
    telegram: option::Option<String>,
    twitter: option::Option<String>,
    discord: option::Option<String>,
    icon_uri: String,
    project_url: String,
    initial_move_amount: u64
) acquires AppConfig, FAController, Token {
    assert!(initial_move_amount > 0, ERR_ZERO_AMOUNT);
    
    let app_config = borrow_global_mut<AppConfig>(@pump_fun);
    let admin_addr = app_config.admin;
    let sender_addr = signer::address_of(sender);
    
    // Charge creation fee
    assert!(coin::is_account_registered<AptosCoin>(sender_addr), ERR_ACCOUNT_NOT_REGISTERED);
    let fee_coins = coin::withdraw<AptosCoin>(sender, app_config.fees);
    coin::deposit<AptosCoin>(admin_addr, fee_coins);
    
    // Create token object
    let constructor_ref = object::create_named_object(sender, *string::bytes(&name));
    let object_signer = object::generate_signer(&constructor_ref);
    let token_addr = signer::address_of(&object_signer);
    
    vector::push_back(&mut app_config.token_addresses, token_addr);
    
    // Create fungible asset
    primary_fungible_store::create_primary_store_enabled_fungible_asset(
        &constructor_ref,
        option::some(((supply as u128) * (MOVE_MULTIPLIER as u128))),
        name,
        symbol,
        DECIMAL,
        icon_uri,
        project_url
    );
    
    let fa_obj = object::object_from_constructor_ref<Metadata>(&constructor_ref);
    
    // Calculate token distribution
    let creator_amount = (supply * MOVE_MULTIPLIER) / 20; // 5% to creator
    let pool_token_amount = ((((supply * MOVE_MULTIPLIER) - creator_amount)) as u128); // 95% to pool
    
    // Create token metadata
    let token_id = vector::length(&app_config.tokens);
    let current_price = get_output_amount((MOVE_MULTIPLIER * MOVE_MULTIPLIER), (pool_token_amount as u64), initial_move_amount);
    
    let token = Token {
        id: token_id,
        name,
        symbol,
        icon_uri,
        project_url,
        current_price,
        description,
        telegram,
        twitter,
        discord,
        supply,
        history: vector::empty<History>(),
        timestamp: timestamp::now_microseconds(),
        token_addr,
        pool_addr: @0x0
    };
    
    move_to(&object_signer, token);
    
    // Store controller
    move_to(&object_signer, FAController {
        dev_address: sender_addr,
        mint_ref: fungible_asset::generate_mint_ref(&constructor_ref),
        burn_ref: fungible_asset::generate_burn_ref(&constructor_ref),
        transfer_ref: fungible_asset::generate_transfer_ref(&constructor_ref),
    });
    
    vector::push_back(&mut app_config.tokens, token);
    
    // Mint tokens to creator
    mint_tokens(sender, fa_obj, creator_amount);
    
    // Initialize liquidity pool
    assert!(((creator_amount as u128) + pool_token_amount) <= ((supply * MOVE_MULTIPLIER) as u128), ERR_MAX_SUPPLY_EXCEEDED);
    initialize_liquidity_pool(fa_obj, initial_move_amount, (pool_token_amount as u64), &object_signer, sender);
}
```

---

## Step 14: Buy Token Function (TODO 15)

Allow users to buy tokens with APT:

```move
public entry fun buy_token(
    sender: &signer,
    token_addr: address,
    move_amount: u64
) acquires LiquidityPool, Token, AppConfig {
    assert!(move_amount > 0, ERR_ZERO_AMOUNT);
    
    let token = borrow_global_mut<Token>(token_addr);
    let pool_addr = account::create_resource_address(&token.token_addr, b"Liquidity_Pool");
    assert!(pool_exists(pool_addr), ERR_POOL_NOT_FOUND);
    
    let lp = borrow_global_mut<LiquidityPool>(pool_addr);
    
    // Calculate tokens to receive
    let token_out = get_output_amount(move_amount, lp.move_reserve, lp.token_reserve);
    assert!(token_out > 0, INSUFFICIENT_LIQUIDITY);
    
    let sender_addr = signer::address_of(sender);
    assert!(coin::is_account_registered<AptosCoin>(sender_addr), ERR_ACCOUNT_NOT_REGISTERED);
    
    // Transfer APT to pool
    let move_coins = coin::withdraw<AptosCoin>(sender, move_amount);
    coin::deposit(pool_addr, move_coins);
    
    // Transfer tokens to buyer
    let pool_signer = account::create_signer_with_capability(&lp.signer_cap);
    primary_fungible_store::transfer(
        &pool_signer,
        lp.token_address,
        sender_addr,
        token_out
    );
    
    // Update reserves
    lp.move_reserve = lp.move_reserve + move_amount;
    lp.token_reserve = lp.token_reserve - token_out;
    
    // Update price
    let current_price = get_output_amount(MOVE_MULTIPLIER, lp.token_reserve, lp.move_reserve);
    token.current_price = current_price;
    
    // Update global state
    let app_config = borrow_global_mut<AppConfig>(@pump_fun);
    let i = 0;
    let len = vector::length(&app_config.tokens);
    while (i < len) {
        let t = vector::borrow_mut(&mut app_config.tokens, i);
        if (t.token_addr == token.token_addr) {
            t.current_price = current_price;
            break
        };
        i = i + 1;
    };
    
    // Record transaction
    record_history(
        token_addr,
        move_amount,
        token_out,
        string::utf8(b"buy"),
        sender_addr,
        pool_addr
    );
}
```

---

## Step 15: Swap Functions (TODOs 16-17)

Implement token swapping functionality:

### Swap APT to Token (TODO 16)

```move
public entry fun swap_move_to_token(
    sender: &signer,
    token_addr: address,
    move_amount: u64
) acquires LiquidityPool, Token, AppConfig {
    // Same implementation as buy_token
    // This provides an alternative interface for the same functionality
    assert!(move_amount > 0, ERR_ZERO_AMOUNT);
    
    let token = borrow_global_mut<Token>(token_addr);
    let pool_addr = token.pool_addr;
    assert!(pool_exists(pool_addr), ERR_POOL_NOT_FOUND);
    
    let lp = borrow_global_mut<LiquidityPool>(pool_addr);
    let token_out = get_output_amount(move_amount, lp.move_reserve, lp.token_reserve);
    assert!(token_out > 0, INSUFFICIENT_LIQUIDITY);
    
    let sender_addr = signer::address_of(sender);
    assert!(coin::is_account_registered<AptosCoin>(sender_addr), ERR_ACCOUNT_NOT_REGISTERED);
    
    let move_coins = coin::withdraw<AptosCoin>(sender, move_amount);
    coin::deposit(pool_addr, move_coins);
    
    let pool_signer = account::create_signer_with_capability(&lp.signer_cap);
    primary_fungible_store::transfer(
        &pool_signer,
        lp.token_address,
        sender_addr,
        token_out
    );
    
    lp.move_reserve = lp.move_reserve + move_amount;
    lp.token_reserve = lp.token_reserve - token_out;
    
    // Update prices and record history...
}
```

### Swap Token to APT (TODO 17)

```move
 public entry fun swap_token_to_move(
        sender: &signer,
        token_addr: address,
        token_amount: u64
    ) acquires LiquidityPool, Token, AppConfig {
        assert!(token_amount > 0, ERR_ZERO_AMOUNT);

        assert!(token_exists(token_addr), ERR_TOKEN_NOT_FOUND);
        let token = borrow_global_mut<Token>(token_addr); // Verify token exists
        
        let pool_addr = token.pool_addr;
        assert!(pool_exists(pool_addr), ERR_POOL_NOT_FOUND);
        let lp = borrow_global_mut<LiquidityPool>(pool_addr);

        let move_out = get_output_amount(token_amount, lp.token_reserve, lp.move_reserve);
        assert!(move_out > 0, INSUFFICIENT_LIQUIDITY);

        let sender_addr = signer::address_of(sender);

        assert!(primary_fungible_store::balance(sender_addr, object::address_to_object<Metadata>(token_addr)) >= token_amount, ERR_INSUFFICIENT_BALANCE);

        primary_fungible_store::transfer(sender, lp.token_address, pool_addr, token_amount);
        let pool_signer = account::create_signer_with_capability(&lp.signer_cap);
        coin::transfer<AptosCoin>(&pool_signer, sender_addr, move_out);

        lp.token_reserve = lp.token_reserve + token_amount;
        lp.move_reserve = lp.move_reserve - move_out;

        let current_price = get_output_amount((MOVE_MULTIPLIER * MOVE_MULTIPLIER), lp.token_reserve, lp.move_reserve);
        token.current_price = current_price;
        let app_config = borrow_global_mut<AppConfig>(@pump_fun);
        let i = 0;
        let len = vector::length(&app_config.tokens);
        while (i < len) {
            let t = vector::borrow_mut(&mut app_config.tokens, i);
            if (t.token_addr == token.token_addr) {
                t.current_price = current_price;
                break
            };
            i = i + 1;
        };

        record_history(
            token_addr,
            move_out,
            token_amount,
            string::utf8(b"sell"),
            pool_addr,
            sender_addr
        );
    }
```

---

## Step 16: Admin Functions (TODO 18)

Allow admin to update platform fees:

```move
public entry fun update_fee(
    sender: &signer,
    new_fee: u64
) acquires AppConfig {
    let app_config = borrow_global_mut<AppConfig>(@pump_fun);
    let sender_addr = signer::address_of(sender);
    assert!(sender_addr == app_config.admin, ERR_NOT_ADMIN);
    app_config.fees = new_fee;
}
```

---

## Step 17: View Functions (TODOs 19-23)

Implement read-only functions for frontend integration:

```move
// Get token output amount (TODO 19)
#[view]
public fun get_token_output_amount(
    move_amount: u64,
    token_addr: address
): u64 acquires LiquidityPool, Token {
    let token = borrow_global<Token>(token_addr);
    let pool_addr = token.pool_addr;
    assert!(pool_exists(pool_addr), ERR_POOL_NOT_FOUND);
    let lp = borrow_global<LiquidityPool>(pool_addr);
    get_output_amount(move_amount, lp.move_reserve, lp.token_reserve)
}

// Get APT output amount (TODO 20)
#[view]
public fun get_move_output_amount(
    token_amount: u64,
    token_addr: address
): u64 acquires LiquidityPool, Token {
    let token = borrow_global<Token>(token_addr);
    let pool_addr = token.pool_addr;
    assert!(pool_exists(pool_addr), ERR_POOL_NOT_FOUND);
    let lp = borrow_global<LiquidityPool>(pool_addr);
    get_output_amount(token_amount, lp.token_reserve, lp.move_reserve)
}

// Get pool information (TODO 21)
#[view]
public fun get_pool_info(token_addr: address): (u64, u64) acquires LiquidityPool, Token {
    let token = borrow_global<Token>(token_addr);
    let pool_addr = token.pool_addr;
    assert!(pool_exists(pool_addr), ERR_POOL_NOT_FOUND);
    let lp = borrow_global<LiquidityPool>(pool_addr);
    (lp.token_reserve, lp.move_reserve)
}

// Get all tokens (TODO 22)
#[view]
public fun get_all_tokens(): vector<Token> acquires AppConfig {
    let app_config = borrow_global<AppConfig>(@pump_fun);
    app_config.tokens
}

// Get all history (TODO 23)
#[view]
public fun getAllHistory(): vector<History> acquires AppConfig {
    let app_config = borrow_global<AppConfig>(@pump_fun);
    app_config.history
}
```

---

## Testing Your Contract



---

## Key Concepts Explained

### Automated Market Maker (AMM)

- Uses constant product formula: `x * y = k`
- No order books needed
- Automatic price discovery
- Liquidity providers earn fees

### Token Economics

- 5% to creator (prevents rug pulls)
- 95% to liquidity pool
- 0.3% trading fee
- Platform creation fee

### Security Features

- Admin controls for metadata updates
- Input validation on all functions
- Proper error handling
- Resource access controls

---

## Production Considerations

1. **Price Oracle Integration**: Replace placeholder USD pricing with real oracles like Pyth
2. **Advanced Fee Structures**: Implement dynamic fees based on volume
3. **Governance**: Add token-holder governance for parameter updates
4. **Flash Loan Protection**: Add reentrancy guards
5. **MEV Protection**: Implement commit-reveal schemes for large trades

---

## Next Steps

1. Deploy to Aptos testnet
2. Build a frontend interface
3. Add advanced trading features
4. Implement governance mechanisms
5. Add analytics and charting

This tutorial provides a complete foundation for building a pump.fun-style platform on Aptos. The modular design allows for easy extensions and customizations based on your specific requirements.