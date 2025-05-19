# Blockchain Document Signing dApp Frontend Integration Tutorial

In this tutorial, we will implement a document signing application that leverages blockchain technology to create, manage, and sign documents with multiple participants. This frontend application interacts with a Move-based smart contract on the Aptos blockchain.

## Overview

The application has four main functionalities:

1. **Document Creation**: Upload files to IPFS and store document metadata on the blockchain
2. **Document Management**: View and manage your created documents
3. **Signer Management**: Add/remove authorized signers for a document
4. **Document Signing**: Authorized users can sign documents

## Prerequisites

Before starting, ensure you have:

- A basic understanding of Next.js and React
- Familiarity with Aptos blockchain concepts
- Wallet integration set up with @aptos-labs/wallet-adapter-react
- Access to the SecureDocs Move smart contract

## Implementation Steps

Let's break down the implementation by addressing each TODO in the codebase:

### 1. Document Creation & Management Page

First, we'll implement the document creation functionality:

#### TODO 1: Define Pinata API keys

For uploading files to IPFS, we need to set up the Pinata API credentials:

```javascript
const PINATA_API_KEY = process.env.NEXT_PUBLIC_PINATA_API_KEY
const PINATA_API_SECRET = process.env.NEXT_PUBLIC_PINATA_API_SECRET
const PINATA_GATEWAY = process.env.NEXT_PUBLIC_PINATA_GATEWAY || "https://gateway.pinata.cloud/ipfs/"
```

#### TODO 2: Create a format date function

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

#### TODO 3: Get user documents using useView hook

```javascript
// Fetch user's documents for the View Documents tab
  const { data, error: docsError, isLoading: isLoadingDocs } = useView({
    moduleName: "secure_docs",
    functionName: "get_documents_by_signer",
    args: [account?.address as `0x${string}`],
  })
```

#### TODO 4: Implement file upload to Pinata

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

#### TODO 5-7: Validate form input, check wallet connection and submit transaction to create document

```javascript
const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()

    // TODOs 5: Validate the form input and toast error if for any invalid form input
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


    // TODOs 6: check if the user wallet is connected and if not, show a toast error
    if (!connected) {
      console.log("Wallet not connected")
      toast({
        title: "Error",
        description: "Please connect your wallet",
        variant: "destructive",
      })
      return
    }

    // TODOs 7: interact with the smart contract to upload document, use submitTransaction function and toast success or error
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

#### TODO 8: Reset form after successful submission

```javascript
const resetForm = () => {
    setTitle("")
    setFile(null)
    setFileUrl("")
    setIsSubmitted(false)
    setDocId("")
  }

```

#### TODOs 9-12: Implement conditional rendering for document list

For these TODOs, we need to update the conditional rendering in the View Documents tab:

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

### 2. Document Management Page

Now let's implement the document management page functionality:

#### TODO 13: Get document data using useView

```javascript
const { data, error, isLoading, refetch } = useView({
    moduleName: "secure_docs",
    functionName: "get_document",
    args: [documentId],
  })
```

#### TODO 14: Implement add signer functionality

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

#### TODO 15: Implement remove signer functionality

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

### 3. Document Signing Page

Next, let's implement the document signing functionality:

#### TODO 16: Get document data using useView

```javascript
 const { data, error, isLoading, refetch } = useView({
    moduleName: "secure_docs",
    functionName: "get_document",
    args: [documentId],
  })
```

#### TODO 17: Parse the document data

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

#### TODOs 18-19: Check if user is allowed to sign and has already signed

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

#### TODOs 20-21: Implement sign document functionality

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

### 4. Sign Documents List Page

Finally, let's implement the sign documents list page:

#### TODO 22: Get documents for the current user

```javascript
// Fetch documents available for the current signer
  const { data, error, isLoading } = useView({
    moduleName: "secure_docs",
    functionName: "get_documents_for_signer",
    args: [account?.address as `0x${string}`],
  })
```

#### TODO 23: Check if the user has already signed a document

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

#### TODOs 24-27: Implement conditional rendering for the documents list

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

## Testing Your Implementation

After implementing all the TODO items, you should test each functionality:

1. **Document Creation**:
   - Connect your wallet
   - Upload a file
   - Create a document
   - Verify it appears in your documents list

2. **Document Management**:
   - View your created documents
   - Add and remove signers
   - Verify changes are reflected

3. **Document Signing**:
   - Use a different wallet address that's authorized to sign
   - Navigate to the document signing page
   - Sign the document
   - Verify your signature appears

## Conclusion

You've now implemented a fully functional blockchain document signing application! This dApp demonstrates how to:

- Upload files to IPFS and store references on the blockchain
- Manage document access through authorized signers
- Securely sign documents with blockchain verification
- Create a user-friendly interface for blockchain interactions

The combination of Next.js frontend with Aptos blockchain backend provides a powerful, secure platform for document signing that can be extended with additional features like notifications, deadlines, or document versioning.

Remember that in a production environment, you should:

- Secure your API keys (using environment variables)
- Add more robust error handling
- Implement proper authentication and authorization
- Consider usability improvements like progress indicators and better validation feedback

Happy building!
