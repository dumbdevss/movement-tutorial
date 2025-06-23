# Document Management Contract Tutorial - Frontend Integration

This tutorial will guide you through integrating your frontend application with our Move-based NFT ticketing smart contract. You'll learn how to implement ticket purchasing functionality using Scaffold hooks to create a seamless connection between your frontend and the blockchain.

## Understanding the Architecture

Before diving into the implementation, let's understand how our document management system works:

**Our Tech Stack:**

- **Move Smart Contract**: Handles document storage, access control, and signature verification
- **IPFS (via Pinata)**: Stores the actual document files in a decentralized manner
- **React Frontend**: Provides the user interface for document management
- **Scaffold Hooks**: Abstracts blockchain interactions into simple React hooks

## Source Code Reference

The complete implementation of the frontend interface can be found in the repository:

**Repository:** [dumbdevss/document-management](https://github.com/dumbdevss/document-management)  
**Branch:** `frontend-integration`

You can view or clone the finished code to compare against your implementation as you follow along with this guide

## Document Creation Page (/app/create-document)

Navigate to the document creation page component to implement TODOs 1-12.

### Understanding IPFS Integration

**Why IPFS for File Storage?**
Storing large files directly on the blockchain would be extremely expensive. Instead, we use IPFS (InterPlanetary File System) to store files in a decentralized manner and only store the IPFS hash on the blockchain. This approach gives us:

- **Cost Efficiency**: Only small hashes are stored on-chain
- **Decentralization**: Files are distributed across IPFS nodes
- **Content Addressing**: Files are identified by their content hash, ensuring integrity

**Why Pinata?**
Pinata is a pinning service that ensures your IPFS files remain available by keeping them "pinned" on their infrastructure. Without a pinning service, IPFS files might become unavailable if no nodes are hosting them.

### TODO 1: Define Pinata API keys

First, we need to set up the Pinata API credentials for IPFS file uploads:

```javascript
const PINATA_API_KEY = process.env.NEXT_PUBLIC_PINATA_API_KEY
const PINATA_API_SECRET = process.env.NEXT_PUBLIC_PINATA_API_SECRET
const PINATA_GATEWAY = process.env.NEXT_PUBLIC_PINATA_GATEWAY || "https://gateway.pinata.cloud/ipfs/"
```

**Implementation Notes:**

- Uses environment variables for secure API key management
- The gateway URL is how we'll construct URLs to access files from IPFS
- Environment variables starting with `NEXT_PUBLIC_` are accessible in the browser

### TODO 2: Create a format date function

Next, implement a utility function to format blockchain timestamps:

```javascript
const formatDate = (timestamp: number) => {
  return new Date(timestamp * 1000).toLocaleDateString("en-US", {
    year: "numeric",
    month: "short",
    day: "numeric",
    hour: "numeric",
    minute: "2-digit",
    timeZone: "Europe/London",
  });
};
```

**Why This Conversion is Needed:**

- Blockchain timestamps are typically stored as Unix timestamps (seconds since Jan 1, 1970)
- JavaScript Date objects expect milliseconds, so we multiply by 1000
- Consistent date formatting improves user experience across different locales

### Understanding Blockchain Data Queries

**What are View Functions?**
View functions are read-only operations that don't modify blockchain state. They're perfect for querying data because:

- They don't cost gas fees
- They execute instantly
- They can be called repeatedly without side effects

**The useView Hook:**
This hook abstracts the complexity of making blockchain calls, handling loading states, and managing errors automatically.

### TODO 3: Get user documents using useView hook

Implement the data fetch for displaying the user's documents:

```javascript
// Fetch user's documents for the View Documents tab
const { data, error: docsError, isLoading: isLoadingDocs } = useView({
  moduleName: "secure_docs",
  functionName: "get_documents_by_signer",
  args: [account?.address as `0x${string}`],
})
```

**Implementation Notes:**

- `moduleName`: References our Move module containing the smart contract functions
- `functionName`: The specific view function we want to call
- `args`: Parameters passed to the function (in this case, the user's wallet address)
- The hook returns data, error state, and loading state for comprehensive UI handling

### TODO 4: Implement file upload to Pinata

Now implement the function to upload files to IPFS:

```javascript
const uploadToPinata = async (file: File) => {
  setIsUploading(true)

  try {
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
      throw new Error("Failed to upload to Pinata")
    }

    const data = await response.json()
    const ipfsHash = data.IpfsHash
    const fileUrl = `${PINATA_GATEWAY}/ipfs/${ipfsHash}`

    setFileUrl(fileUrl)
    toast({
      title: "File Uploaded",
      description: "Your file has been uploaded successfully.",
    })
  } catch (error) {
    console.error("Error uploading to Pinata:", error)
    throw error
  } finally {
    setIsUploading(false)
  }
}
```

**Understanding the Upload Process:**

1. **FormData**: Browser API for constructing multipart/form-data (required for file uploads)
2. **IPFS Hash**: A unique identifier generated from the file's content - if the file changes, the hash changes
3. **Gateway URL**: Provides HTTP access to IPFS content through traditional web requests
4. **Error Handling**: Essential for network operations that can fail

### Understanding Blockchain Transactions

**Transactions vs. View Calls:**

- **View calls**: Read data, free, instant
- **Transactions**: Modify state, cost gas, require wallet signatures, take time to confirm

**Why Wallet Connection Matters:**
Every blockchain transaction must be signed by a private key to prove authorization. Wallet connection gives our app permission to request these signatures.

### TODOs 5-7: Form validation, wallet connection check, and transaction submission

Next, implement the form submission handler with validation:

```javascript
const handleSubmit = async (e: React.FormEvent) => {
  e.preventDefault()

  // TODO 5: Validate the form input
  if (!title.trim()) {
    toast({
      title: "Error",
      description: "Please enter a document title",
      variant: "destructive",
    })
    return
  }

  if (!fileUrl) {
    toast({
      title: "Error",
      description: "Please upload a document file",
      variant: "destructive",
    })
    return
  }

  // TODO 6: Check if wallet is connected
  if (!connected) {
    console.log("Wallet not connected")
    toast({
      title: "Error",
      description: "Please connect your wallet",
      variant: "destructive",
    })
    return
  }

  // TODO 7: Submit transaction to blockchain
  try {
    const id = nanoid();
    setDocId(id);
    
    // Submit transaction to blockchain
    await submitTransaction("upload_document", [
      title,
      fileUrl,
      id
    ])
    
    // Set submission successful state
    setIsSubmitted(true)

    toast({
      title: "Success",
      description: "Document created successfully. You can now manage signers from the document page.",
    })
  } catch (error) {
    console.error("Transaction error:", error)
    toast({
      title: "Transaction Failed",
      description: "There was an error creating the document.",
      variant: "destructive",
    })
  }
}
```

**Why Generate a Unique ID?**

- `nanoid()` creates a URL-safe, unique identifier
- This ID becomes the primary key for our document in the smart contract
- Having predictable IDs could create security vulnerabilities or naming conflicts

**Transaction Submission Process:**

1. **Validation**: Ensure all required data is present before expensive blockchain operations
2. **Wallet Check**: Verify user can sign transactions
3. **Transaction**: Send the request to the blockchain
4. **Confirmation**: Wait for blockchain confirmation before updating UI

### TODO 8: Reset form after successful submission

Implement the form reset function:

```javascript
const resetForm = () => {
  setTitle("")
  setFile(null)
  setFileUrl("")
  setIsSubmitted(false)
  setDocId("")
}
```

**Why Reset Forms?**

- Prevents accidental duplicate submissions
- Provides clear visual feedback that the operation completed
- Prepares the form for the next document creation

### Understanding UI State Management

Modern web applications need to handle multiple states gracefully:

- **Loading states**: Show users that something is happening
- **Error states**: Inform users when something goes wrong
- **Empty states**: Guide users when there's no data to display
- **Success states**: Confirm that operations completed successfully

### TODOs 9-12: Implement conditional rendering for document list

Update the UI to handle different document list states:

```javascript
{!connected && (
  <div className="flex flex-col items-center justify-center py-10 text-center">
    <FileText className="h-12 w-12 text-muted-foreground/50" />
    <h3 className="mt-4 text-lg font-medium">Connect Your Wallet</h3>
    <p className="mt-2 text-sm text-muted-foreground">
      Please connect your wallet to view your documents.
    </p>
  </div>
)}

{connected && isLoadingDocs && (
  <div className="flex flex-col items-center justify-center py-10 text-center">
    <Loader2 className="h-8 w-8 animate-spin text-muted-foreground/70" />
    <p className="mt-4 text-sm text-muted-foreground">Loading your documents...</p>
  </div>
)}

{connected && !isLoadingDocs && docsError && (
  <div className="flex flex-col items-center justify-center py-10 text-center">
    <p className="text-sm text-red-500">Failed to load documents. Please try again.</p>
  </div>
)}

{connected && !isLoadingDocs && !docsError && userDocuments && userDocuments?.length === 0 && (
  <div className="flex flex-col items-center justify-center py-10 text-center">
    <FileText className="h-12 w-12 text-muted-foreground/50" />
    <h3 className="mt-4 text-lg font-medium">No Documents Found</h3>
    <p className="mt-2 text-sm text-muted-foreground">
      You haven't created any documents yet.
    </p>
    <Button 
      className="mt-4" 
      onClick={() => setActiveTab("create")}
    >
      Create Your First Document
    </Button>
    </div>
  )}
```

**Progressive Disclosure Pattern:**
Each state shows only the information relevant to the user's current situation, reducing cognitive load and providing clear next steps.

## Document Management Page (/app/manage-document/[id])

Navigate to the document management page component to implement TODOs 13-15.

### Understanding Access Control in Blockchain

**Why Signer Management Matters:**
In traditional document systems, access control happens on servers controlled by companies. In blockchain systems, access control is enforced by smart contract code that no single party can change arbitrarily.

**The Concept of "Allowed Signers":**

- Only the document owner can add/remove authorized signers
- This list is stored immutably on the blockchain
- Smart contract functions check this list before allowing signatures

### TODO 13: Get document data using useView

Implement data fetching for a specific document:

```javascript
const { data, error, isLoading, refetch } = useView({
  moduleName: "secure_docs",
  functionName: "get_document",
  args: [documentId],
})
```

**Why We Need Refetch:**
After modifying signer lists, we need to update the UI to reflect changes. The `refetch` function lets us refresh the data without a full page reload.

### TODO 14: Implement add signer functionality

Implement the function to add an authorized signer:

```javascript
const handleAddSigner = async () => {
  if (!newSignerAddress.trim()) {
    toast({
      title: "Error",
      description: "Please enter a wallet address for the new signer",
      variant: "destructive",
    })
    return
  }

  try {
    await submitTransaction("add_signer", [
      documentId,
      newSignerAddress as `0x${string}`,
    ])

    toast({
      title: "Success",
      description: "Signer added successfully",
    })

    setNewSignerAddress("")
    refetch()
  } catch (error) {
    console.error("Error adding signer:", error)
    toast({
      title: "Error",
      description: "Failed to add signer. Please try again.",
      variant: "destructive",
    })
  }
}
```

**Wallet Address Validation:**

- Blockchain addresses must be exactly formatted
- Invalid addresses will cause transaction failures
- Frontend validation saves users gas fees from failed transactions

### TODO 15: Implement remove signer functionality

Now implement the function to remove an authorized signer:

```javascript
const handleRemoveSigner = async (signerAddress: `0x${string}`) => {
  try {
    await submitTransaction("remove_signer", [
      documentId,
      signerAddress as `0x${string}`,
    ])

    toast({
      title: "Success",
      description: "Signer removed successfully",
    })
    
    refetch()
  } catch (error) {
    console.error("Error removing signer:", error)
    toast({
      title: "Error",
      description: "Failed to remove signer. Please try again.",
      variant: "destructive",
    })
  }
}
```

**Security Consideration:**
The smart contract enforces that only the document owner can remove signers. Even if someone bypassed the frontend, the blockchain would reject unauthorized attempts.

## Document Signing Page (/app/sign-document)

Navigate to the document signing page component to implement TODOs 16-21.

### Understanding Digital Signatures in Blockchain

**What Makes Blockchain Signatures Special?**

- **Non-repudiation**: Signers cannot later deny they signed
- **Immutability**: Signatures cannot be removed or altered
- **Transparency**: Anyone can verify signatures independently
- **Cryptographic Proof**: Each signature is mathematically verifiable

**The Signing Process:**

1. User views the document content
2. User confirms they want to sign
3. Wallet creates a cryptographic signature
4. Smart contract records the signature with timestamp

### TODO 16: Get document data using useView

Implement data fetching for the document to be signed:

```javascript
const { data, error, isLoading, refetch } = useView({
  moduleName: "secure_docs",
  functionName: "get_document",
  args: [documentId],
})
```

**Why Fetch Document Data for Signing:**
Users need to see what they're signing before committing their signature to the blockchain.

### TODO 17: Parse the document data

Parse the returned blockchain data into a usable format:

```javascript
// Parse document data
const document = data ? {
  title: data[0], // doc.name
  ipfs_hash: data[1], // doc.ipfs_hash
  created_at: data[2], // doc.created_at
  owner: data[3], // doc.owner
  signatures: data[4], // doc.signatures
  allowed_signers: data[5], // doc.allowed_signers
} : null;
```

**Understanding Blockchain Data Structure:**
Move functions often return data as arrays or tuples. We destructure this into named properties for easier frontend use. The comments show which struct fields correspond to each array index.

### TODOs 18-19: Check if user is allowed to sign and has already signed

Implement the authorization checks for signing:

```javascript
// Check if current user is allowed to sign
const isAllowedSigner = walletAddress && document ?
  document.allowed_signers.some(
    signer => signer.toLowerCase() === walletAddress.toLowerCase()
  ) : false;

// Check if user has already signed
const hasAlreadySigned = walletAddress && document ?
  document.signatures.some(
    sig => sig.toLowerCase() === walletAddress.toLowerCase()
  ) : false;
```

**Why These Checks Matter:**

- **Authorization**: Prevents unauthorized signatures
- **Duplicate Prevention**: Avoids redundant signatures from the same user
- **Case Insensitivity**: Wallet addresses can be represented in different cases

**The Array.some() Method:**
This JavaScript method returns true if at least one element in the array passes the test function. Perfect for checking membership in lists.

### TODOs 20-21: Implement sign document functionality

Implement the document signing function:

```javascript
const handleSignDocument = async () => {
  // Check if signature is empty or user is not connected
  if (!signature.trim()) {
    toast({
      title: "Missing Signature",
      description: "Please enter your signature to sign the document.",
      variant: "destructive",
    });
    return;
  }

  if (!connected || !walletAddress) {
    toast({
      title: "Wallet Not Connected",
      description: "Please connect your wallet to sign this document.",
      variant: "destructive",
    });
    return;
  }

  try {
    const result = await submitTransaction({
      functionName: "sign_document",
      args: [documentId],
      options: {
        max_gas_amount: 5000,
      },
    });

    if (result) {
      toast({
        title: "Document Signed",
        description: "You have successfully signed this document.",
      });
      setSignature("");
      refetch(); // Refresh the document data
    }
  } catch (error) {
    console.error("Error signing document:", error);
    toast({
      title: "Failed to Sign",
      description: "There was an error signing the document. Please try again.",
      variant: "destructive",
    });
  }
};
```

**The Signature Field:**
While blockchain wallets provide cryptographic signatures automatically, we collect a "display signature" (like a name or initials) for human-readable identification in the UI.

## Sign Documents List Page (/app/sign-documents)

Navigate to the sign documents list page component to implement TODOs 22-27.

### Understanding Document Discovery

**The Challenge:**
How do users find documents they need to sign without browsing through every document on the blockchain?

**Our Solution:**
We maintain a mapping in the smart contract that allows efficient lookup of documents by authorized signer address. This is why we use a `Table` data structure in Move - it provides O(1) lookup performance.

### TODO 22: Get documents for the current user

Implement data fetching for documents requiring the user's signature:

```javascript
// Fetch documents available for the current signer
const { data, error, isLoading } = useView({
  moduleName: "secure_docs",
  functionName: "get_documents_for_signer",
  args: [account?.address as `0x${string}`],
})
```

**Efficient Data Retrieval:**
This function uses the signer-to-documents mapping in our smart contract, avoiding the need to scan through all documents to find relevant ones.

### TODO 23: Check if the user has already signed a document

Implement a utility function to check document signing status:

```javascript
const hasUserSigned = (document: any) => {
  if (!account?.address || !document || !document.signatures) return false;

  // Check if the current user's address is in the document's signatures
  const userHasSigned = document.signatures.some(
    (sig: string) => sig.toLowerCase() === account.address?.toLowerCase()
  );

  return userHasSigned;
};
```

**Why This Function is Useful:**

- Shows different UI states for signed vs. unsigned documents
- Prevents confusion about signing status
- Enables filtering and sorting of document lists

### TODOs 24-27: Implement conditional rendering for the documents list

Update the UI to handle different document list states:

```javascript
{!connected && (
  <Card className="mb-8">
    <CardContent className="flex flex-col items-center justify-center py-12">
      <FileText className="h-12 w-12 text-muted-foreground/50" />
      <h3 className="mt-4 text-lg font-medium">Connect Your Wallet</h3>
      <p className="mt-2 text-center text-sm text-muted-foreground">
        Please connect your wallet to view documents that require your signature.
      </p>
    </CardContent>
  </Card>
)}

{connected && isLoading && (
  <Card className="mb-8">
    <CardContent className="flex flex-col items-center justify-center py-12">
      <Loader2 className="h-8 w-8 animate-spin text-muted-foreground/70" />
      <p className="mt-4 text-sm text-muted-foreground">Loading your documents...</p>
    </CardContent>
  </Card>
)}

{connected && !isLoading && error && (
  <Card className="mb-8 border-red-200 bg-red-50">
    <CardContent className="flex flex-col items-center justify-center py-12">
      <p className="text-center text-red-600">
        Error loading documents. Please try again later.
      </p>
    </CardContent>
  </Card>
)}

{connected && !isLoading && !error && signerDocuments.length === 0 && (
  <Card className="mb-8">
    <CardContent className="flex flex-col items-center justify-center py-12">
      <FileText className="h-12 w-12 text-muted-foreground/50" />
      <h3 className="mt-4 text-lg font-medium">No Documents</h3>
      <p className="mt-2 text-center text-sm text-muted-foreground">
        There are no documents requiring your signature at this time.
      </p>
    </CardContent>
  </Card>
)}
```

**Card-Based Layout:**
Using consistent Card components creates a professional appearance and ensures proper spacing and visual hierarchy across all states.

## Conclusion

Congratulations! You've successfully built a complete blockchain document management application by understanding and implementing:

1. **Smart Contract Integration**: How to call Move functions from React using Scaffold hooks
2. **IPFS File Storage**: Decentralized file storage with Pinata pinning services
3. **Access Control**: Blockchain-based permission systems that can't be bypassed
4. **State Management**: Handling multiple UI states for professional user experience
5. **Transaction Handling**: Managing blockchain transactions with proper error handling
6. **Data Structures**: Choosing the right blockchain data structures for performance

**Key Takeaways:**

- Blockchain applications require different thinking about data persistence and access control
- User experience in dApps requires careful handling of loading states and transaction feedback
- Combining on-chain and off-chain storage (blockchain + IPFS) provides the best of both worlds
- Smart contract design decisions directly impact frontend complexity and user experience

**Next Steps:**
Now that you understand the fundamentals, you can extend this system with features like:

- Document versioning
- Signature deadlines
- Multi-signature requirements
- Document templates
- Notification systems

Next Tutorial: [Testing the Document Management DApp](./testing-document-management.md)
