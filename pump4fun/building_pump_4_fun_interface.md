# PumpFun Frontend Tutorial - Complete TODO Implementation

## Introduction

This tutorial guides you through implementing the frontend interface for PumpFun, a decentralized token creation and swapping platform on the Aptos blockchain. We'll work through all TODOs systematically to build a fully functional React/Next.js application with token creation, swapping, and holder viewing capabilities.

## Prerequisites

- Node.js (v16+), React, Next.js, TypeScript
- Apollo Client for GraphQL queries
- Pinata IPFS API keys for image storage
- Aptos Wallet Adapter
- Environment variables in `.env`:

```env
NEXT_PUBLIC_PINATA_API_KEY=your_api_key
NEXT_PUBLIC_PINATA_API_SECRET=your_api_secret
NEXT_PUBLIC_PINATA_GATEWAY=https://gateway.pinata.cloud
```

## TODO 1: Component Structure Setup

First, ensure your component files are properly structured:

- `token-creator.tsx` - Token creation form
- `token-holders.tsx` - Display token holders
- `token-swap-all.tsx` - Token swapping interface
- `page.tsx` - Main application page

## TODO 2: Implement HandleChange in TokenCreator

In `token-creator.tsx`, implement the form input handler:

```tsx
const handleChange = (e: React.ChangeEvent<HTMLInputElement | HTMLTextAreaElement>) => {
  const { name, value } = e.target
  setFormData((prev) => ({ ...prev, [name]: value }))
}
```

## TODO 3: Implement HandleImageChange in TokenCreator

Handle image upload to Pinata IPFS:

```tsx
const handleImageChange = async (e: React.ChangeEvent<HTMLInputElement>) => {
  if (e.target.files && e.target.files[0]) {
    const file = e.target.files[0]
    if (file) {
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
      
      const data = await response.json()
      const ipfsHash = data.IpfsHash
      const fileUrl = `${PINATA_GATEWAY}/ipfs/${ipfsHash}`
      
      setImageFile(file)
      setImagePreview(fileUrl)
    }
  }
}
```

## TODO 4: Implement HandleSubmit in TokenCreator

Handle form submission for token creation:

```tsx
const handleSubmit = async (e: React.FormEvent) => {
  e.preventDefault()
  setIsCreating(true)
  
  try {
    const response = await createToken(
      formData.name,
      formData.symbol,
      imagePreview || "",
      formData.projectUrl,
      DECIMAL * parseFloat(formData.initialLiquidity),
      formData.supply,
      formData.description,
      formData.telegram || null,
      formData.twitter || null,
      formData.discord || null
    )
    
    if (response) {
      setIsSuccess(true)
    }
  } catch (error) {
    console.error("Error creating token:", error)
  } finally {
    setIsCreating(false)
  }
}
```

## TODO 5: Implement CloseModal in TokenCreator

Reset form and close modal:

```tsx
const closeModal = () => {
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
  setImageFile(null)
  setImagePreview(null)
  setStep(1)
  setIsSuccess(false)
  onClose()
}
```

## TODO 6-17: Connect Form Elements in TokenCreator

Connect form handlers to JSX elements:

```tsx
// Form onSubmit
<form onSubmit={handleSubmit}>

// Input onChange handlers
<input name="name" onChange={handleChange} />
<input name="symbol" onChange={handleChange} />
<input name="projectUrl" onChange={handleChange} />
<textarea name="description" onChange={handleChange} />
<input name="twitter" onChange={handleChange} />
<input name="discord" onChange={handleChange} />
<input name="telegram" onChange={handleChange} />
<input name="supply" onChange={handleChange} />
<input name="initialLiquidity" onChange={handleChange} />

// Image input
<input type="file" onChange={handleImageChange} />

// Success modal close button
<button onClick={closeModal}>Close</button>
```

## TODO 18: Define GraphQL Query in TokenHolders

In `token-holders.tsx`, define the query:

```tsx
const GET_TOKEN_HOLDERS = gql`
  query GetTokenHolders($assetType: String!) {
    current_fungible_asset_balances(
      where: {asset_type: {_eq: $assetType}, amount: {_gt: 0}}
    ) {
      owner_address
      amount
      last_transaction_timestamp
      metadata {
        creator_address
      }
    }
  }
`
```

## TODO 19: Implement UseQuery Hook in TokenHolders

Fetch token holders data:

```tsx
const { loading, error, data } = useQuery(GET_TOKEN_HOLDERS, {
  variables: { assetType: tokenAddress },
  skip: !tokenAddress,
})
```

## TODO 20: Process Token Holders Data

Map query response to component format:

```tsx
const holders = data?.current_fungible_asset_balances?.map((holder: any, index: number) => ({
  id: index,
  address: holder.owner_address,
  amount: parseInt(holder.amount),
  timestamp: holder.last_transaction_timestamp,
})) || []
```

## TODO 21: Calculate Total Supply

Calculate total supply for percentage calculations:

```tsx
const totalSupply = holders.reduce((sum: number, holder: any) => sum + holder.amount, 0)
```

## TODO 22: Define GraphQL Query in TokenSwapAll

In `token-swap-all.tsx`, define user balance query:

```tsx
const GET_USER_TOKEN_BALANCE = gql`
  query GetUserTokenBalance($ownerAddress: String!, $assetType: String!) {
    current_fungible_asset_balances(
      where: {owner_address: {_eq: $ownerAddress}, asset_type: {_eq: $assetType}}
    ) {
      amount
    }
  }
`
```

## TODO 23: Implement UseQuery Hook in TokenSwapAll

Fetch user token balance:

```tsx
const { data: tokenBalanceData, loading: tokenBalanceLoading, refetch: refetchTokenBalance } = useQuery(
  GET_USER_TOKEN_BALANCE,
  {
    variables: { ownerAddress: account?.address, assetType: currentTokenAddress },
    skip: !account?.address || !currentTokenAddress,
  }
)
```

## TODO 24: Implement HandleSwap in TokenSwapAll

Handle token swapping logic:

```tsx
const handleSwap = async () => {
  if (!fromAmount || parseFloat(fromAmount) <= 0) return
  
  setIsLoading(true)
  try {
    const amount = parseFloat(fromAmount) * 1e8 // Convert to blockchain precision
    
    if (fromToken === "MOVE") {
      await swapMoveToToken(currentTokenAddress, amount)
    } else {
      await swapTokensToMove(currentTokenAddress, amount)
    }
    
    setFromAmount("")
    setToAmount("")
    refetchTokenBalance()
  } catch (error) {
    console.error("Error swapping tokens:", error)
  } finally {
    setIsLoading(false)
  }
}
```

## TODO 25: Connect Swap Button in TokenSwapAll

Connect the swap button to handleSwap:

```tsx
<Button
  onClick={handleSwap}
  disabled={!fromAmount || isLoading || parseFloat(fromAmount) <= 0}
  className="w-full bg-gradient-to-r from-purple-500 to-pink-500"
>
  {isLoading ? (
    <>
      <RefreshCw className="mr-2 h-4 w-4 animate-spin" />
      Swapping...
    </>
  ) : (
    "Swap Tokens"
  )}
</Button>
```

## TODO 26: Implement UseView Hook for Tokens in Page

In `page.tsx`, fetch all tokens:

```tsx
const {
  data: tokensData,
  refetch: refreshTokensData,
} = useView("get_all_tokens", {})
```

## TODO 27: Implement UseView Hook for History in Page

Fetch transaction history:

```tsx
const {
  data: allHistoryData,
  refetch: refreshAllHistoryData,
} = useView("get_all_history", {})
```

## TODO 28: Implement CreateToken Function in Page

Handle token creation transaction:

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
  if (!account) {
    toast({ title: "Error", description: "Please connect your wallet", variant: "destructive" })
    return null
  }
  
  try {
    const response = await submitTransaction({
      function: "create_token",
      arguments: [
        tokenName,
        tokenSymbol,
        icon_uri,
        project_uri,
        initial_liquidity,
        supply,
        description,
        telegram,
        twitter,
        discord,
      ],
    })
    
    toast({ title: "Success", description: "Token created successfully!" })
    refreshTokensData()
    return response
  } catch (error) {
    console.error("Error creating token:", error)
    toast({ title: "Error", description: "Failed to create token", variant: "destructive" })
    return null
  }
}
```

## TODO 29: Implement SwapTokensToMove Function in Page

Handle token to MOVE swapping:

```tsx
const swapTokensToMove = async (token_addr: string, token_amount: number) => {
  if (!account) {
    toast({ title: "Error", description: "Please connect your wallet", variant: "destructive" })
    return
  }
  
  try {
    await submitTransaction({
      function: "swap_tokens_to_move",
      arguments: [token_addr, token_amount],
    })
    
    toast({ title: "Success", description: "Swap completed successfully!" })
    refreshTokensData()
  } catch (error) {
    console.error("Error swapping tokens:", error)
    toast({ title: "Error", description: "Failed to swap tokens", variant: "destructive" })
  }
}
```

## TODO 30: Implement SwapMoveToToken Function in Page

Handle MOVE to token swapping:

```tsx
const swapMoveToToken = async (token_addr: string, move_amount: number) => {
  if (!account) {
    toast({ title: "Error", description: "Please connect your wallet", variant: "destructive" })
    return
  }
  
  try {
    await submitTransaction({
      function: "swap_move_to_token",
      arguments: [token_addr, move_amount],
    })
    
    toast({ title: "Success", description: "Swap completed successfully!" })
    refreshTokensData()
  } catch (error) {
    console.error("Error swapping tokens:", error)
    toast({ title: "Error", description: "Failed to swap tokens", variant: "destructive" })
  }
}
```

## TODO 31: Implement HandleTokenClick Function in Page

Handle token selection for details modal:

```tsx
const handleTokenClick = (token: Token) => {
  setSelectedToken(token)
}
```

## TODO 32-33: Connect TokenCard Click Events in Page

Connect TokenCard components to handleTokenClick:

```tsx
{refinedTokenData.map((token, index) => (
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

## TODO 34-35: Connect TokenSwapAll Props in Page

Pass swap functions to TokenSwapAll component:

```tsx
<TokenSwapAll 
  tokens={refinedTokenData} 
  swapMoveToToken={swapMoveToToken}
  swapTokensToMove={swapTokensToMove}
/>
```

## TODO 36: Connect TokenSwap Props in Modal

Pass swap functions to TokenSwap in the modal:

```tsx
<TokenSwap 
  selectedToken={selectedToken} 
  tokens={refinedTokenData} 
  swapMoveToToken={swapMoveToToken}
  swapTokensToMove={swapTokensToMove}
/>
```

## TODO 37: Connect TokenCreator Props

Pass createToken function to TokenCreator:

```tsx
<TokenCreator 
  onClose={() => setShowCreateModal(false)} 
  createToken={createToken} 
/>
```

## Testing & Deployment

1. **Run locally**: `npm run dev`
2. **Test features**:
   - Create tokens with images
   - View token holders
   - Swap MOVE ↔ tokens
   - Verify modals and animations

3. **Deploy**:
   - Build: `npm run build`
   - Deploy to Vercel: `vercel --prod`

## Key Notes

- Ensure Apollo Client is configured with correct GraphQL endpoint
- Verify wallet connectivity with Aptos network
- Check console logs for upload/transaction errors
- The `DECIMAL` constant (100000000) adjusts precision for blockchain
- All amounts are scaled by 1e8 for blockchain compatibility

Your PumpFun frontend is now complete with all TODOs implemented!
