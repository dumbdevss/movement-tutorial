# NFT Ticketing Contract Tutorial - Frontend Integration

This tutorial will guide you through integrating your frontend application with our Move-based NFT ticketing smart contract. You'll learn how to implement ticket purchasing functionality using Scaffold hooks to create a seamless connection between your frontend and the blockchain.

## Source Code Reference

The complete implementation of the frontend interface can be found in the repository:

**Repository:** [dumbdevss/document-management](https://github.com/dumbdevss/document-management)  
**Branch:** `frontend-integration`

You can view or clone the finished code to compare against your implementation as you follow along with this guide. The frontend code includes:

- React components for document management 
- Integration with Move smart contracts using Scaffold hooks
- IPFS integration for file storage
- Complete styling using Tailwind CSS

## Document Creation Page (/app/create-document)

Navigate to the document creation page component to implement TODOs 1-12.

### TODO 1: Define Pinata API keys

First, we need to set up the Pinata API credentials for IPFS file uploads:

```javascript
const PINATA_API_KEY = process.env.NEXT_PUBLIC_PINATA_API_KEY
const PINATA_API_SECRET = process.env.NEXT_PUBLIC_PINATA_API_SECRET
const PINATA_GATEWAY = process.env.NEXT_PUBLIC_PINATA_GATEWAY || "https://gateway.pinata.cloud/ipfs/"
```

**Implementation Notes:**

- Uses environment variables for secure API key management
- Provides a default gateway URL as fallback

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

**Implementation Notes:**

- Converts blockchain timestamps (in seconds) to JavaScript Date objects (in milliseconds)
- Formats the date with consistent localization parameters

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

- Uses the `useView` hook to query blockchain data
- Passes the user's wallet address as an argument
- Captures loading state and errors for UI management

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

**Implementation Notes:**

- Prepares a FormData object with the file
- Sends a POST request to Pinata API with appropriate headers
- Constructs an IPFS URL from the returned hash
- Provides user feedback and manages loading state

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

**Implementation Notes:**

- Validates required form fields before proceeding
- Verifies wallet connection status
- Generates a unique document ID with nanoid()
- Submits transaction to the blockchain with required parameters
- Provides appropriate user feedback for success/failure

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

**Implementation Notes:**

- Clears all form fields and state variables
- Resets submission status for next document creation

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

**Implementation Notes:**

- Shows different UI states based on connection, loading, and data status
- Provides clear user guidance for each state
- Offers a quick action to create documents when none exist

## Document Management Page (/app/manage-document/[id])

Navigate to the document management page component to implement TODOs 13-15.

### TODO 13: Get document data using useView

Implement data fetching for a specific document:

```javascript
const { data, error, isLoading, refetch } = useView({
  moduleName: "secure_docs",
  functionName: "get_document",
  args: [documentId],
})
```

**Implementation Notes:**

- Uses `useView` hook to fetch specific document data
- Includes a refetch function for data refresh after modifications
- Captures loading and error states for UI management

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

**Implementation Notes:**

- Validates the new signer address before proceeding
- Submits a blockchain transaction to add the signer
- Clears the input field and refreshes document data on success
- Provides appropriate error handling and user feedback

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

**Implementation Notes:**

- Takes the signer address as a parameter
- Submits transaction to remove the signer from the blockchain record
- Refreshes document data to reflect the change
- Handles errors with appropriate user feedback

## Document Signing Page (/app/sign-document)

Navigate to the document signing page component to implement TODOs 16-21.

### TODO 16: Get document data using useView

Implement data fetching for the document to be signed:

```javascript
const { data, error, isLoading, refetch } = useView({
  moduleName: "secure_docs",
  functionName: "get_document",
  args: [documentId],
})
```

**Implementation Notes:**

- Uses `useView` hook to fetch document data
- Includes refetch function to update UI after signing
- Captures loading and error states for UI handling

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

**Implementation Notes:**

- Maps the array data returned from the blockchain to named properties
- Provides clear comments to identify each data element
- Handles null case when data isn't available yet

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

**Implementation Notes:**

- Uses JavaScript Array.some() to check if the current wallet address is in the allowed signers list
- Performs case-insensitive comparison of wallet addresses
- Checks if the user has already signed the document
- Handles null cases for wallet address or document data

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

**Implementation Notes:**

- Validates signature input and wallet connection
- Submits transaction with specific gas limit options
- Clears signature input and refreshes data after successful signing
- Provides appropriate error handling and user feedback

## Sign Documents List Page (/app/sign-documents)

Navigate to the sign documents list page component to implement TODOs 22-27.

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

**Implementation Notes:**

- Uses `useView` hook to fetch documents that need the user's signature
- Passes the current wallet address as an argument
- Captures loading and error states for UI handling

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

**Implementation Notes:**

- Takes a document object as parameter
- Handles null/undefined cases safely
- Uses Array.some() to check if user's address is in signatures list
- Performs case-insensitive comparison

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

**Implementation Notes:**

- Shows appropriate UI for each state: not connected, loading, error, and empty list
- If wallet is not connected, show the connect wallet *Card* component
- If wallet is connected and there is error doing viewing of documents, show an error *Card* component
- If wallet is connected and there is no error while viewing documents but the document array is empty, then show a No document *Card* componenet

<!-- ## Testing Your Implementation

After implementing all TODOs, test each functionality step by step:

1. **Document Creation Flow**:
   - Navigate to `/app/create-document`
   - Connect your wallet
   - Fill in document details and upload a file
   - Create the document and verify success

2. **Document Management Flow**:
   - Navigate to `/app/manage-document/[id]`
   - Add and remove signers
   - Verify changes are reflected in the UI

3. **Document Signing Flow**:
   - Navigate to `/app/sign-documents` to see documents requiring signatures
   - View and sign specific documents at `/app/sign-document/[id]`
   - Verify signature appears after signing -->

## Conclusion

Congratulations! You've successfully built a complete blockchain document management application by:

1. Implementing the Move smart contract with:
   - Document creation and storage
   - Signer management functionality
   - Secure signature verification

2. Creating the frontend interface with:
   - IPFS integration for file uploads
   - Document access control through authorized signers
   - Blockchain transaction handling for document signing
   - Responsive and user-friendly UI components

Next Tutorial: [Testing the Document Management DApp](./testing-document-management.md)
