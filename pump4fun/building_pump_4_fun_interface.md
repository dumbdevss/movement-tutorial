# PumpFun Frontend Tutorial - Complete TODO Implementation

## Introduction

This tutorial guides you through implementing the frontend interface for PumpFun, a decentralized token creation and swapping platform on the Movement blockchain. We'll work through all TODOs systematically to build a fully functional interface with token creation, swapping, and holder viewing capabilities

## Source Code Reference

The complete implementation of the frontend interface can be found in the repository:

**Repository:** [dumbdevss/pump-fun](https://github.com/dumbdevss/pump-fun)  
**Branch:** `frontend-integration`

## TODO 1: Setup Pinata and Pinata API key

1. **Create Pinata Account**
   - Go to [pinata.cloud](https://www.pinata.cloud/)
   - Sign up for free account

2. **Get API Keys**
   - Login → Go to "API Keys" section
   - Click "New Key"
   - Copy your API Key and API Secret

3. **Setup Environment Variables**
   - Create `.env.local` in your project root:
  
   ```bash
   NEXT_PUBLIC_PINATA_API_KEY=your_api_key_here
   NEXT_PUBLIC_PINATA_API_SECRET=your_secret_here
   NEXT_PUBLIC_PINATA_GATEWAY=https://gateway.pinata.cloud/ipfs/
   ```

4. **Update Your Code**

   ```javascript
   // Replace the empty strings with:
   const PINATA_API_KEY = process.env.NEXT_PUBLIC_PINATA_API_KEY
   const PINATA_API_SECRET = process.env.NEXT_PUBLIC_PINATA_API_SECRET
   const PINATA_GATEWAY = process.env.NEXT_PUBLIC_PINATA_GATEWAY || "https://gateway.pinata.cloud/ipfs/"
   ```

## TODO 2: Implement HandleChange in TokenCreator

Navigate `token-creator.tsx`, implement the form input handler:

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
// TODO 6: Connect form onSubmit handler
<form onSubmit={handleSubmit}>

TODO 7: Connect name input onChange
<input name="name" onChange={handleChange} />

// TODO 8: Connect symbol input onChange
<input name="symbol" onChange={handleChange} />

// TODO 9: Connect projectUrl input onChange
<input name="projectUrl" onChange={handleChange} />

// TODO 10: Connect description textarea onChange
<textarea name="description" onChange={handleChange} />

// TODO 11: Connect twitter input onChange
<input name="twitter" onChange={handleChange} />

// TODO 12: Connect discord input onChange
<input name="discord" onChange={handleChange} />

// TODO 13: Connect telegram input onChange
<input name="telegram" onChange={handleChange} />

// TODO 14: Connect supply input onChange
<input name="supply" onChange={handleChange} />

// TODO 15: Connect initialLiquidity input onChange
<input name="initialLiquidity" onChange={handleChange} />

// TODO 16: Connect image input onChange
<input type="file" onChange={handleImageChange} />

// TODO 17: Connect success modal close button
<button onClick={closeModal}>Close</button>

</form>
```

## TODO 18: Define GraphQL Query in TokenHolders

In `/component/token-holders.tsx`, define the query:

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
    skip: !tokenAddress, // Skip the query if no token address is provided
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

In `/component/token-swap-all.tsx`, define user balance query:

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
      last_transaction_timestampAdd commentMore actions
    }
  }
`;

```

## TODO 23: Implement UseQuery Hook in TokenSwapAll

Fetch user token balance:

```tsx
  const { data: tokenBalanceData, loading: tokenBalanceLoading, refetch: refetchTokenBalance } = useQuery(GET_USER_TOKEN_BALANCE, {
    variables: {
      owner_address: account?.address || "",
      asset_type: currentTokenAddress
    },
    skip: !account?.address || !currentTokenAddress,
    fetchPolicy: "network-only"
  });
```

## TODO 24: Implement HandleSwap in TokenSwapAll

Handle token swapping logic:

```tsx
const handleSwap = async () => {
    if (!fromAmount || parseFloat(fromAmount) <= 0) return;

    setIsLoading(true);
    try {
      const amount = parseFloat(fromAmount);

      if (fromToken === "MOVE") {
        // Swapping MOVE to token
        await swapMoveToToken(currentTokenAddress, (amount * 1e8));
      } else {
        // Swapping token to MOVE
        await swapTokensToMove(currentTokenAddress, (amount * 1e8));
      }

      // Clear form and refetch balances
      setFromAmount("");
      setToAmount("");
      refetchTokenBalance();
    } catch (error) {
      console.error("Swap failed:", error);
    } finally {
      setIsLoading(false);
    }
  };
```

## TODO 25: Connect Swap Button in TokenSwapAll

Connect the swap button to handleSwap:

```tsx
  <Button
            onClick={handleSwap}
            disabled={!fromAmount || isLoading || parseFloat(fromAmount) <= 0}
            className="w-full bg-gradient-to-r from-purple-500 to-pink-500 hover:from-purple-600 hover:to-pink-600"
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
  } = useView({
    moduleName: "pump_for_fun",
    functionName: "get_all_tokens",
  })
```

## TODO 27: Implement UseView Hook for History in Page

Fetch transaction history:

```tsx
  const {
    data: allHistoryData,
    refetch: refreshAllHistoryData,
  } = useView({
    moduleName: "pump_for_fun",
    functionName: "getAllHistory",
  })
```

## TODO 28: Implement CreateToken Function in Page

Handle token creation transaction:

```tsx
  const createToken = async (tokenName: string, tokenSymbol: string, icon_uri: string, project_uri: string, initial_liquidity: number, supply: string, description: string, telegram: string | null, twitter: string | null, discord: string | null) => {
    if (!account) {
      toast({
        title: "Please connect your wallet",
        description: "You need to connect your wallet to create a token",
        variant: "destructive",
      })
      return null
    }
    try {
      const response = await submitTransaction("create_token", [tokenName, tokenSymbol, parseInt(supply), description, telegram, twitter, discord, icon_uri, project_uri, initial_liquidity]);

      if (response as any) {
        toast({
          title: "Token created successfully",
          description: `You have successfully created the token ${tokenName} (${tokenSymbol})`,
          variant: "default",
        })
        refreshTokensData()
        return response;
      } else {
        toast({
          title: "Token creation failed",
          description: `There was an error creating the token ${tokenName} (${tokenSymbol})`,
          variant: "destructive",
        })
        return null;
      }
    } catch (error: any) {
      toast({
        title: "Token creation failed",
        description: `There was an error creating the token ${tokenName} (${tokenSymbol}): ${error.message}`,
        variant: "destructive",
      })
    }
  }
```

## TODO 29: Implement SwapTokensToMove Function in Page

Handle token to MOVE swapping:

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
      const response = await submitTransaction("swap_token_to_move", [token_addr as `0x${string}`, token_amount]);
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

## TODO 30: Implement SwapMoveToToken Function in Page

Handle MOVE to token swapping:

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
      const response = await submitTransaction("swap_move_to_token", [token_addr as `0x${string}`, move_amount]);

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
// TODO 32: 
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

                //TODO 33:
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

## Testing the Application

### 1. Running the Application Locally

Start the development server:

```bash
npm run dev
```

This will launch the frontend at [http://localhost:3000](http://localhost:3000).

### 2. Testing Features

- **Connect Wallet**: Ensure wallet connection works and displays the correct address.
- **Create Tokens**: Test creating tokens with and without images. Verify that new tokens appear in the token list.
- **View Token Holders**: Select a token and check that the holders list displays correct addresses and balances.
- **Swap MOVE ↔ Tokens**: Test swapping MOVE to tokens and vice versa. Confirm balances update accordingly.
- **Modals and Animations**: Open and close modals (e.g., token creation, swap, details) and verify smooth animations.
- **Error Handling**: Simulate errors (e.g., invalid input, failed transaction, upload issues) and check for appropriate user feedback.

## Key Notes

- Ensure Apollo Client is configured with correct GraphQL endpoint
- Verify wallet connectivity with Aptos network
- Check console logs for upload/transaction errors
- The `DECIMAL` constant (100000000) adjusts precision for blockchain
- All amounts are scaled by 1e8 for blockchain compatibility

Your PumpFun frontend is now complete with all TODOs implemented!
