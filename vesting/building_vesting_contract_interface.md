# Token Vesting Frontend Interface Tutorial - Complete Implementation Guide

This tutorial will guide you through building a complete admin dashboard for a token vesting smart contract using React, TypeScript, and the Aptos blockchain. You'll learn how to implement bulk stream creation, individual stream management, and user-facing claim functionality to create a professional-grade vesting platform.

## Understanding the Architecture

Before diving into implementation, let's understand how our token vesting system works:

**Our Tech Stack:**

- **Move Smart Contract**: Handles vesting logic, stream creation, and token distribution on Aptos
- **React Frontend**: Provides admin dashboard and user interface for stream management
- **TypeScript**: Ensures type safety and better development experience
- **Aptos Wallet Adapter**: Handles blockchain interactions and transaction signing
- **SheetJS (XLSX)**: Processes Excel/CSV files for bulk stream creation

**System Components:**

- **Admin Dashboard**: Create and manage vesting streams, deposit tokens
- **User Interface**: View personal streams, claim vested tokens, track progress
- **Smart Contract**: Enforces vesting rules, handles token distribution securely

## Source Code Reference

The complete implementation of the token vesting interface can be found in the repository:

**Repository:** [dumbdevss/vesting-tutorial](https://github.com/dumbdevss/vesting-tutorial)
**Branch:** `frontend-implementation`

You can view or clone the finished code to compare against your implementation as you follow along with this guide.

## Understanding Token Vesting Concepts

**What is Token Vesting?**
Token vesting is a mechanism that releases tokens gradually over time, preventing immediate sell-offs and aligning long-term incentives. Key concepts include:

- **Vesting Schedule**: The timeline over which tokens are released
- **Cliff Period**: An initial period where no tokens are released
- **Stream**: A continuous flow of tokens to a recipient over time
- **Claimable Amount**: Tokens that have vested and can be withdrawn

**Why Use Smart Contracts for Vesting?**
Smart contracts provide:

- **Trustless Operation**: No intermediary can modify vesting terms
- **Transparency**: All vesting schedules are publicly verifiable
- **Automated Distribution**: Tokens release automatically based on time
- **Immutable Rules**: Vesting terms cannot be changed after creation

## Admin Dashboard Implementation (/app/admin)

Navigate to the admin dashboard component to implement TODOs 1-16.

### Understanding Data Fetching in Blockchain Applications

**View Functions vs. Transactions:**

- **View Functions**: Read blockchain state without gas costs
- **Transactions**: Modify blockchain state, require gas fees and signatures

**The useView Hook:**
This hook abstracts blockchain data fetching, providing:
- Automatic loading state management
- Error handling for network issues
- Reactive updates when blockchain state changes

### TODO 1: Implement data fetching for all streams

First, let's fetch all vesting streams from the smart contract:

```tsx
// Replace the placeholder data fetching:
const { data, error, isLoading } = {
  data: [[]],
  error: null,
  isLoading: false
}

// With the actual useView hook implementation:
const { data, error, isLoading } = useView({
  moduleName: "vesting",
  functionName: "get_all_streams",
})
```

**Implementation Notes:**

- `moduleName`: References our Move module containing vesting functions
- `functionName`: The specific view function returning all streams
- No arguments needed since we're fetching all streams globally
- Returns data, error state, and loading state for comprehensive UI handling

### Understanding Authorization in Decentralized Applications

**Why Authorization Matters:**
Admin functions should only be accessible by authorized addresses to prevent:

- Unauthorized stream creation
- Malicious token deposits
- Unauthorized access to sensitive operations

**Environment Variables for Security:**
Storing the module address in environment variables:

- Keeps sensitive data out of the codebase
- Allows different configurations for different environments
- Prevents accidental exposure of production addresses

### TODO 2: Implement authorization check

Implement proper authorization by comparing wallet addresses:

```tsx
// Replace the hardcoded authorization:
const isAuthorized = false // Replace with actual check against module address

// With the actual authorization check:
const isAuthorized = account?.address === process.env.NEXT_PUBLIC_MODULE_ADDRESS
```

**Implementation Notes:**

- Compares the connected wallet address with the authorized module owner
- Uses environment variables for security best practices
- Only authorized users can access admin functions
- Provides clear separation between admin and user interfaces

### Understanding File Processing for Bulk Operations

**Why Support Bulk Operations?**
Creating vesting streams individually is inefficient for large distributions. Bulk operations enable:

- Processing hundreds of recipients simultaneously
- Reducing transaction costs through batching
- Minimizing human error in data entry
- Streamlining token distribution workflows

**File Format Considerations:**
Supporting multiple formats (XLS, XLSX, CSV) ensures compatibility with various data sources and user preferences.

### TODO 3: Implement file upload functionality

Implement the comprehensive file upload handler:

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

**Key Implementation Features:**

1. **File Reading**: Uses FileReader API to process files client-side
2. **Data Processing**: Converts Excel/CSV data to JSON format
3. **Validation**: Ensures wallet addresses, amounts, and time formats are valid
4. **Error Handling**: Provides detailed feedback for various failure scenarios
5. **State Management**: Updates component state with processed data
6. **User Feedback**: Sets loading, success, and error states appropriately

**Validation Rules:**

- **Wallet Address**: Must be 64-character hexadecimal string with '0x' prefix
- **Amount**: Must be positive number
- **Duration/Cliff**: Must be parseable time format (1d, 1min, 1mon, 1yr)

### Understanding Form State Management

**Why Array-Based State?**
Managing recipients, amounts, durations, and cliffs as separate arrays allows:

- Efficient batch processing
- Easy validation of individual entries
- Flexible form handling for dynamic input counts

### TODO 4: Implement input change handler

Implement the comprehensive input handler for all form fields:

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

**Implementation Notes:**

- Uses switch statement for clean field handling
- Maintains immutability with spread operator
- Supports indexed updates for array-based state
- Handles type conversion appropriately
- Provides debugging output for development

### Understanding Blockchain Transactions

**Transaction Lifecycle:**

1. **Preparation**: Validate data and prepare transaction parameters
2. **Submission**: Send transaction to blockchain network
3. **Confirmation**: Wait for network confirmation
4. **Response**: Handle success/failure and update UI

### TODO 5: Implement single stream creation

Implement the function to create individual vesting streams:

```tsx
const handleCreateStream = async (
  recipient: `0x${string}`,
  amount: number,
  duration: number,
  cliff: number
) => {
  try {
    const streamId = nanoid(15)
    await submitTransaction("create_stream", [
      recipient, 
      amount * decimal, 
      duration, 
      cliff, 
      streamId
    ])
    
    let response = transactionResponse as any
    if (response) {
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
      })
      
      // Clear form after successful creation
      setRecipients([])
      setAmounts([])
      setDurations([])
      setCliffs([])
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

**Key Features:**

1. **Unique Stream ID**: Uses nanoid for collision-resistant identifiers
2. **Decimal Conversion**: Multiplies amount by decimal factor for precision
3. **Transaction Submission**: Calls smart contract function with proper parameters
4. **User Feedback**: Provides success/error notifications with transaction links
5. **Form Reset**: Clears form data after successful submission
6. **Error Propagation**: Throws errors for upstream handling

### TODO 6: Implement bulk stream creation

Implement the function to create multiple streams simultaneously:

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
    
    await submitTransaction("create_multiple_streams", [
      recipients, 
      formatedAmounts, 
      durations, 
      cliffs, 
      streamIds
    ])

    let response = transactionResponse as any
    if (response) {
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
      })
    }
  } catch (error) {
    toast({
      title: "Error",
      description: `Failed to create streams: ${error}`,
      variant: "destructive",
    })
    throw error
  }
}
```

**Implementation Notes:**

- **Array Length Validation**: Ensures all input arrays have the same length
- **Batch ID Generation**: Creates unique IDs for all streams
- **Decimal Formatting**: Applies decimal conversion to all amounts
- **Single Transaction**: Creates all streams in one blockchain transaction
- **Comprehensive Error Handling**: Provides detailed feedback for failures

### TODO 7: Implement token deposit functionality

Implement the function to deposit tokens into the vesting contract:

```tsx
const handleDeposit = async (amount: number) => {
  try {
    await submitTransaction("deposit", [amount * decimal])
    
    let response = transactionResponse as any
    if (response) {
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
      })
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

**Why Token Deposits Are Necessary:**

- Vesting contracts need token reserves to distribute
- Deposits ensure sufficient liquidity for all streams
- Admin control over token supply prevents over-distribution

### TODOs 8-16: Connect event handlers to UI components

Now we need to connect all our functions to the UI elements:

```tsx
// TODO 8: Connect file upload
onChange={handleFileUpload}

// TODO 9: Connect bulk stream creation
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

// TODO 10-13: Connect individual form inputs
onChange={(e) => handleInputChange(e, 0)}

// TODO 14: Connect individual stream creation
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

// TODO 15: Connect deposit input
onChange={(e) => handleInputChange(e, 0)}

// TODO 16: Connect deposit button
onClick={() => {
  if (depositAmount > 0) {
    handleDeposit(depositAmount)
  }
}}
```

**Input Validation on Click:**
Each button click validates required fields before proceeding, preventing unnecessary transaction attempts.

## User Interface Implementation (/app/dashboard)

Navigate to the user streams dashboard page to implement TODOs 17-30.

### Understanding User-Facing Stream Management

**User Needs:**

- View their personal vesting streams
- See vesting progress and available amounts
- Claim vested tokens when available
- Track important dates (start, cliff, end)

**Real-Time Calculations:**
Vesting progress and available amounts must be calculated in real-time based on current timestamp and stream parameters.

### TODO 17: Implement user-specific data fetching

Fetch streams specific to the connected user:

```tsx
// Replace the placeholder:
const { data, error, isLoading } = {
  data: [[]],
  error: null,
  isLoading: false
}

// With the user-specific implementation:
const { data, error, isLoading } = useView({
  moduleName: "vesting",
  functionName: "get_streams_for_user",
  args: [account?.address as `0x${string}`],
})
```

**Implementation Notes:**

- Uses `get_streams_for_user` function for efficient filtering
- Passes user's wallet address as argument
- Returns only streams where user is the recipient
- Maintains reactive state updates

### Understanding Vesting Mathematics

**Vesting Progress Calculation:**
Progress = (Current Time - Start Time) / Total Duration

**Key Considerations:**

- **Cliff Period**: No tokens available until cliff time passes
- **Linear Vesting**: Tokens vest continuously after cliff
- **Completion**: 100% vested when duration expires

### TODO 18: Implement stream claiming functionality

Implement the function to claim vested tokens:

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
```

**Key Features:**

1. **Partial Claims**: Users can claim any amount up to available balance
2. **Transaction Verification**: Checks for successful transaction hash
3. **User Feedback**: Provides immediate success/failure notifications
4. **Explorer Integration**: Links to transaction details
5. **Error Recovery**: Graceful handling of failed claims

### TODO 19: Implement vesting progress calculation

Implement the function to calculate vesting progress:

```tsx
const calculateProgress = (stream: any) => {
  console.log(currentTime)
  const elapsedTime = currentTime - (stream.start_time * 1000)
  
  if (elapsedTime < (stream.cliff * 1000)) {
    return 0
  } else if (elapsedTime >= stream.duration) {
    return 100
  } else {
    return (elapsedTime / stream.duration) * 100
  }
}
```

**Progress Calculation Logic:**

1. **Before Cliff**: 0% progress regardless of time elapsed
2. **After Cliff, Before End**: Linear progress based on elapsed time
3. **After End**: 100% progress (fully vested)

**Time Conversion Notes:**

- Smart contract stores timestamps in seconds
- JavaScript Date works with milliseconds
- Conversion factor: multiply by 1000

### TODO 20: Implement available amount calculation

Implement the function to calculate claimable tokens:

```tsx
const calculateAvailable = (stream: any) => {
  const progress = calculateProgress(stream) / 100
  const totalVested = Number.parseFloat(stream.total_amount) * progress
  const available = (totalVested - Number.parseFloat(stream.claimed_amount)) / decimal
  console.log("Available:", available) 
  return Math.max(0, available).toFixed(2)
}
```

**Available Amount Formula:**
Available = (Total Amount × Progress) - Already Claimed

**Implementation Notes:**

- Converts progress percentage to decimal
- Accounts for previously claimed amounts
- Applies decimal conversion for display
- Ensures non-negative results
- Formats to 2 decimal places for readability

### TODO 21: Implement date formatting

Implement consistent date formatting across the interface:

```tsx
const formatDate = (timestamp: number) => {
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

**Date Formatting Features:**

- Converts Unix timestamp to readable format
- Uses consistent timezone (Europe/London)
- Includes both date and time information
- Handles localization properly

### TODO 22: Implement claim amount input handler

Implement the handler for claim amount changes:

```tsx
const handleClaimAmountChange = (streamId: string, value: string) => {
  setClaimAmounts((prev) => ({
    ...prev,
    [streamId]: value,
  }))
}
```

**State Management Pattern:**

- Uses object with streamId as key
- Maintains immutability with spread operator
- Allows independent claim amounts per stream
- Supports real-time input validation

### TODOs 23-30: Connect UI components with calculated values

Connect all the calculated values and handlers to UI elements:

```tsx
// TODO 23: Connect start date
Started {formatDate(stream.start_time)}

// TODO 24: Connect claim amount input
onChange={(e) => handleClaimAmountChange(stream.stream_id, e.target.value)}

// TODO 25: Connect claim button with validation
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

// TODO 26-27: Connect progress display
<span className="font-medium text-yellow-400">
  {calculateProgress(stream).toFixed(1)}%
</span>
<div style={{ width: `${calculateProgress(stream)}%` }} />

// TODO 28: Connect available amount
{calculateAvailable(stream)} MOVE

// TODO 29: Connect end date
<p className="font-semibold">
  {formatDate(parseInt(stream.start_time) + parseInt(stream.duration))}
</p>

// TODO 30: Connect cliff date
Cliff: {formatDate(parseInt(stream.start_time) + parseInt(stream.cliff))}
```

**UI Integration Features:**

1. **Real-Time Updates**: All values update automatically as time progresses
2. **Input Validation**: Claim amounts validated against available balance
3. **Visual Progress**: Progress bars show vesting status visually
4. **Comprehensive Information**: All relevant dates and amounts displayed

## Understanding Time Format Parsing

**Supported Time Formats:**
Our application supports flexible time input formats for user convenience:

- `1d` = 1 day (86,400 seconds)
- `1min` = 1 minute (60 seconds)
- `1mon` = 1 month (2,592,000 seconds)
- `1yr` = 1 year (31,536,000 seconds)
- `30d` = 30 days
- `6mon` = 6 months

**Why Flexible Formats Matter:**

- Users think in human terms (days, months, years)
- Reduces input errors
- Improves user experience
- Maintains precision in smart contract

## Error Handling Strategies

**Comprehensive Error Coverage:**

1. **File Processing Errors**: Invalid formats, missing columns, corrupt data
2. **Validation Errors**: Invalid addresses, negative amounts, unparseable times
3. **Network Errors**: Connection issues, transaction failures
4. **Smart Contract Errors**: Insufficient funds, unauthorized access
5. **User Input Errors**: Invalid claim amounts, missing fields

**Error Communication:**
- **Toast Notifications**: Immediate feedback for user actions
- **Console Logging**: Detailed information for debugging
- **Error States**: UI components show error conditions
- **Graceful Degradation**: App continues functioning despite errors

## Conclusion

You've successfully built a comprehensive token vesting platform with both admin and user interfaces. The implementation includes:

**Admin Capabilities:**

- Bulk stream creation via file upload
- Individual stream management
- Token deposit functionality
- Comprehensive validation and error handling

**User Features:**

- Personal stream viewing
- Real-time vesting progress tracking
- Flexible token claiming
- Detailed stream information

**Technical Achievements:**

- Robust file processing and validation
- Secure blockchain integration
- Professional UI/UX patterns
- Comprehensive error handling
- Real-time calculations and updates

**Next Steps:**
Extend this foundation with advanced features like:

- Multi-token support
- Flexible vesting curves (non-linear)
- Stream modification capabilities
- Advanced analytics and reporting
- Multi-signature admin controls

This vesting platform provides a solid foundation for token distribution and demonstrates professional blockchain application development practices.