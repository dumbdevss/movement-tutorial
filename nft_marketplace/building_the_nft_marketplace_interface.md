# Building the Frontend for Your NFT Marketplace

This guide provides a structured walkthrough for building the frontend of an NFT marketplace, integrating a React-based interface with a Move smart contract on the Aptos blockchain. It covers NFT image storage using IPFS via Pinata, enabling users to upload images, create or select collections, and mint NFTs. Each implementation step is explained, with reasons for the chosen approaches where necessary, following the provided TODOs.

## Understanding the Architecture

The NFT marketplace frontend enables users to interact with a Move smart contract on the Aptos blockchain to create, browse, and manage NFTs. The interface is built with React (Next.js), styled with Tailwind CSS, and uses Scaffold-Move hooks for blockchain interactions and Pinata for IPFS storage.

**Tech Stack:**

- **Move Smart Contract**: Handles NFT creation, collection management, and ownership on Aptos.
- **IPFS (via Pinata)**: Stores NFT images off-chain, with IPFS hashes stored on-chain for cost efficiency.
- **React Frontend**: Built with Next.js and Tailwind CSS for a responsive, user-friendly interface.
- **Scaffold Hooks**: Simplifies blockchain interactions with `useView` and `useSubmitTransaction`.
- **Aptos Wallet Adapter**: Facilitates wallet connections for transaction signing.

## Source Code Reference

The complete implementation of the frontend interface can be found in the repository:

**Repository:** [dumbdevss/document-management](https://github.com/dumbdevss/document-management)  
**Branch:** `frontend-integration`

## Why IPFS for NFT Storage?

Storing large image files on-chain is cost-prohibitive due to high gas fees. IPFS (InterPlanetary File System) with Pinata provides a decentralized, cost-effective solution:

- **Cost Efficiency**: Only the IPFS hash (a small string) is stored on-chain, reducing costs.
- **Decentralization**: Files are distributed across IPFS nodes, ensuring availability without reliance on a single server.
- **Content Addressing**: IPFS hashes uniquely identify files, ensuring data integrity.
- **Pinata Integration**: Pinata pins files to ensure persistent availability, simplifying IPFS management.

This approach balances blockchain immutability (storing hashes) with off-chain scalability (storing images).

---

## CreateNFT Page Implementation (`/app/create-nft`)

The `CreateNFT` page (`app/create-nft/page.tsx`) allows users to upload an image, enter NFT details, select or create a collection, and mint the NFT. Below are the implementations for the specified TODOs.

### TODO 1: Fetch NFTs for Sale with `useView` Hook

The `useView` hook retrieves NFTs listed for sale from the blockchain without gas fees, ideal for read-only queries.

```javascript
const { data, error, isLoading } = useView({
  moduleName: "NFTMarketplace",
  functionName: "get_nfts_for_sale",
})
```

**Implementation Notes:**:

- **Purpose**: Fetches all NFTs marked as `for_sale` to display in the `FeaturedNFTs` component on the `Home` page.
- **Why `useView`?**: It abstracts complex blockchain queries, providing a simple React hook interface with automatic state management (`data`, `error`, `isLoading`).
- **Data Handling**: `data?.[0] as NFT[] || []` ensures safe parsing, defaulting to an empty array if `data` is undefined.
- **Usage**: The fetched NFTs are stored in `nftsForSale` and rendered in a visually appealing grid.

### TODO 2: Configure Pinata API Keys

Securely configure Pinata API credentials for IPFS uploads using environment variables.

```javascript
const PINATA_API_KEY = process.env.NEXT_PUBLIC_PINATA_API_KEY
const PINATA_API_SECRET = process.env.NEXT_PUBLIC_PINATA_API_SECRET
const PINATA_GATEWAY = process.env.NEXT_PUBLIC_PINATA_GATEWAY || "https://gateway.pinata.cloud/ipfs/"
```

**Implementation Notes:**:

- **Purpose**: Enables secure image uploads to IPFS via Pinata's API.
- **Why Environment Variables?**: Hardcoding API keys in source code risks exposure; environment variables (in `.env`) ensure security and flexibility across environments.
- **Setup Steps**:
  1. Sign up at [Pinata.cloud](https://pinata.cloud).
  2. Obtain API key and secret from the Pinata dashboard.
  3. Add to `.env`:

     ```env
     NEXT_PUBLIC_PINATA_API_KEY=your_api_key
     NEXT_PUBLIC_PINATA_API_SECRET=your_api_secret
     NEXT_PUBLIC_PINATA_GATEWAY=https://gateway.pinata.cloud/ipfs/
     ```

- **Gateway URL**: The default Pinata gateway ensures reliable access to uploaded files.

### TODO 3: Use `useSubmitTransaction` Hook

The `useSubmitTransaction` hook simplifies blockchain transaction submissions.

```javascript
const { submitTransaction, transactionInProcess, transactionResponse } = useSubmitTransaction("NFTMarketplace")
```

**Implementation Notes:**:

- **Purpose**: Provides functions and state for submitting transactions (e.g., minting NFTs).
- **Why `useSubmitTransaction`?**: It abstracts wallet signing and transaction logic, reducing boilerplate and ensuring compatibility with the Aptos Wallet Adapter.
- **Returns**:
  - `submitTransaction`: Submits transactions to the `NFTMarketplace` module.
  - `transactionInProcess`: Tracks transaction status for UI feedback.
  - `transactionResponse`: Provides transaction results (e.g., hash) for user confirmation.

### TODO 4: Fetch User Collections with `useView` Hook

Fetch the user's existing collections to allow selection during NFT creation.

```javascript
const { data, error, isLoading, refetch } = useView({
  moduleName: "NFTMarketplace",
  functionName: "get_all_collections_by_user",
  args: [account?.address as `0x${string}`, 10, 0],
})
```

**Implementation Notes:**:

- **Purpose**: Retrieves the user's collections for display in a dropdown or list.
- **Why `useView`?**: Ensures gas-free data retrieval and provides state management for loading and errors.
- **Arguments**:
  - `account?.address`: The connected wallet's address.
  - `10`: Limits results to 10 collections.
  - `0`: Starts at the first collection (pagination offset).
- **Data Handling**: `data?.[0] || []` ensures safe parsing, storing results in `collections`.

### TODO 5: Fetch Minted NFT Data with `useView` Hook

Fetch details of a newly minted NFT to display in a receipt.

```javascript
const { 
  data: fetchedNFTData, 
  refetch: refetchNFT 
} = useView({
  moduleName: "NFTMarketplace",
  functionName: "get_nft_by_collection_name_and_token_name",
  args: [
    mintedNFTInfo?.collection || "",
    mintedNFTInfo?.name || "",
    account?.address as `0x${string}` || ""
  ],
})
```

**Implementation Notes:**:

- **Purpose**: Retrieves the minted NFT's details post-minting for user confirmation.
- **Why `useView`?**: Provides a gas-free way to query specific NFT data.
- **Arguments**: Uses `mintedNFTInfo` (set after minting) to specify the collection, NFT name, and owner address.
- **Dynamic Fetching**: Only runs when `mintedNFTInfo` is populated, triggered by `useEffect` to update `mintedNFT`.

### TODO 6: Implement `handleImageChange` Function

Handle image uploads to IPFS and update the UI with a preview.

```javascript
const handleImageChange = async (e: React.ChangeEvent<HTMLInputElement>) => {
  setUploadingFile(true)
  const file = e.target.files?.[0]
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
    const imageUrl = URL.createObjectURL(file)
    setPreviewImage(imageUrl)
    setFileUrl(fileUrl)
    setUploadingFile(false)
  }
}
```

**Implementation Notes:**:

- **Purpose**: Uploads images to IPFS via Pinata and generates a local preview URL.
- **Why Pinata?**: Simplifies IPFS uploads with a reliable API and ensures files are pinned for accessibility.
- **Steps**:
  1. Set `uploadingFile` to show a loading indicator.
  2. Append the file to `FormData` for the API request.
  3. Send a POST request to Pinata's pinning endpoint.
  4. Construct `fileUrl` using the IPFS hash and gateway.
  5. Create a local `imageUrl` for immediate preview.
  6. Update state and clear loading indicator.
- **Error Handling**: Assumes valid files for simplicity; production apps should validate file size and type.

### TODO 7: Implement `mintNFT` Function

Submit the NFT minting transaction and handle the response.

```javascript
const mintNFT = async (e: React.FormEvent) => {
  e.preventDefault()

  try {
    const finalCollectionName = collectionChoice === "new" ? newCollectionData.name : nftData.collection
    const collectionDescription = collectionChoice === "new" ? newCollectionData.description : "null"
    const collectionUri = collectionChoice === "new" ? newCollectionData.uri : "null"
    const finalTokenName = nftData.name

    let response = await submitTransaction("mint_nft", [
      finalCollectionName,
      collectionDescription,
      collectionUri,
      finalTokenName,
      nftData.description,
      nftData.category,
      fileUrl
    ])
    
    setHash(response)
    
    setMintedNFTInfo({
      collection: finalCollectionName,
      name: finalTokenName
    })
    
    setTimeout(async () => {
      await refetchNFT()
      setShowReceipt(true)
    }, 2000)
  } catch (error) {
    console.error("Failed to mint NFT:", error)
  }
}
```

**Implementation Notes:**:

- **Purpose**: Mints an NFT by submitting a transaction to the `NFTMarketplace` module.
- **Why `submitTransaction`?**: Abstracts wallet signing and transaction submission, ensuring secure and user-friendly minting.
- **Steps**:
  1. Determine collection data based on user choice (new or existing collection).
  2. Call `submitTransaction` with `mint_nft` and required arguments.
  3. Store the transaction hash in `hash`.
  4. Update `mintedNFTInfo` to trigger NFT data fetching.
  5. Wait 2 seconds for blockchain confirmation, then fetch NFT data and show receipt.
- **Error Handling**: Logs errors; production apps should add user-facing feedback.

### TODO 8: Connect `handleImageChange` to File Input

Connect the image upload handler to the file input.

```javascript
<input
  id="image"
  type="file"
  className="absolute inset-0 opacity-0 cursor-pointer"
  accept="image/*"
  onChange={handleImageChange}
  required
/>
```

**Implementation Notes:**:

- **Purpose**: Triggers `handleImageChange` when a file is selected.
- **Why Hidden Input?**: Overlaid on a styled button for better UX, with `opacity-0` and `cursor-pointer`.
- **Attributes**:
  - `accept="image/*"`: Restricts to image files.
  - `required`: Ensures an image is uploaded before submission.

### TODO 9: Display "No Collections" Message Conditionally

Show a warning if the user has no collections.

```javascript
{collections.length === 0 && !isLoading && (
  <p className="text-sm text-amber-500">
    No existing collections found. Create a new collection first.
  </p>
)}
```

**Implementation Notes:**:

- **Purpose**: Guides users to create a collection if none exist.
- **Why Conditional?**: Only shows when `collections` is empty and fetching is complete (`!isLoading`).
- **Styling**: Uses `text-amber-500` for a clear, non-intrusive warning.

### TODO 10: Disable Button During Transaction

Prevent multiple submissions by disabling the submit button during transactions.

```javascript
<Button
  type="submit"
  className="w-full bg-gradient-to-r from-purple-600 to-pink-600 hover:from-purple-700 hover:to-pink-700"
  disabled={transactionInProcess || uploadingFile || !fileUrl || (collectionChoice === "existing" && !nftData.collection)}
>
  {transactionInProcess ? (
    <>
      <Loader2 className="mr-2 h-4 w-4 animate-spin" />
      Minting NFT...
    </>
  ) : uploadingFile ? (
    <>
      <Loader2 className="mr-2 h-4 w-4 animate-spin" />
      Uploading...
    </>
  ) : (
    "Create NFT"
  )}
</Button>
```

**Implementation Notes:**:

- **Purpose**: Enhances UX by preventing duplicate submissions and showing progress.
- **Why Disable?**: Avoids user errors during critical operations (uploading or minting).
- **Conditions**:
  - `transactionInProcess`: Transaction is processing.
  - `uploadingFile`: Image is uploading.
  - `!fileUrl`: No image uploaded.
  - `collectionChoice === "existing" && !nftData.collection`: No collection selected.
- **Dynamic Text**: Shows appropriate loading messages or "Create NFT".

### Connecting Form Submission

Link the `mintNFT` function to the form's `onSubmit` event.

```javascript
<form onSubmit={mintNFT} className="space-y-6">
```

**Implementation Notes:**:

- **Purpose**: Triggers `mintNFT` when the form is submitted.
- **Why Form?**: Combines all inputs (image, details, collection) into a single submission action.
- **Validation**: `required` attributes on inputs ensure basic validation.

---

## Marketplace Page Implementation (`/app/marketplace`)

The `Marketplace` page (`app/marketplace/page.tsx`) allows users to browse NFTs for sale, filter by criteria, and sort results.

### TODO 11: Fetch Collections with `useView` Hook

Fetch all collections to populate the collection filter.

```javascript
const {
  data: collectionsData,
  error: collectionsError,
  isLoading: isLoadingCollections,
  refetch: refetchCollections
} = useView({
  moduleName: "NFTMarketplace",
  functionName: "get_all_collections",
  args: [10, 0],
})
```

**Implementation Notes:**:

- **Purpose**: Retrieves collections for filtering NFTs in the `NFTFilters` component.
- **Why `useView`?**: Gas-free and provides state management for UI updates.
- **Arguments**: Limits to 10 collections with no offset for simplicity.

### TODO 12: Fetch NFTs for Sale with `useView` Hook

Fetch all NFTs listed for sale.

```javascript
const {
  data: nftsData,
  error: nftsError,
  isLoading: isLoadingNFTs,
  refetch: refetchNFTs
} = useView({
  moduleName: "NFTMarketplace",
  functionName: "get_nfts_for_sale",
})
```

**Implementation Notes:**:

- **Purpose**: Retrieves NFTs for display in the `NFTGrid` component.
- **Why No Arguments?**: `get_nfts_for_sale` fetches all listed NFTs without filtering on-chain.

### TODO 13: Implement NFT Filtering and Sorting Logic

Filter and sort NFTs based on user input.

```javascript
useEffect(() => {
  if (!allNFTs.length) {
    setFilteredNFTs([])
    return
  }

  let filtered = [...allNFTs]

  // Apply search filter
  if (searchQuery && searchQuery.trim() !== "") {
    filtered = filtered.filter(
      (nft) =>
        nft.name.toLowerCase().includes(searchQuery.toLowerCase()) ||
        nft.collection_name.toLowerCase().includes(searchQuery.toLowerCase()) ||
        nft.owner.toLowerCase().includes(searchQuery.toLowerCase().replace("0x", ""))
    )
  }

  // Apply category filters
  const activeCategoryFilters = Object.entries(categoryFilters).filter(([_, isActive]) => isActive)
  if (activeCategoryFilters.length > 0) {
    filtered = filtered.filter(nft => {
      const category = nft.category.toLowerCase()
      return activeCategoryFilters.some(([key, _]) => {
        return category === key
      })
    })
  }

  // Apply status filters
  if (statusFilters.buyNow) {
    filtered = filtered.filter(nft => nft.sale_type === 0 && nft.for_sale)
  }
  if (statusFilters.onAuction) {
    filtered = filtered.filter(nft => nft.sale_type === 1 && nft.for_sale)
  }
  if (statusFilters.newItem) {
    const oneDayAgo = Date.now() - 86400000
    filtered = filtered.filter(nft => nft.created_at > oneDayAgo)
  }
  if (statusFilters.hasOffers) {
    filtered = filtered.filter(nft => nft.auction && nft.auction?.vec?.length > 0)
  }

  // Apply collection filters
  const activeCollectionFilters = Object.entries(collectionFilters).filter(([_, isActive]) => isActive)
  if (activeCollectionFilters.length > 0) {
    filtered = filtered.filter(nft => {
      const collection = nft.collection_name.toLowerCase().replace(/\s+/g, '')
      return activeCollectionFilters.some(([key, _]) => {
        return collection.includes(key?.toLowerCase().replace(/\s+/g, ''))
      })
    })
  }

  // Apply price range filter
  filtered = filtered.filter(nft => (nft.price / 100000000) >= priceRange[0] && (nft.price / 100000000) <= priceRange[1])

  // Apply sorting
  switch (sortBy) {
    case "price-low":
      filtered.sort((a, b) => a.price - b.price)
      break
    case "price-high":
      filtered.sort((a, b) => b.price - a.price)
      break
    case "recent":
    default:
      filtered.sort((a, b) => b.created_at - a.created_at)
      break
  }

  setFilteredNFTs(filtered)
}, [allNFTs, searchQuery, categoryFilters, statusFilters, collectionFilters, priceRange, sortBy])
```

**Implementation Notes:**:

- **Purpose**: Filters and sorts NFTs based on user-selected criteria for a personalized browsing experience.
- **Why `useEffect`?**: Re-runs filtering/sorting logic when inputs change, ensuring real-time updates.
- **Steps**:
  1. Copy `allNFTs` to avoid mutating original data.
  2. Apply search filter (case-insensitive) on NFT name, collection, or owner.
  3. Filter by active categories, status (buy now, auction, new, has offers), and collections.
  4. Filter by price range (converted to APT).
  5. Sort by price (low/high) or recency.
- **Performance**: Uses `useMemo` for `allNFTs` to optimize re-renders.

### TODO 14: Implement Retry Functionality

Add a retry button for failed data fetches.

```javascript
<button
  onClick={() => {
    refetchCollections();
    refetchNFTs();
  }}
  className="mt-4 px-4 py-2 bg-primary text-white rounded-md hover:bg-primary/80"
>
  Retry
</button>
```

**Implementation Notes:**:

- **Purpose**: Allows users to retry fetching collections and NFTs after errors.
- **Why Refetch?**: Provides a user-friendly way to recover without page refresh.
- **Styling**: Consistent Tailwind classes ensure a professional look.

---

## NFTDetail Page Implementation (`/app/nft/[id]`)

The `NFTDetail` page (`app/nft/[id]/page.tsx`) displays details for a specific NFT and supports interactions like purchasing, bidding, transferring, and resolving auctions.

### TODO 15: Use `useSubmitTransaction` Hook

```javascript
const { submitTransaction, transactionInProcess } = useSubmitTransaction("NFTMarketplace")
```

**Implementation Notes:**:

- **Purpose**: Enables transaction submissions for purchase, bid, transfer, and auction resolution.
- **Why `useSubmitTransaction`?**: Simplifies complex blockchain interactions with a single hook.

### TODO 16: Fetch NFT Details with `useView` Hook

```javascript
const {
  data: nftRawData,
  refetch: refetchNFT,
  isLoading
} = useView({
  moduleName: "NFTMarketplace",
  functionName: "get_nft_details",
  args: [parseInt(id as string)],
})
```

**Implementation Notes:**:

- **Purpose**: Fetches detailed NFT data by ID for display.
- **Why `useView`?**: Gas-free and efficient for single NFT queries.
- **Arguments**: Parses URL `id` to an integer for the `get_nft_details` function.

### TODO 17: Implement `handlePurchase` Function

```javascript
const handlePurchase = async () => {
  try {
    setTransactionType("purchase");
    const response = await submitTransaction("purchase_nft", [
      parseInt(id as string),
    ]);
    setHash(response);
    setShowBuyDialog(false);
    setTimeout(async () => {
      await refetchNFT();
      setShowReceipt(true);
    }, 2000);
  } catch (error) {
    console.error("Failed to purchase NFT:", error);
  }
}
```

**Implementation Notes:**:

- **Purpose**: Executes a direct purchase of an NFT.
- **Steps**:
  1. Set transaction type for receipt.
  2. Submit `purchase_nft` transaction with NFT ID.
  3. Store hash, close dialog, and refresh data after 2 seconds.
- **Why Delay?**: Ensures blockchain confirmation before updating UI.

### TODO 18: Implement `handlePlaceOffer` Function

```javascript
const handlePlaceOffer = async () => {
  try {
    setShowOfferDialog(false);
    setTransactionType("offer");
    const offerAmountInSmallestUnits = Math.floor(offerAmount * 100000000);
    const response = await submitTransaction("place_offer", [
      parseInt(id as string),
      offerAmountInSmallestUnits,
    ]);
    setHash(response);
    setTimeout(async () => {
      await refetchNFT();
      setShowReceipt(true);
    }, 2000);
  } catch (error) {
    console.error("Failed to place offer:", error);
  }
}
```

**Implementation Notes:**:

- **Purpose**: Places a bid or offer on an NFT.
- **Steps**:
  1. Convert offer amount to smallest units (10^8).
  2. Submit `place_offer` transaction with NFT ID and amount.
  3. Handle dialog, hash, and data refresh.

### TODO 19: Implement `handleTransferNFT` Function

```javascript
const handleTransferNFT = async () => {
  try {
    setShowTransferDialog(false);
    setTransactionType("transfer");
    const response = await submitTransaction("transfer_nft", [
      parseInt(id as string),
      transferAddress,
    ]);
    setHash(response);
    setTimeout(async () => {
      await refetchNFT();
      setShowReceipt(true);
    }, 2000);
  } catch (error) {
    console.error("Failed to transfer NFT:", error);
  }
}
```

**Implementation Notes:**:

- **Purpose**: Transfers an NFT to another address.
- **Steps**:
  1. Submit `transfer_nft` transaction with NFT ID and recipient address.
  2. Handle dialog, hash, and data refresh.

### TODO 20: Implement `handleResolveAuction` Function

```javascript
const handleResolveAuction = async () => {
  try {
    setTransactionType("resolve");
    const response = await submitTransaction("finalize_auction", [
      parseInt(id as string),
    ]);
    setHash(response);
    setTimeout(async () => {
      await refetchNFT();
      setShowReceipt(true);
    }, 2000);
  } catch (error) {
    console.error("Failed to resolve auction:", error);
  }
}
```

**Implementation Notes:**:

- **Purpose**: Finalizes an auction after it ends.
- **Steps**: Submit `finalize_auction` transaction with NFT ID and handle response.

### TODO 21: Implement `handleLike` Function

```javascript
const handleLike = () => {
  setLike(!like);
  toast({
    title: like ? 'Unliked' : 'Liked',
    description: like ? "You unliked this NFT" : "You liked this NFT",
    variant: 'default',
    className: 'bg-green-700 text-foreground',
  });
}
```

**Implementation Notes:**:

- **Purpose**: Toggles the like state and shows feedback.
- **Why Toast?**: Provides immediate, non-intrusive user feedback.

### TODO 22: Connect `handleLike` to Like Button

```javascript
<Button onClick={handleLike} variant="outline" size="icon">
  <Heart className={`h-4 w-4 ${like && 'fill-red-700'}`} />
</Button>
```

**Implementation Notes:**:

- **Purpose**: Triggers `handleLike` on click with dynamic heart styling.
- **Why Outline Variant?**: Subtle design for secondary actions.

### TODO 23: Connect `handleResolveAuction` to Resolve Auction Button

```javascript
<Button
  onClick={handleResolveAuction}
  disabled={transactionInProcess}
  className="bg-gradient-to-r from-purple-600 to-pink-600 hover:from-purple-700 hover:to-pink-700"
>
  Resolve Auction
</Button>
```

**Implementation Notes:**:

- **Purpose**: Triggers auction resolution with disabled state during transactions.
- **Why Disabled?**: Prevents multiple submissions.

### TODO 24: Connect `handlePurchase` to Buy Dialog Button

```javascript
<Button
  className="w-full bg-gradient-to-r from-purple-600 to-pink-600 hover:from-purple-700 hover:to-pink-700"
  onClick={handlePurchase}
  disabled={transactionInProcess}
>
  {transactionInProcess ? "Processing..." : "Confirm Purchase"}
</Button>
```

**Implementation Notes:**:

- **Purpose**: Triggers purchase with dynamic text and disabled state.

### TODO 25: Update `transactionInProcess` in Offer Dialog Button

```javascript
<Button
  className="w-full bg-gradient-to-r from-purple-600 to-pink-600 hover:from-purple-700 hover:to-pink-700"
  onClick={handlePlaceOffer}
  disabled={transactionInProcess || (isAuction && offerAmount <= nft.price) || offerAmount <= 0}
>
  {transactionInProcess ? "Processing..." : `Confirm ${isAuction ? "Bid" : "Offer"}`}
</Button>
```

**Implementation Notes:**:

- **Purpose**: Ensures valid bids/offers with disabled state for invalid inputs or ongoing transactions.

### TODO 26: Connect `handlePlaceOffer` to Offer Dialog Button

 Reused from TODO 25.

### TODO 27: Update `transactionInProcess` in Transfer Dialog Button

```javascript
<Button
  className="w-full bg-gradient-to-r from-purple-600 to-pink-600 hover:from-purple-700 hover:to-pink-700"
  onClick={handleTransferNFT}
  disabled={transactionInProcess || !transferAddress.startsWith('0x') || transferAddress.length < 10}
>
  {transactionInProcess ? "Processing..." : "Confirm Transfer"}
</Button>
```

**Implementation Notes:**:

- **Purpose**: Validates transfer address and disables button during transactions.

### TODO 28: Connect `handleTransferNFT` to Transfer Dialog Button

 Reused from TODO 27.

---

## UserPortfolio Page Implementation

The `UserPortfolio` page displays the user's owned NFTs and collections, with options to list or cancel sales.

### TODO 30: Fetch User's NFTs with `useView` Hook

```javascript
const {
  data: nftsData,
  error: nftsError,
  isLoading: isLoadingNFTs,
  refetch: refetchNFTs
} = useView({
  moduleName: "NFTMarketplace",
  functionName: "get_user_nfts",
  args: [account?.address as `0x${string}`]
})
```

**Implementation Notes:**:

- **Purpose**: Fetches NFTs owned by the connected wallet.
- **Why `useView`?**: Efficient for read-only queries with state management.

### TODO 31: Implement `handleListNFT` Function

```javascript
const handleListNFT = async () => {
  if (!selectedNFT) {
    toast({
      title: "No NFT Selected",
      description: "Please select an NFT to list",
      variant: "destructive"
    })
    return
  }

  try {
    if (saleType === SaleType.AUCTION && (!auctionMinBid || !auctionDeadline)) {
      throw new Error("Auction minimum bid and deadline are required")
    }

    const price = parseFloat(listingPrice)
    if ((isNaN(price) || price <= 0) && saleType === SaleType.INSTANT) {
      throw new Error("Invalid listing price")
    }

    const auctionMinPrice = parseFloat(auctionMinBid)
    if ((isNaN(auctionMinPrice) || auctionMinPrice <= 0) && saleType === SaleType.AUCTION) {
      throw new Error("Invalid auction minimum bid")
    }

    const priceInOctas = Math.floor(saleType === SaleType.INSTANT ? price * 100000000 : (auctionMinPrice * 100000000))
    const deadlineTimestamp = auctionDeadline
      ? Math.floor(new Date(auctionDeadline).getTime() / 1000)
      : null

    if (saleType === SaleType.AUCTION && deadlineTimestamp && deadlineTimestamp <= Math.floor(Date.now() / 1000)) {
      throw new Error("Auction deadline must be in the future")
    }

    await submitTransaction("list_nft_for_sale", [
      parseInt(selectedNFT.id),
      priceInOctas,
      saleType === SaleType.INSTANT ? 0 : 1,
      deadlineTimestamp
    ])

    toast({
      title: "NFT Listed Successfully",
      description: `${selectedNFT.name} has been listed for ${saleType === SaleType.INSTANT ? price : auctionMinBid} MOVE`,
      variant: "default"
    })

    setIsListingDialogOpen(false)
    setListingPrice("")
    setAuctionMinBid("")
    setAuctionDeadline("")
    setSelectedNFT(null)
    setSaleType(SaleType.INSTANT)

    setTimeout(() => {
      refetchNFTs()
    }, 2000)
  } catch (error: any) {
    console.error("Failed to list NFT:", error)
    toast({
      title: "Failed to List NFT",
      description: error.message || "An error occurred while listing your NFT",
      variant: "destructive"
    })
  }
}
```

**Implementation Notes:**:

- **Purpose**: Lists an NFT for sale (instant or auction).
- **Steps**:
  1. Validate NFT selection and input fields.
  2. Convert price/bid to octas and deadline to timestamp.
  3. Submit `list_nft_for_sale` transaction.
  4. Show success toast and reset form.
- **Why Validation?**: Prevents invalid transactions, enhancing user trust.

### TODO 32: Implement `handleCancelListing` Function


```javascript
const handleCancelListing = async (nft: NFT) => {
  if (!nft || !nft.id) {
    toast({
      title: "No NFT Selected",
      description: "Please select an NFT to cancel listing",
      variant: "destructive"
    })
    return
  }

  if (!nft.for_sale) {
    toast({
      title: "NFT Not Listed",
      description: "This NFT is not currently listed for sale",
      variant: "destructive"
    })
    return
  }

  try {
    const nftId = typeof nft.id === 'string' ? parseInt(nft.id, 10) : nft.id
    if (isNaN(nftId)) {
      throw new Error("Invalid NFT ID format")
    }

    const txResult = await submitTransaction("cancel_listing", [nftId])

    if (txResult) {
      toast({
        title: "NFT Listing Cancelled",
        description: `${nft.name} listing has been cancelled successfully`,
        variant: "default",
        action: (
          <Link
            href={`http://explorer.movementnetwork.xyz/txn/${txResult}`}
            target="_blank"
            className="text-blue-500 hover:underline"
          >
            View Transaction
          </Link>
        )
      })
    }

    setTimeout(() => {
      refetchNFTs()
    }, 2000)
  } catch (error) {
    console.error("Failed to cancel listing:", error)
    const errorMessage = error instanceof Error
      ? error.message
      : "An unknown error occurred while cancelling the listing"
    toast({
      title: "Failed to Cancel Listing",
      description: errorMessage,
      variant: "destructive"
    })
  }
}
```

**Implementation Notes:**:

- **Purpose**: Cancels an NFT listing.
- **Steps**:
  1. Validate NFT and listing status.
  2. Submit `cancel_listing` transaction.
  3. Show success toast with transaction link.
- **Why Transaction Link?**: Enhances transparency by linking to the blockchain explorer.

### TODO 33: Implement `openListingDialog` Function

```javascript
const openListingDialog = (nft: NFT) => {
  setSelectedNFT(nft)
  setIsListingDialogOpen(true)
}
```

**Implementation Notes:**:

- **Purpose**: Opens the listing dialog for a selected NFT.
- **Why Simple?**: Minimal logic ensures fast and reliable dialog triggering.

### TODO 34: Create Dynamic NFT Status Badge

```javascript
const getNftStatusBadge = (nft: NFT) => {
  if (nft.sale_type === 0 && nft.for_sale) {
    return (
      <Badge className="bg-green-100 text-green-800 whitespace-nowrap hover:bg-green-200">
        For Sale
      </Badge>
    )
  } else if (nft.sale_type === 1 && nft.for_sale) {
    return (
      <Badge className="bg-green-100 text-green-800 whitespace-nowrap hover:bg-green-200">
        For Auction
      </Badge>
    )
  } else {
    return (
      <Badge className="bg-purple-100 text-purple-800 whitespace-nowrap hover:bg-purple-200">
        Not Listed
      </Badge>
    )
  }
}
```

**Implementation Notes:**:

- **Purpose**: Displays the NFT's listing status visually.
- **Why Badges?**: Clear, color-coded indicators improve UX.

### TODO 35: Connect `handleCancelListing` to Cancel Listing Button

```javascript
<Button
  variant="destructive"
  size="sm"
  className="flex-1 hover:bg-red-700 transition-colors"
  onClick={() => handleCancelListing(nft)}
>
  Cancel Listing
</Button>
```

**Implementation Notes:**:

- **Purpose**: Triggers listing cancellation.
- **Why Destructive Variant?**: Signals a significant action to users.

### TODO 36: Connect `openListingDialog` to List for Sale Button

```javascript
<Button
  variant="outline"
  size="sm"
  className="flex-1 hover:bg-blue-50 transition-colors"
  onClick={() => openListingDialog(nft)}
>
  List for Sale
</Button>
```

**Implementation Notes:**:

- **Purpose**: Opens the listing dialog for the selected NFT.
- **Why Outline Variant?**: Subtle design for non-destructive actions.

### TODO 37: Connect `handleListNFT` to Listing Dialog Button


```javascript
<Button
  onClick={handleListNFT}
  disabled={(saleType === SaleType.INSTANT && !listingPrice) ||
    (saleType === SaleType.AUCTION && (!auctionMinBid || !auctionDeadline)) ||
    transactionInProcess}
>
  {transactionInProcess ? (
    <>
      <Loader2 className="mr-2 h-4 w-4 animate-spin" />
      {saleType === SaleType.INSTANT ? 'Listing NFT...' : 'Creating Auction...'}
    </>
  ) : (
    saleType === SaleType.INSTANT ? 'List for Sale' : 'Create Auction'
  )}
</Button>
```

**Implementation Notes:**:

- **Purpose**: Submits the listing transaction with input validation.
- **Why Dynamic Text?**: Clarifies the action (sale or auction) and shows progress.

---

## UI Flow Summary

- **CreateNFT Page**:
  - Uploads images to IPFS, previews them, and mints NFTs with collection selection.
  - Uses Scaffold hooks for blockchain interactions and Tailwind for styling.
- **Marketplace Page**:
  - Displays filtered and sorted NFTs with responsive design.
  - Supports dynamic filtering and sorting for user-friendly browsing.
- **NFTDetail Page**:
  - Shows NFT details with interaction options (buy, bid, transfer, resolve auction).
  - Uses dialogs and toasts for user feedback.
- **UserPortfolio Page**:
  - Displays owned NFTs and collections with listing and cancellation options.
  - Provides clear status indicators and transaction feedback.

---

## Key Takeaways

- **Blockchain Integration**: Scaffold hooks simplify Aptos blockchain interactions, reducing complexity.
- **IPFS Storage**: Pinata and IPFS enable cost-effective, decentralized file storage.
- **User Experience**: Conditional rendering, loading states, and toasts ensure a professional interface.
- **Security**: Environment variables and wallet authentication protect sensitive data and transactions.

---

## Next Steps

Explore the [Testing Your NFT Marketplace DApp](./testing-nft-marketplace.md) tutorial to ensure your implementation is robust. Compare your code with the repository's `page.tsx` files for accuracy.
