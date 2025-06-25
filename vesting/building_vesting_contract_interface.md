# Building a Token Vesting Frontend Interface - Complete Tutorial

## Overview

This tutorial will guide you through building a complete frontend interface for a token vesting smart contract using React, TypeScript, and the Aptos blockchain. We'll implement all the TODOs step by step to create a fully functional admin dashboard.

## Prerequisites

- Basic knowledge of React and TypeScript
- Understanding of blockchain concepts
- Familiarity with Aptos blockchain
- Node.js and npm/yarn installed

## Project Structure

Our admin dashboard includes:
- **Bulk Upload**: Upload XLS files with multiple vesting streams
- **Individual Stream Creation**: Create single vesting streams
- **Stream Management**: View and manage existing streams
- **Token Deposit**: Deposit tokens into the vesting contract

## Step 1: Setting Up the Data Fetching Hook

### TODO 1: Implement useView hook for fetching all streams

Replace the placeholder data fetching with the actual `useView` hook:

```tsx
// Replace this placeholder:
const { data, error, isLoading } = {
  data: [[]],
  error: null,
  isLoading: false
}

// With this implementation:
const { data, error, isLoading } = useView({
  moduleName: "vesting",
  functionName: "get_all_streams",
})
```

**What this does:**
- Connects to the smart contract's `get_all_streams` function
- Automatically handles loading states and errors
- Returns stream data in a reactive way

## Step 2: Implementing Authorization

### TODO 2: Implement authorization check

Replace the hardcoded authorization with proper address comparison:

```tsx
// Replace this:
const isAuthorized = false // Replace with actual check against module address

// With this:
const isAuthorized = account?.address === process.env.NEXT_PUBLIC_MODULE_ADDRESS
```

**What this does:**
- Compares the connected wallet address with the module owner address
- Only allows authorized users to access admin functions
- Uses environment variables for security

## Step 3: File Upload Functionality

### TODO 3: Implement handleFileUpload function

```tsx
const handleFileUpload = async (e: React.ChangeEvent<HTMLInputElement>) => {
    if (!e.target.files || e.target.files.length === 0) return

    setIsUploading(true)
    setUploadError(null)

    const file = e.target.files[0]

    try {
      const reader = new FileReader()
      reader.onload = async (event) => {
        if (!event.target?.result) {
          throw new Error("Failed to read file")
        }

        const data = new Uint8Array(event.target.result as ArrayBuffer)
        const workbook = XLSX.read(data, { type: "array" })
        const sheetName = workbook.SheetNames[0]
        const sheet = workbook.Sheets[sheetName]
        const jsonData = XLSX.utils.sheet_to_json(sheet)

        const processedData = jsonData.map((row: any) => ({
          wallet_address: row.wallet_address || row.address || "",
          amount: Number(row.Amount) || 0,
          duration: String(row.duration || ""),
          cliff: String(row.cliff || ""),
        }))

        const validData = processedData.filter((row) => {
          const isValidAddress = /^0x[a-fA-F0-9]{64}$/.test(row.wallet_address)
          const isValidAmount = row.amount > 0
          let isValidDuration = false
          let isValidCliff = false
          try {
            parseTimeToSeconds(row.duration)
            isValidDuration = true
          } catch {
            // Invalid duration format
          }
          try {
            parseTimeToSeconds(row.cliff)
            isValidCliff = true
          } catch {
            // Invalid cliff format
          }
          return isValidAddress && isValidAmount && isValidDuration && isValidCliff
        })

        if (validData.length === 0) {
          throw new Error(
            "No valid data found in the file. Ensure columns include wallet_address, amount, duration, cliff (e.g., 1d, 1min, 1mon, 1yr)."
          )
        }

        // Extract and set state variables from validData
        const newRecipients = validData.map(row => row.wallet_address as `0x${string}`)
        const newAmounts = validData.map(row => row.amount)
        const newDurations = validData.map(row => row.duration)
        const newCliffs = validData.map(row => row.cliff)

        // Update state with extracted data
        setRecipients(newRecipients)
        setAmounts(newAmounts)
        setDurations(newDurations)
        setCliffs(newCliffs)

        setIsUploading(false)
        setUploadSuccess(true)
      }

      reader.onerror = () => {
        setIsUploading(false)
        setUploadError("Error reading file")
      }

      reader.readAsArrayBuffer(file)
    } catch (error: any) {
      setIsUploading(false)
      setUploadError(error.message || "Failed to process file")
    }
  }
```

**Key Features:**
- Validates file format and data structure
- Supports both XLS and CSV files
- Provides detailed error messages
- Updates UI state appropriately

## Step 4: Input Change Handler

### TODO 4: Implement handleInputChange function

```tsx
const handleInputChange = (e: React.ChangeEvent<HTMLInputElement>, index: number) => {
    const { name, value } = e.target
    console.log("Input changed:", name, value)

    switch (name) {
      case "wallet-address":
        const updatedRecipients = [...recipients]
        updatedRecipients[index] = value as `0x${string}`
        setRecipients(updatedRecipients)
        break
      case "amount":
        const updatedAmounts = [...amounts]
        updatedAmounts[index] = Number(value)
        setAmounts(updatedAmounts)
        break
      case "duration":
        const updatedDurations = [...durations]
        updatedDurations[index] = value
        setDurations(updatedDurations)
        break
      case "cliff":
        const updatedCliffs = [...cliffs]
        updatedCliffs[index] = value
        setCliffs(updatedCliffs)
        break
      case "deposit-amount":
        setDepositAmount(Number(value))
        break
      default:
        break
    }
  }
```

**What this does:**
- Handles all form input changes
- Updates the appropriate state arrays
- Maintains immutability with spread operator

## Step 5: Single Stream Creation

### TODO 5: Implement handleCreateStream function

```tsx
const handleCreateStream = async (
    recipient: `0x${string}`,
    amount: number,
    duration: number,
    cliff: number
  ) => {
    try {
      const streamId = nanoid(15)
      await submitTransaction("create_stream", [recipient, amount * decimal, duration, cliff, streamId])
      let response = transactionResponse as any
      if (response ) {
        toast({
          title: "Stream Created",
          description: `Stream created successfully.`,
          action: (
            <a
              href={`https://explorer.movementnetwork.xyz/txn/${response?.transactionHash}`}
              target="_blank"
              rel="noopener noreferrer"
            >
              <ToastAction altText="View transaction">View txn</ToastAction>
            </a>
          ),
        });
        setRecipients([]);
        setAmounts([]);
        setDurations([]);
        setCliffs([]);
      } else {
        toast({
          title: "Stream Creation Failed",
          description: `Transaction hash not available or transaction failed.`,
          variant: "destructive",
        })
      }
    } catch (error) {
      toast({
        title: "Error",
        description: `Failed to create stream for ${recipient}: ${error}`,
        variant: "destructive",
      })
      throw error
    }
  }
```

## Step 6: Multiple Streams Creation

### TODO 6: Implement handleCreateMultipleStreams function

```tsx
const handleCreateMultipleStreams = async (
    recipients: `0x${string}`[],
    amounts: number[],
    durations: number[],
    cliffs: number[]
  ) => {
    try {
      if (
        recipients.length !== amounts.length ||
        recipients.length !== durations.length ||
        recipients.length !== cliffs.length
      ) {
        throw new Error("All input arrays must have the same length")
      }

      const streamIds = Array.from({ length: recipients.length }, () => nanoid(15))
      const formatedAmounts = amounts.map((amount) => amount * decimal)
      await submitTransaction("create_multiple_streams", [recipients, formatedAmounts, durations, cliffs, streamIds])

      let response = transactionResponse as any
      if (response ) {
        toast({
          title: "Streams Created",
          description: `Streams created successfully for users.`,
          action: (
            <a
              href={`https://explorer.movementnetwork.xyz/txn/${response?.transactionHash}`}
              target="_blank"
              rel="noopener noreferrer"
            >
              <ToastAction altText="View transaction">View txn</ToastAction>
            </a>
          ),
        })
      } else {
        toast({
          title: "Stream Creation Failed",
          description: `Transaction hash not available or transaction failed.`,
          variant: "destructive",
        });
      }
    } catch (error) {
      toast({
        title: "Error",
        description: `Failed to create streams: ${error}`,
        variant: "destructive",
      });
      throw error
    }
  }
```

## Step 7: Token Deposit Functionality

### TODO 7: Implement handleDeposit function

```tsx
 const handleDeposit = async (amount: number) => {
    try {
      await submitTransaction("deposit", [amount * decimal])
      let response = transactionResponse as any
      if (response ) {
        toast({
          title: "Deposit Successful",
          description: `Deposit of ${amount} tokens successful.`,
          action: (
            <a
              href={`https://explorer.movementnetwork.xyz/txn/${response?.transactionHash}`}
              target="_blank"
              rel="noopener noreferrer"
            >
              <ToastAction altText="View transaction">View txn</ToastAction>
            </a>
          ),
        });
      } else {
        toast({
          title: "Deposit Failed",
          description: `Transaction hash not available or transaction failed.`,
          variant: "destructive",
        })
        console.error("Transaction hash not available or transaction failed.")
        return null
      }
    } catch (error) {
      console.error("Error submitting deposit transaction:", error)
      throw error
    }
  }
```

## Step 8-16: Connecting Event Handlers

Now we need to connect all our functions to the UI components:

### TODO 8: Connect file upload
```tsx
// Replace:
onChange={() => {}}
// With:
onChange={handleFileUpload}
```

### TODO 9: Connect bulk stream creation
```tsx
// Replace:
onClick={() => {}}
// With:
onClick={() => {
  if (recipients[0] && amounts[0] && durations[0] && cliffs[0]) {
    handleCreateMultipleStreams(
      recipients,
      amounts,
      durations.map((duration) => parseTimeToSeconds(duration)),
      cliffs.map((cliff) => parseTimeToSeconds(cliff))
    )
  }
}}
```

### TODO 10-13: Connect individual form inputs
```tsx
// Replace all onChange={() => {}} with:
onChange={(e) => handleInputChange(e, 0)}
```

### TODO 14: Connect individual stream creation
```tsx
// Replace:
onClick={() => {}}
// With:
onClick={() => {
  if (recipients[0] && amounts[0] && durations[0] && cliffs[0]) {
    handleCreateStream(
      recipients[0] as `0x${string}`,
      amounts[0],
      parseTimeToSeconds(durations[0]),
      parseTimeToSeconds(cliffs[0])
    )
  }
}}
```

### TODO 15: Connect deposit input

```tsx
// Replace:
onChange={() => {}}
// With:
onChange={(e) => handleInputChange(e, 0)}
```

### TODO 16: Connect deposit button

```tsx
// Replace:
onClick={() => {}}
// With:
onClick={() => {
  if (depositAmount > 0) {
    handleDeposit(depositAmount)
  }
}}
```

TODO 17: Implement useView hook for fetching user streams

```tsx
// Replace this placeholder:
const { data, error, isLoading } = {
  data: [[]],
  error: null,
  isLoading: false
}

// With this implementation:
 const { data, error, isLoading } = useView({
    moduleName: "vesting",
    functionName: "get_streams_for_user",
    args: [account?.address as `0x${string}`],
  });
```

**What this does:**
- Fetches streams specific to the connected wallet address
- Uses the `get_streams_for_user` function from the vesting module
- Maintains reactive state management with loading and error states

## Step 18: Implementing Claim Stream Function

Implement the `claimStream` function to allow users to claim available tokens from their vesting streams.

```tsx
const claimStream = async (streamId: string, amount: number) => {
    try {
      await submitTransaction("claim", [streamId, amount])
      const response = transactionResponse as any
      if (response && response.transactionHash) {
        toast({
          title: "Claim Successful",
          description: `Claim of ${amount} tokens successful.`,
          action: (
            <a
              href={`https://explorer.movementnetwork.xyz/txn/${response?.transactionHash}`}
              target="_blank"
              rel="noopener noreferrer"
            >
              <ToastAction altText="View transaction">View txn</ToastAction>
            </a>
          ),
        })
        return response.transactionHash
      } else {
        throw new Error("Transaction hash not available or transaction failed.")
      }
    } catch (error) {
      console.error("Error claiming stream:", error)
      toast({
        variant: "destructive",
        title: "Claim Failed",
        description: "Failed to claim tokens. Please try again.",
      })
      throw error
    }
  }

  // Calculate vesting progress and available amount
  const calculateProgress = (stream: any) => {
    console.log(currentTime)
    const elapsedTime = currentTime - (stream.start_time * 1000);
    if (elapsedTime < (stream.cliff * 1000)) {
      return 0;
    } else if (elapsedTime >= stream.duration) {
      return 100;
    } else {
      return (elapsedTime / stream.duration) * 100;
    }
  }
```

Key Features:

Submits claim transaction with stream ID and amount
Handles transaction success with toast notification
Provides transaction explorer link
Proper error handling with user feedback

Step 19: Implementing Progress Calculation
Implement the calculateProgress function to show vesting progress for each stream.

```tsx
  const calculateProgress = (stream: any) => {
    console.log(currentTime)
    const elapsedTime = currentTime - (stream.start_time * 1000);
    if (elapsedTime < (stream.cliff * 1000)) {
      return 0;
    } else if (elapsedTime >= stream.duration) {
      return 100;
    } else {
      return (elapsedTime / stream.duration) * 100;
    }
  }
```

**What this does:**
- Calculates time elapsed since stream start
- Checks cliff period
- Returns completion percentage
- Handles edge cases (before cliff, after completion)

## Step 20: Implementing Available Amount Calculation

Implement the `calculateAvailable` function to determine claimable tokens.
```tsx
  const calculateAvailable = (stream: any) => {
    const progress = calculateProgress(stream) / 100
    const totalVested = Number.parseFloat(stream.total_amount) * progress
    const available = (totalVested - Number.parseFloat(stream.claimed_amount)) / decimal
    console.log("Available:", available) 
    return Math.max(0, available).toFixed(2)
  }
```

Key Features:

Uses progress calculation
Accounts for previously claimed amounts
Formats output with proper decimal places
Ensures non-negative results

Step 21: Implementing Date Formatting
Implement the formatDate function for consistent date display.

```typescript
const formatDate = (timestamp: number) => {
  // Convert timestamp to readable date string in Europe/London timezone
  return new Date(timestamp * 1000).toLocaleDateString("en-US", {
    year: "numeric",
    month: "short",
    day: "numeric",
    hour: "numeric",
    minute: "2-digit",
    timeZone: "Europe/London",
  })
}
```

What this does:

Converts Unix timestamp to readable format
Uses specified timezone
Formats with month, day, year, and time

Step 22: Implementing Claim Amount Handler
Implement the handleClaimAmountChange function for input handling.

```typescript
const handleClaimAmountChange = (streamId: string, value: string) => {
  // Update claimAmounts state with new value
  setClaimAmounts((prev) => ({
    ...prev,
    [streamId]: value,
  }))
}
```


What this does:

Updates state immutably
Maintains claim amounts for each stream
Handles input changes reactively

Steps 23-30: Connecting UI Components
Connect the implemented functions to the UI components.
TODO 23: Connect start date
// Replace:
Started {'TBD'}
// With:
Started {formatDate(stream.start_time)}

TODO 24: Connect claim amount input
// Replace:
onChange={() => {}}
// With:
onChange={(e) => handleClaimAmountChange(stream.stream_id, e.target.value)}

TODO 25: Connect claim button
// Replace:
onClick={() => {}}
// With:
onClick={() => {
  const amount = Number(claimAmounts[stream.stream_id])
  if (amount > 0 && amount <= Number(calculateAvailable(stream))) {
    claimStream(stream.stream_id, amount)
  } else {
    toast({
      variant: "destructive",
      title: "Invalid Amount",
      description: "Please enter a valid amount within the available balance.",
    })
  }
}}

TODO 26-27: Connect progress display
// Replace:
<span className="font-medium text-yellow-400">{'0'}%</span>
style={{ width: `${0}%` }}
// With:
<span className="font-medium text-yellow-400">{calculateProgress(stream).toFixed(1)}%</span>
style={{ width: `${calculateProgress(stream)}%` }}

TODO 28: Connect available amount
// Replace:
{'0.00'} MOVE
// With:
{calculateAvailable(stream)} MOVE

TODO 29: Connect end date
// Replace:
<p className="font-semibold">{'TBD'}</p>
// With:
<p className="font-semibold">{formatDate(parseInt(stream.start_time) + parseInt(stream.duration))}</p>

TODO 30: Connect cliff date
// Replace:
Cliff: {'TBD'}
// With:
Cliff: {formatDate(parseInt(stream.start_time) + parseInt(stream.cliff))}

## Key Features Implemented

### 1. **File Upload & Processing**
- Supports XLS, XLSX, and CSV files
- Validates data format and structure
- Provides detailed error feedback
- Processes multiple streams at once

### 2. **Form Validation**
- Validates wallet addresses (64-character hex)
- Checks time format parsing
- Ensures positive amounts
- Provides real-time feedback

### 3. **Blockchain Integration**
- Uses Aptos wallet adapter
- Submits transactions to smart contract
- Handles transaction responses
- Provides transaction links

### 4. **User Experience**
- Loading states during processing
- Success/error notifications
- Form clearing after success
- Responsive design

### 5. **Security**
- Authorization checks
- Input validation
- Error handling
- Environment variable usage

## Time Format Parsing

The application supports flexible time formats:
- `1d` = 1 day
- `1min` = 1 minute  
- `1mon` = 1 month
- `1yr` = 1 year
- `30d` = 30 days
- `6mon` = 6 months

## Error Handling

The application includes comprehensive error handling:
- File reading errors
- Data validation errors
- Transaction failures
- Network issues
- Authorization failures

## Testing the Implementation

1. **Test File Upload**: Create an Excel file with columns: `wallet_address`, `amount`, `duration`, `cliff`
2. **Test Individual Creation**: Fill out the single stream form
3. **Test Authorization**: Connect with different wallet addresses
4. **Test Deposit**: Try depositing tokens
5. **Test Error Cases**: Try invalid addresses, amounts, or time formats

## Conclusion

This tutorial implemented a complete token vesting frontend interface with:
- Bulk stream creation via file upload
- Individual stream creation
- Token deposit functionality
- Stream management and viewing
- Comprehensive error handling
- Responsive UI design

The implementation provides a professional-grade admin interface for managing token vesting streams on the Aptos blockchain.