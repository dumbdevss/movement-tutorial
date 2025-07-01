# PumpFun Frontend Tutorial - Complete Integration Guide

This comprehensive tutorial will guide you through building a complete decentralized token creation and swapping platform on the Movement blockchain. You'll learn how to implement token creation, swapping functionality, and holder management using React, GraphQL, and Move smart contracts.

## Understanding the Architecture

Before diving into implementation, let's understand how our PumpFun platform works:

**Our Tech Stack:**

- **Move Smart Contract**: Handles token creation, liquidity management, and swap operations
- **IPFS (via Pinata)**: Stores token images and metadata in a decentralized manner
- **React Frontend**: Provides the user interface for token operations
- **GraphQL (Apollo Client)**: Queries blockchain data for token holders and balances
- **Scaffold Hooks**: Abstracts blockchain interactions into simple React hooks

## Source Code Reference

The complete implementation of the frontend interface can be found in the repository:

**Repository:** [dumbdevss/pump-fun](https://github.com/dumbdevss/pump-fun)  
**Branch:** `frontend-integration`

You can view or clone the finished code to compare against your implementation as you follow along with this guide.

## Setting Up IPFS Integration

### Understanding IPFS for Token Metadata

**Why IPFS for Token Images?**
Storing images directly on the blockchain would be prohibitively expensive. Instead, we use IPFS (InterPlanetary File System) to store token images and metadata in a decentralized manner, then store only the IPFS hash on-chain. This approach provides:

- **Cost Efficiency**: Only small hashes are stored on-chain
- **Decentralization**: Images are distributed across IPFS nodes
- **Content Addressing**: Images are identified by their content hash, ensuring integrity
- **Immutability**: Once uploaded, images cannot be altered without changing their hash

**Why Pinata?**
Pinata is a pinning service that ensures your IPFS files remain available by keeping them "pinned" on their infrastructure. Without pinning, IPFS files might become unavailable if no nodes are hosting them.

### TODO 1: Setup Pinata and API Configuration

First, let's set up Pinata for decentralized image storage:

**Step 1: Create Pinata Account:**

1. Navigate to [pinata.cloud](https://www.pinata.cloud/)
2. Sign up for a free account
3. Verify your email address

**Step 2: Generate API Keys:**

1. Login to your Pinata dashboard
2. Navigate to the "API Keys" section
3. Click "New Key" and select appropriate permissions
4. Copy your API Key and API Secret

**Step 3: Configure Environment Variables**
Create a `.env` file in your project root:

```bash
NEXT_PUBLIC_PINATA_API_KEY=your_api_key_here
NEXT_PUBLIC_PINATA_API_SECRET=your_secret_here
NEXT_PUBLIC_PINATA_GATEWAY=https://gateway.pinata.cloud/ipfs/
```

**Step 4: Update Your Code Configuration**
Navigate to /components/token-creator.tsx, replace placeholder values:

```tsx
const PINATA_API_KEY = process.env.NEXT_PUBLIC_PINATA_API_KEY
const PINATA_API_SECRET = process.env.NEXT_PUBLIC_PINATA_API_SECRET
const PINATA_GATEWAY = process.env.NEXT_PUBLIC_PINATA_GATEWAY || "https://gateway.pinata.cloud/ipfs/"
```

**Security Note:** Environment variables prefixed with `NEXT_PUBLIC_` are accessible in the browser. Never store sensitive server-side secrets with this prefix.

## Token Creation Component (/components/token-creator.tsx)

Navigate to the token creator component to implement TODOs 2-17.

### Understanding Form State Management

**Why Controlled Components?**
In React, controlled components ensure that form state is managed by React rather than the DOM. This provides:

- **Single source of truth**: Form data is always in sync with component state
- **Validation**: Easy to implement real-time validation
- **Persistence**: Form state survives re-renders
- **Programmatic control**: Easy to reset, populate, or modify form data

### TODO 2: Implement Form Input Handler

Implement the universal form input handler:

```tsx
const handleChange = (e: React.ChangeEvent<HTMLInputElement | HTMLTextAreaElement>) => {
  const { name, value } = e.target
  setFormData((prev) => ({ ...prev, [name]: value }))
}
```

**Implementation Notes:**

- **Event Destructuring**: Extracts `name` and `value` from the input element
- **Functional setState**: Uses the previous state to ensure updates don't overwrite other form fields
- **Type Safety**: TypeScript ensures we handle both input and textarea elements correctly

### Understanding IPFS Upload Process

**The Upload Flow:**

1. User selects an image file
2. File is uploaded to Pinata's IPFS service
3. Pinata returns an IPFS hash
4. We construct a gateway URL for accessing the image
5. The URL is stored in component state for form submission

### TODO 3: Implement Image Upload to IPFS

Handle image file uploads to IPFS:

```tsx
const handleImageChange = async (e: React.ChangeEvent<HTMLInputElement>) => {
  if (e.target.files && e.target.files[0]) {
    const file = e.target.files[0]
    
    try {
      setIsUploadingImage(true)
      
      // Validate file type and size
      if (!file.type.startsWith('image/')) {
        throw new Error('Please select an image file')
      }
      
      if (file.size > 5 * 1024 * 1024) { // 5MB limit
        throw new Error('Image size must be less than 5MB')
      }
      
      const formData = new FormData()
      formData.append("file", file)
      
      const response = await fetch("https://api.pinata.cloud/pinning/pinFileToIPFS", {
        method: "POST",
        headers: {
          "pinata_api_key": PINATA_API_KEY || "",
          "pinata_secret_api_key": PINATA_API_SECRET || "",
        },
        body: formData,
      })
      
      if (!response.ok) {
        throw new Error(`Upload failed: ${response.statusText}`)
      }
      
      const data = await response.json()
      const ipfsHash = data.IpfsHash
      const fileUrl = `${PINATA_GATEWAY}ipfs/${ipfsHash}`
      
      setImageFile(file)
      setImagePreview(fileUrl)
      
      toast({
        title: "Image uploaded successfully",
        description: "Your token image has been stored on IPFS",
      })
    } catch (error) {
      console.error("Error uploading image:", error)
      toast({
        title: "Upload failed",
        description: error.message || "Failed to upload image to IPFS",
        variant: "destructive",
      })
    } finally {
      setIsUploadingImage(false)
    }
  }
}
```

**Implementation Notes:**

- **File Validation**: Checks file type and size before uploading
- **FormData**: Required for multipart file uploads
- **Error Handling**: Comprehensive error handling for network failures
- **Loading States**: Provides user feedback during upload process
- **IPFS Hash**: The unique identifier returned by IPFS becomes part of our gateway URL

### Understanding Token Creation Process

**The Token Creation Flow:**

1. User fills out token details (name, symbol, description, etc.)
2. Smart contract validates the parameters
3. Token is created with specified supply and initial liquidity
4. Creator becomes the initial holder of the token supply
5. Initial liquidity is locked in the swap contract

### TODO 4: Implement Token Creation Submission

Handle the token creation form submission:

```tsx
const handleSubmit = async (e: React.FormEvent) => {
  e.preventDefault()
  
  // Validate required fields
  if (!formData.name.trim()) {
    toast({
      title: "Validation Error",
      description: "Token name is required",
      variant: "destructive",
    })
    return
  }
  
  if (!formData.symbol.trim()) {
    toast({
      title: "Validation Error", 
      description: "Token symbol is required",
      variant: "destructive",
    })
    return
  }
  
  if (!formData.supply || parseFloat(formData.supply) <= 0) {
    toast({
      title: "Validation Error",
      description: "Token supply must be greater than 0",
      variant: "destructive",
    })
    return
  }
  
  if (!formData.initialLiquidity || parseFloat(formData.initialLiquidity) <= 0) {
    toast({
      title: "Validation Error",
      description: "Initial liquidity must be greater than 0",
      variant: "destructive",
    })
    return
  }
  
  setIsCreating(true)
  
  try {
    const response = await createToken(
      formData.name,
      formData.symbol,
      imagePreview || "",
      formData.projectUrl,
      DECIMAL * parseFloat(formData.initialLiquidity), // Convert to blockchain units
      formData.supply,
      formData.description,
      formData.telegram || null,
      formData.twitter || null,
      formData.discord || null
    )
    
    if (response) {
      setIsSuccess(true)
      toast({
        title: "Token Created Successfully",
        description: `${formData.name} (${formData.symbol}) has been created`,
      })
    }
  } catch (error) {
    console.error("Error creating token:", error)
    toast({
      title: "Token Creation Failed",
      description: "There was an error creating your token. Please try again.",
      variant: "destructive",
    })
  } finally {
    setIsCreating(false)
  }
}
```

**Implementation Notes:**

- **Frontend Validation**: Prevents unnecessary blockchain transactions
- **DECIMAL Conversion**: Blockchain math typically uses fixed-point arithmetic
- **Error Handling**: Comprehensive error feedback for users
- **Loading States**: Prevents multiple submissions during processing

### TODO 5: Implement Modal Reset and Close

Reset form state and close the creation modal:

```tsx
const closeModal = () => {
  // Reset form data
  setFormData({
    name: "",
    symbol: "",
    projectUrl: "",
    description: "",
    twitter: "",
    discord: "",
    telegram: "",
    supply: "",
    initialLiquidity: "",
  })
  
  // Reset image state
  setImageFile(null)
  setImagePreview(null)
  
  // Reset wizard state
  setStep(1)
  setIsSuccess(false)
  setIsCreating(false)
  
  // Close modal
  onClose()
}
```

**Why Complete Reset is Important:**

- **Clean State**: Ensures no residual data affects the next token creation
- **User Experience**: Users expect a fresh form when reopening the modal
- **Memory Management**: Prevents accumulation of unused file references

### TODOs 6-17: Connect Form Elements

Connect all form handlers to their respective JSX elements:

```tsx
// TODO 6: Form submission handler
<form onSubmit={handleSubmit} className="space-y-6">

// TODO 7: Token name input
<input 
  name="name" 
  value={formData.name}
  onChange={handleChange}
  placeholder="Enter token name"
  required
/>

// TODO 8: Token symbol input
<input 
  name="symbol" 
  value={formData.symbol}
  onChange={handleChange}
  placeholder="Enter token symbol (e.g., BTC)"
  required
/>

// TODO 9: Project URL input
<input 
  name="projectUrl" 
  value={formData.projectUrl}
  onChange={handleChange}
  placeholder="https://yourproject.com"
  type="url"
/>

// TODO 10: Description textarea
<textarea 
  name="description" 
  value={formData.description}
  onChange={handleChange}
  placeholder="Describe your token and project"
  rows={4}
/>

// TODO 11: Twitter handle input
<input 
  name="twitter" 
  value={formData.twitter}
  onChange={handleChange}
  placeholder="@yourhandle"
/>

// TODO 12: Discord server input
<input 
  name="discord" 
  value={formData.discord}
  onChange={handleChange}
  placeholder="Discord invite link"
/>

// TODO 13: Telegram group input
<input 
  name="telegram" 
  value={formData.telegram}
  onChange={handleChange}
  placeholder="Telegram group link"
/>

// TODO 14: Token supply input
<input 
  name="supply" 
  value={formData.supply}
  onChange={handleChange}
  placeholder="1000000"
  type="number"
  min="1"
  required
/>

// TODO 15: Initial liquidity input
<input 
  name="initialLiquidity" 
  value={formData.initialLiquidity}
  onChange={handleChange}
  placeholder="Amount in MOVE"
  type="number"
  min="0.01"
  step="0.01"
  required
/>

// TODO 16: Image upload input
<input 
  type="file" 
  accept="image/*"
  onChange={handleImageChange}
  disabled={isUploadingImage}
/>

// TODO 17: Success modal close button
<Button onClick={closeModal} variant="outline">
  Close
</Button>
</form>
```

**Form UX Best Practices:**

- **Required Fields**: Clearly marked and validated
- **Input Types**: Appropriate input types improve mobile experience
- **Placeholders**: Provide clear examples of expected input
- **Validation**: Real-time feedback prevents submission errors

## Token Holders Component (/components/token-holders.tsx)

Navigate to the token holders component to implement TODOs 18-21.

### Understanding GraphQL Integration

**Why GraphQL for Blockchain Data?**
GraphQL provides several advantages for querying blockchain data:

- **Efficient Queries**: Request only the data you need
- **Real-time Updates**: Can be configured for live data
- **Type Safety**: Schema validation prevents errors
- **Flexible Filtering**: Complex queries with multiple conditions

**Indexer Services:**
Movement blockchain uses indexers that parse blockchain events and store them in queryable databases. This allows complex queries that would be expensive to perform directly on-chain.

### TODO 18: Define GraphQL Query for Token Holders

Define the query to fetch token holder information:

```tsx
const GET_TOKEN_HOLDERS = gql`
  query GetTokenHolders($assetType: String!) {
    current_fungible_asset_balances(
      where: {
        asset_type: {_eq: $assetType}, 
        amount: {_gt: 0}
      }
      order_by: {amount: desc}
    ) {
      owner_address
      amount
      last_transaction_timestamp
      metadata {
        creator_address
        name
        symbol
      }
    }
  }
`
```

**Query Breakdown:**

- **Variables**: `$assetType` allows dynamic token address filtering
- **Filtering**: Only returns holders with balance > 0
- **Sorting**: Orders by balance amount (largest holders first)
- **Metadata**: Includes token information for verification

### TODO 19: Implement Data Fetching Hook

Use Apollo Client to fetch token holders:

```tsx
const { loading, error, data, refetch } = useQuery(GET_TOKEN_HOLDERS, {
  variables: { assetType: tokenAddress },
  skip: !tokenAddress, // Skip query if no token address provided
  pollInterval: 30000, // Refresh every 30 seconds
  errorPolicy: 'all', // Show partial data even with errors
})
```

**Hook Configuration:**

- **Skip Condition**: Prevents unnecessary queries
- **Polling**: Keeps data fresh with automatic updates
- **Error Policy**: Graceful handling of partial failures
- **Refetch**: Manual refresh capability for user-initiated updates

### TODO 20: Process and Transform Data

Transform the GraphQL response into component-friendly format:

```tsx
const holders = useMemo(() => {
  if (!data?.current_fungible_asset_balances) return []
  
  return data.current_fungible_asset_balances.map((holder: any, index: number) => ({
    id: index,
    address: holder.owner_address,
    amount: parseInt(holder.amount),
    percentage: 0, // Calculated below
    timestamp: holder.last_transaction_timestamp,
    displayAddress: `${holder.owner_address.slice(0, 6)}...${holder.owner_address.slice(-4)}`,
  }))
}, [data])
```

**Data Transformation Notes:**

- **useMemo**: Prevents unnecessary recalculations on re-renders
- **Amount Parsing**: Converts string amounts to numbers for calculations
- **Address Display**: Creates user-friendly truncated addresses
- **Indexing**: Provides stable keys for React rendering

### TODO 21: Calculate Total Supply and Percentages

Calculate total supply for percentage ownership:

```tsx
const { totalSupply, holdersWithPercentages } = useMemo(() => {
  const total = holders.reduce((sum: number, holder: any) => sum + holder.amount, 0)
  
  const withPercentages = holders.map(holder => ({
    ...holder,
    percentage: total > 0 ? (holder.amount / total) * 100 : 0
  }))
  
  return {
    totalSupply: total,
    holdersWithPercentages: withPercentages
  }
}, [holders])
```

**Percentage Calculation:**

- **Reduce Operation**: Efficiently sums all holder balances
- **Division by Zero**: Guards against empty token scenarios
- **Precision**: Maintains decimal precision for small percentages

## Token Swap Component (/components/token-swap-all.tsx)

Navigate to the token swap component to implement TODOs 22-25.

### Understanding Automated Market Makers (AMM)

**How Token Swapping Works:**
Our platform uses an Automated Market Maker (AMM) model:

1. **Liquidity Pools**: Each token has a pool containing both the token and MOVE
2. **Price Discovery**: Prices are determined by the ratio of tokens in the pool
3. **Slippage**: Large trades can move prices significantly
4. **Fees**: A small percentage goes to liquidity providers

**Mathematical Formula:**
The constant product formula: `x * y = k`
- `x` = amount of token A in pool
- `y` = amount of token B in pool  
- `k` = constant (must remain the same after trades)

### TODO 22: Define User Balance Query

Query the user's token balance for swap calculations:

```tsx
const GET_USER_TOKEN_BALANCE = gql`
  query GetUserTokenBalance($owner_address: String!, $asset_type: String!) {
    current_fungible_asset_balances(
      where: {
        owner_address: {_eq: $owner_address},
        asset_type: {_eq: $asset_type}
      }
    ) {
      asset_type
      amount
      last_transaction_timestamp
      metadata {
        name
        symbol
        decimals
      }
    }
  }
`
```

**Balance Query Features:**

- **User-Specific**: Filters by connected wallet address
- **Token-Specific**: Gets balance for the selected token
- **Metadata**: Includes token details for display purposes

### TODO 23: Implement Balance Data Fetching

Fetch user's token balance with proper error handling:

```tsx
const { 
  data: tokenBalanceData, 
  loading: tokenBalanceLoading, 
  refetch: refetchTokenBalance,
  error: balanceError 
} = useQuery(GET_USER_TOKEN_BALANCE, {
  variables: {
    owner_address: account?.address || "",
    asset_type: currentTokenAddress
  },
  skip: !account?.address || !currentTokenAddress,
  fetchPolicy: "cache-and-network", // Always try to get fresh data
  errorPolicy: "all"
})

// Calculate user's token balance
const userTokenBalance = useMemo(() => {
  if (!tokenBalanceData?.current_fungible_asset_balances?.[0]) return 0
  return parseInt(tokenBalanceData.current_fungible_asset_balances[0].amount) / DECIMAL
}, [tokenBalanceData])
```

**Fetch Policy Considerations:**

- **cache-and-network**: Shows cached data immediately, then updates with network data
- **Skip Logic**: Prevents queries when required data is missing
- **Balance Calculation**: Converts blockchain units to human-readable format

### TODO 24: Implement Swap Logic

Handle the core swapping functionality:

```tsx
const handleSwap = async () => {
  // Validation
  if (!fromAmount || parseFloat(fromAmount) <= 0) {
    toast({
      title: "Invalid Amount",
      description: "Please enter a valid swap amount",
      variant: "destructive",
    })
    return
  }
  
  if (!connected || !account?.address) {
    toast({
      title: "Wallet Not Connected",
      description: "Please connect your wallet to swap tokens",
      variant: "destructive",
    })
    return
  }
  
  const amount = parseFloat(fromAmount)
  
  // Check balance
  if (fromToken === "MOVE") {
    // Check MOVE balance (implement MOVE balance query separately)
    const moveBalance = await getMoveBalance()
    if (amount > moveBalance) {
      toast({
        title: "Insufficient Balance",
        description: "You don't have enough MOVE tokens",
        variant: "destructive",
      })
      return
    }
  } else {
    // Check token balance
    if (amount > userTokenBalance) {
      toast({
        title: "Insufficient Balance", 
        description: "You don't have enough tokens",
        variant: "destructive",
      })
      return
    }
  }
  
  setIsLoading(true)
  
  try {
    if (fromToken === "MOVE") {
      // Swapping MOVE to token
      await swapMoveToToken(currentTokenAddress, Math.floor(amount * DECIMAL))
    } else {
      // Swapping token to MOVE
      await swapTokensToMove(currentTokenAddress, Math.floor(amount * DECIMAL))
    }
    
    // Clear form and refresh balances
    setFromAmount("")
    setToAmount("")
    await refetchTokenBalance()
    
    toast({
      title: "Swap Successful",
      description: `Successfully swapped ${amount} ${fromToken}`,
    })
    
  } catch (error) {
    console.error("Swap failed:", error)
    toast({
      title: "Swap Failed",
      description: error.message || "Transaction failed. Please try again.",
      variant: "destructive",
    })
  } finally {
    setIsLoading(false)
  }
}
```

**Swap Implementation Notes:**

- **Multi-layer Validation**: Frontend validation prevents wasted transactions
- **Balance Checking**: Ensures users have sufficient funds
- **Unit Conversion**: Converts human-readable amounts to blockchain units
- **State Management**: Clears form and refreshes data after successful swaps

### TODO 25: Connect Swap Interface

Connect the swap button to the handleSwap function:

```tsx
<Button
  onClick={handleSwap}
  disabled={!fromAmount || isLoading || parseFloat(fromAmount) <= 0 || !connected}
  className="w-full bg-gradient-to-r from-purple-500 to-pink-500 hover:from-purple-600 hover:to-pink-600 transition-all duration-200"
>
  {isLoading ? (
    <>
      <RefreshCw className="mr-2 h-4 w-4 animate-spin" />
      Swapping...
    </>
  ) : (
    <>
      <ArrowUpDown className="mr-2 h-4 w-4" />
      Swap Tokens
    </>
  )}
</Button>
```

**Button UX Features:**

- **Disabled States**: Prevents invalid operations
- **Loading Indicators**: Visual feedback during transactions
- **Gradient Styling**: Modern, engaging appearance
- **Icons**: Visual cues for button purpose

## Main Page Component (page.tsx)

Navigate to the main page component to implement TODOs 26-37.

### Understanding Data Flow Architecture

**Component Hierarchy:**
```
Page (Data Layer)
├── TokenCreator (Creation)
├── TokenSwapAll (Trading)  
├── TokenCard (Display)
└── TokenDetails Modal
    ├── TokenSwap (Individual)
    └── TokenHolders (Analytics)
```

**Data Flow:**
1. Page component fetches all blockchain data
2. Data is passed down to child components as props
3. Actions (create, swap) are passed down as callback functions
4. Child components trigger parent updates via callbacks

### TODO 26: Fetch All Tokens Data

Implement the primary data fetching for all platform tokens:

```tsx
const {
  data: tokensData,
  isLoading: tokensLoading,
  error: tokensError,
  refetch: refreshTokensData,
} = useView({
  moduleName: "pump_for_fun",
  functionName: "get_all_tokens",
  args: [], // No arguments needed for getting all tokens
})

// Process tokens data
const refinedTokenData = useMemo(() => {
  if (!tokensData) return []
  
  return tokensData.map((token: any, index: number) => ({
    id: index,
    name: token.name,
    symbol: token.symbol,
    description: token.description,
    image: token.icon_uri,
    creator: token.creator,
    supply: parseInt(token.supply),
    timestamp: parseInt(token.created_at),
    project_uri: token.project_uri,
    social_links: {
      twitter: token.twitter,
      discord: token.discord,
      telegram: token.telegram,
    },
    token_address: token.token_address,
  }))
}, [tokensData])
```

**Data Processing Benefits:**

- **Consistent Format**: Transforms blockchain data into predictable structure
- **Type Safety**: Explicit property mapping prevents runtime errors
- **Performance**: useMemo prevents unnecessary re-processing
- **Indexing**: Provides stable IDs for React list rendering

### TODO 27: Fetch Transaction History

Implement transaction history fetching for activity feed:

```tsx
const {
  data: allHistoryData,
  isLoading: historyLoading,
  error: historyError,
  refetch: refreshAllHistoryData,
} = useView({
  moduleName: "pump_for_fun",
  functionName: "getAllHistory",
  args: [],
})

// Process history data
const processedHistory = useMemo(() => {
  if (!allHistoryData) return []
  
  return allHistoryData.map((transaction: any, index: number) => ({
    id: index,
    type: transaction.transaction_type, // 'create', 'swap_to_move', 'swap_to_token'
    user: transaction.user_address,
    token_address: transaction.token_address,
    amount: parseInt(transaction.amount),
    timestamp: parseInt(transaction.timestamp),
    hash: transaction.transaction_hash,
  }))
    .sort((a, b) => b.timestamp - a.timestamp) // Most recent first
    .slice(0, 50) // Limit to recent 50 transactions
}, [allHistoryData])
```

**History Data Features:**

- **Transaction Types**: Different types of platform activities
- **Sorting**: Most recent transactions first
- **Pagination**: Limits data for performance
- **Hash Storage**: Links to blockchain explorer

### TODO 28: Implement Token Creation Transaction

Handle token creation with comprehensive error handling:

```tsx
const createToken = async (
  tokenName: string, 
  tokenSymbol: string, 
  icon_uri: string, 
  project_uri: string, 
  initial_liquidity: number, 
  supply: string, 
  description: string, 
  telegram: string | null, 
  twitter: string | null, 
  discord: string | null
) => {
  // Wallet connection check
  if (!account) {
    toast({
      title: "Wallet Required",
      description: "Please connect your wallet to create a token",
      variant: "destructive",
    })
    return null
  }
  
  // Input validation
  if (!tokenName.trim() || !tokenSymbol.trim()) {
    toast({
      title: "Missing Information",
      description: "Token name and symbol are required",
      variant: "destructive",
    })
    return null
  }
  
  if (parseInt(supply) <= 0 || initial_liquidity <= 0) {
    toast({
      title: "Invalid Values",
      description: "Supply and initial liquidity must be greater than 0",
      variant: "destructive",
    })
    return null
  }
  
  try {
    // Submit transaction to blockchain
    const response = await submitTransaction("create_token", [
      tokenName.trim(),
      tokenSymbol.trim().toUpperCase(),
      parseInt(supply),
      description.trim(),
      telegram,
      twitter, 
      discord,
      icon_uri,
      project_uri,
      initial_liquidity
    ])

    if (response) {
      toast({
        title: "Token Created Successfully",
        description: `${tokenName} (${tokenSymbol}) has been created and is now live`,
        variant: "default",
      })
      
      // Refresh data to show new token
      await refreshTokensData()
      await refreshAllHistoryData()
      
      return response
    } else {
      throw new Error("Transaction failed without error details")
    }
  } catch (error: any) {
    console.error("Token creation error:", error)
    
    // Parse error messages
    let errorMessage = "An unexpected error occurred"
    if (error.message?.includes("insufficient funds")) {
      errorMessage = "Insufficient MOVE balance for transaction"
    } else if (error.message?.includes("symbol already exists")) {
      errorMessage = "Token symbol already exists, please choose another"
    } else if (error.message) {
      errorMessage = error.message
    }
    
    toast({
      title: "Token Creation Failed",
      description: errorMessage,
      variant: "destructive",
    })
    
    return null
  }
}
```

**Transaction Function Features:**

- **Pre-validation**: Catches errors before expensive blockchain calls
- **Error Parsing**: Converts blockchain errors to user-friendly messages
- **Data Refresh**: Updates UI immediately after successful transactions
- **Return Values**: Allows calling components to handle success/failure

### TODO 29: Implement SwapTokensToMove Function

First, implement the function to swap custom tokens for MOVE tokens:

```tsx
const swapTokensToMove = async (token_addr: string, token_amount: number) => {
  if (!account) {
    toast({
      title: "Please connect your wallet",
      description: "You need to connect your wallet to swap tokens",
      variant: "destructive",
    })
    return
  }
  
  try {
    const response = await submitTransaction("swap_token_to_move", [
      token_addr as `0x${string}`, 
      token_amount
    ]);
    
    if (response as any) {
      toast({
        title: "Swap successful",
        description: `You have successfully swapped tokens to MOVE`,
        variant: "default",
      })
      refreshTokensData()
    } else {
      toast({
        title: "Swap failed",
        description: `There was an error swapping tokens`,
        variant: "destructive",
      })
    }
  } catch (error: any) {
    toast({
      title: "Swap failed",
      description: `There was an error swapping tokens: ${error.message}`,
      variant: "destructive",
    })
  }
}
```

**Implementation Notes:**

- **Wallet Check**: Prevents transaction attempts without proper authentication
- **Token Address**: The smart contract needs to know which token to swap from
- **Amount Handling**: Token amounts should already be scaled by the DECIMAL factor
- **Data Refresh**: After successful swaps, we refresh the UI to show updated balances

**Understanding Swap Mechanics:**
When swapping tokens to MOVE, the smart contract:

1. Takes your custom tokens
2. Adds them to the liquidity pool
3. Calculates the equivalent MOVE amount based on current pool ratios
4. Transfers MOVE tokens to your wallet

### TODO 30: Implement SwapMoveToToken Function

Next, implement the reverse operation - swapping MOVE for custom tokens:

```tsx
const swapMoveToToken = async (token_addr: string, move_amount: number) => {
  if (!account) {
    toast({
      title: "Please connect your wallet",
      description: "You need to connect your wallet to swap tokens",
      variant: "destructive",
    })
    return
  }
  
  try {
    const response = await submitTransaction("swap_move_to_token", [
      token_addr as `0x${string}`, 
      move_amount
    ]);

    if (response as any) {
      toast({
        title: "Swap successful",
        description: `You have successfully swapped MOVE to tokens`,
        variant: "default",
      })
      refreshTokensData()
    } else {
      toast({
        title: "Swap failed",
        description: `There was an error swapping tokens`,
        variant: "destructive",
      })
    }
  } catch (error: any) {
    toast({
      title: "Swap failed",
      description: `There was an error swapping tokens: ${error.message}`,
      variant: "destructive",
    })
  }
}
```

**Implementation Notes:**

- **Wallet Connection**: Both swap functions check for a connected wallet before proceeding.
- **Transaction Submission**: Uses a generic `submitTransaction` helper to interact with smart contracts.
- **Parameter Types**: Token addresses are cast to the expected string format for Move contracts.
- **Error Handling**: User-friendly toast notifications are shown for both success and failure cases.
- **UI Refresh**: After a successful swap, token data is refreshed to reflect updated balances.
- **Amount Scaling**: Amounts passed to the contract should be pre-scaled using the `DECIMAL` constant.
- **Separation of Concerns**: Each function handles only one swap direction, improving maintainability.

### TODO 31: Implement HandleTokenClick Function

Implement token selection for the details modal:

```tsx
const handleTokenClick = (token: Token) => {
  setSelectedToken(token)
}
```

### TODOs 32-33: Connect TokenCard Click Events

Connect TokenCard components to the handleTokenClick function:

```tsx
// TODO 32: Main token grid
{filteredTokens.map((token, index) => (
  <motion.div
    key={token.id}
    initial={{ opacity: 0, y: 20 }}
    animate={{ opacity: 1, y: 0 }}
    transition={{ delay: index * 0.1 }}
    onClick={() => handleTokenClick(token)}
  >
    <TokenCard token={token} />
  </motion.div>
))}

// TODO 33: Recent tokens section
{filteredTokens
  .sort((a, b) => b.timestamp - a.timestamp)
  .slice(0, 6)
  .map((token, index) => (
    <motion.div
      key={token.id}
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ delay: index * 0.1 }}
      onClick={() => handleTokenClick(token)}
    >
      <TokenCard token={token} />
    </motion.div>
  ))}
```

**Understanding the Code Structure:**

- **Filtering**: `filteredTokens` allows users to search/filter the token list
- **Sorting**: Recent tokens are sorted by timestamp (newest first)
- **Limiting**: `.slice(0, 6)` shows only the 6 most recent tokens
- **Animation**: Framer Motion provides staggered entrance animations
- **Event Handling**: Each card becomes clickable to open the detail modal

### TODOs 34-35: Connect TokenSwapAll Props

Pass swap functions to the TokenSwapAll component:

```tsx
<TokenSwapAll 
  tokens={refinedTokenData} 
  swapMoveToToken={swapMoveToToken}
  swapTokensToMove={swapTokensToMove}
/>
```

### TODO 36: Connect TokenSwap Props in Modal

Pass swap functions to the TokenSwap component within the modal:

```tsx
<TokenSwap 
  selectedToken={selectedToken} 
  tokens={refinedTokenData} 
  swapMoveToToken={swapMoveToToken}
  swapTokensToMove={swapTokensToMove}
/>
```

**Modal-Specific Considerations:**

- **Selected Token**: The modal focuses on one specific token for detailed operations
- **Full Token Data**: Still needs access to all tokens for swap calculations
- **Consistent Interface**: Uses the same swap functions as the main interface

### TODO 37: Connect TokenCreator Props

Pass the createToken function to the TokenCreator component:

```tsx
<TokenCreator
  onClose={() => setShowCreateModal(false)}
  createToken={createToken}
/>
```

## Testing the Application

### 1. Setting Up the Development Environment

**Starting the Application:**
Run this command on your terminal.

```bash
npm run dev
```

Navigate to `http://localhost:3000` to checkout the application

## Key Technical Notes

### Apollo Client Configuration

Ensure your Apollo Client is properly configured:

```tsx
const client = new ApolloClient({
  uri: process.env.NEXT_PUBLIC_GRAPHQL_ENDPOINT,
  cache: new InMemoryCache(),
  defaultOptions: {
    watchQuery: {
      errorPolicy: 'all'
    }
  }
});
```

**Why GraphQL for Token Data:**

- **Efficient Queries**: Fetch only needed data
- **Real-time Updates**: Subscribe to token changes
- **Type Safety**: Schema provides compile-time checks

### Blockchain Precision Handling

**The DECIMAL Constant:**
```tsx
const DECIMAL = 100000000; // 1e8
```

**Usage Examples:**

- User enters "1.5" tokens → Store as `150000000` on blockchain
- Blockchain returns `250000000` → Display as "2.5" tokens
- Always scale amounts before sending to smart contract

## Next Steps

After successfully implementing the token swap functionality, consider adding these advanced features:

**Enhanced Trading Features:**

- Price charts and historical data
- Advanced order types (limit orders, stop losses)
- Portfolio tracking and analytics

**Liquidity Management:**

- Add/remove liquidity interfaces
- Liquidity provider rewards tracking
- Impermanent loss calculations

**Social Features:**

- Token creator profiles
- Community voting on token listings
- Social trading features

## Conclusion

You've successfully built a comprehensive token swap DApp that demonstrates:

1. **Smart Contract Integration**: Seamless interaction with Move-based DEX contracts
2. **Token Management**: Complete lifecycle from creation to trading
3. **User Experience**: Professional UI with smooth animations and clear feedback
4. **Error Handling**: Robust error management for blockchain operations

This foundation provides everything needed for a production-ready decentralized exchange, with room for expansion into more sophisticated DeFi features.
