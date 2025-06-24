# NFT Ticketing Contract Tutorial - Frontend Integration

This tutorial guides you through integrating your frontend application with a Move-based NFT ticketing smart contract. You'll learn how to implement ticket purchasing, user dashboard, and ticket transfer functionalities using Scaffold hooks to create a seamless connection between your frontend and the blockchain.

## Understanding the Architecture

Before diving into the implementation, let's understand how our NFT ticketing system works:

**Our Tech Stack:**

- **Move Smart Contract**: Manages ticket minting, ownership, and transfers
- **IPFS (via Pinata)**: Stores QR codes and metadata for tickets
- **React Frontend**: Provides the user interface for ticket management
- **Scaffold Hooks**: Abstracts blockchain interactions into simple React hooks

## Source Code Reference

The complete implementation of the frontend interface can be found in the repository:

**Repository:** [dumbdevss/nft-ticketing](https://github.com/dumbdevss/nft-ticketing)  
**Branch:** `frontend-integration`

## Dashboard Page (/app/dashboard)

Navigate to the dashboard page component (`/app/dashboard/page.tsx`) to implement TODOs 1-4.

### Understanding Blockchain Data Queries

**What are View Functions?**
View functions read blockchain state without modifying it, making them ideal for querying ticket data:

- No gas fees.
- Instant execution.
- Safe for repeated calls.

**The useView Hook:**
Simplifies blockchain queries by handling data fetching, errors, and loading states.

### TODO 1: Define WalletAccount Type

Define the `WalletAccount` interface:

```tsx
interface WalletAccount {
  address: string;
}
```

**Implementation Notes:**

- Ensures type safety for wallet-related operations.
- Used to type the `useWallet` hook response.

### TODO 2: Use useWallet Hook

Get the connected wallet and address using the `useWallet` hook:

```tsx
const { account } = useWallet() as { account: WalletAccount | null };
```

**Implementation Notes:**

- Retrieves the user's wallet address for querying tickets.
- Handles cases where no wallet is connected.

### TODO 3: Get User Tickets Using useView Hook

Fetch user tickets with the `useView` hook:

```tsx
const { data, error, isLoading, refetch } = useView({
  moduleName: "ticketing",
  functionName: "get_tickets_by_user",
  args: account?.address ? [`${account.address}`] : [],
}) as UseViewResponse<[Ticket[]]>;
```

**Implementation Notes:**

- Queries the `get_tickets_by_user` view function.
- Passes the user's address as an argument.
- Returns ticket data, error, and loading states.

### TODO 4: Get Reformatted User Tickets Data

Reformat ticket data to include event details:

```tsx
const enrichedTickets: EnrichedTicket[] = data?.[0]?.map((ticket: Ticket) => {
  const events: Event[] = JSON.parse(localStorage.getItem("events") || "[]");
  const event = events.find((e) => e.id === ticket.event_id);

  return {
    ...ticket,
    name: event?.name || "Unknown Event",
    date: event?.date || "Unknown Date",
    location: event?.location || "Unknown Location",
    image: event?.image,
  };
}) || [];

localStorage.setItem(`tickets-${account?.address}`, JSON.stringify(enrichedTickets));
```

**Implementation Notes:**

- Enriches tickets with event details from `localStorage`.
- Stores enriched tickets in `localStorage` for offline access.
- Handles cases where event data is missing.

## Events Page (/app/events)

Navigate to the events page component (`/app/events/page.tsx`) to implement TODOs 5-8.

### Understanding IPFS Integration

**Why IPFS for QR Codes?**
Storing QR codes on IPFS ensures decentralized access and immutability. Only the IPFS hash is stored on the blockchain, reducing costs while maintaining integrity.

**Why Pinata?**
Pinata ensures QR codes remain accessible by pinning them to IPFS nodes, preventing data loss.

### TODO 5: Create a Pinata Client

The Pinata client is initialized using environment variables:

```tsx
const pinata = new PinataSDK({
  pinataJwt: process.env.NEXT_PUBLIC_PINATA_JWT,
  pinataGateway: process.env.NEXT_PUBLIC_PINATA_GATEWAY,
});
```

**Implementation Notes:**

- Environment variables (`NEXT_PUBLIC_PINATA_JWT` and `NEXT_PUBLIC_PINATA_GATEWAY`) are stored in the `.env` file.
- The gateway URL constructs accessible IPFS file URLs.
- Variables prefixed with `NEXT_PUBLIC_` are available client-side.

Ensure your `.env` file includes:

```
NEXT_PUBLIC_PINATA_JWT = "your-pinata-jwt-token"
NEXT_PUBLIC_PINATA_GATEWAY = "your-pinata-gateway"
```

### TODO 6: Generate QR Code and Upload to Pinata

Implement the `generateQRAndUploadToPinataIPFS` function to create and upload QR codes:

```tsx
const generateQRAndUploadToPinataIPFS = async (data: any): Promise<string> => {
  try {
    // Create QR code SVG element
    const qrCodeSvg = (
      <QRCodeSVG
        value={typeof data === 'string' ? data : `http://localhost:3000/validate/${data.userAddress}_${data.eventId}`}
        size={250}
        level={"H"}
        includeMargin={true}
      />
    );
    
    // Convert React element to SVG string
    const svgString = ReactDOMServer.renderToStaticMarkup(qrCodeSvg);
    const svgBlob = new Blob([svgString], { type: 'image/svg+xml' });
    const svgFile = new File([svgBlob], "qrcode.svg", { type: "image/svg+xml" });
    
    const upload = await pinata.upload.public.file(svgFile);
    let url = `https://${process.env.NEXT_PUBLIC_PINATA_GATEWAY}/ipfs/${upload.cid}`;
    return url; // Returns the IPFS CID
  } catch (error) {
    console.error('Error generating QR code and uploading to IPFS:', error);
    throw error;
  }
};
```

**Implementation Notes:**

- The QR code encodes a validation URL with the user's address and event ID.
- The SVG is converted to a file for Pinata upload.
- Error handling ensures robust operation.

### TODO 7: Implement Ticket Purchase Function

Implement the `handleBuyTicket` function to mint NFT tickets:

```tsx
const handleBuyTicket = async (event: Event) => {
  const selectedType = selectedTicketTypes[event.id];
  const ticketTypeInfo = event.ticketTypes.find(t => t.type === selectedType);

  if (!ticketTypeInfo) return;

  try {
    // Handle NFT minting based on selectedType
    const transactionType =
      selectedType === "transferrable"
        ? "mint_transferable_ticket"
        : "mint_soulbound_ticket";

    let qrCodeUrl = await generateQRAndUploadToPinataIPFS({
      eventId: event.id,
      userAddress: account?.address,
    });

    await submitTransaction(transactionType, [
      parseInt(event.id),
      qrCodeUrl,
      `${event.name} ticket for user ${account?.address}`,
      `${event.image}`,
    ]);

    // Success toast after successful transaction
    toast({
      title: "Purchase Successful",
      description: `Successfully purchased ${selectedType} ticket for ${event.name} at ${ticketTypeInfo.price}`,
      duration: 5000,
    });
  } catch (error) {
    // Error toast if transaction fails
    toast({
      title: "Purchase Failed",
      description: `Failed to purchase ticket for ${event.name}. Please try again.`,
      variant: "destructive",
      duration: 5000,
    });
    console.error("Transaction error:", error);
  }
};
```

**Implementation Notes:**

- Determines transaction type based on ticket type (transferrable or soulbound).
- Generates and uploads a QR code to IPFS.
- Submits the transaction using the `submitTransaction` hook.
- Provides user feedback via toast notifications.

### TODO 8: Implement Buy Ticket Button Handler

Connect the buy ticket button to the `handleBuyTicket` function:

```tsx
<Button
  className="w-full"
  onClick={() => handleBuyTicket(event)}
>
  Buy NFT Ticket
</Button>
```

**Implementation Notes:**

- The button is already wired to call `handleBuyTicket` with the event object.
- Ensures seamless user interaction for purchasing tickets.

## Ticket Page (/app/ticket/[id])

Navigate to the ticket page component (`/app/ticket/[id]/page.tsx`) to implement TODOs 9-11.

### Understanding Ticket Transfers

**Why Transfer Tickets?**
Transferrable tickets allow users to send NFTs to other wallets, enabling secondary markets or gifting.

**Blockchain-Based Transfers:**
The smart contract ensures only the ticket owner can transfer, and transfers are recorded immutably.

### TODO 9: Define GraphQL Query for User NFTs

Define the GraphQL query to fetch user NFTs:

```tsx
const GET_USER_NFTS_QUERY = gql`
  query GetUserNFTs($ownerAddress: String!) {
    current_token_ownerships_v2(
      where: {
        owner_address: {_eq: $ownerAddress},
        amount: {_gt: 0},
        is_fungible_v2: {},
        current_token_data: {
          current_collection: {collection_name: {_eq: "Event Tickets"}}
        }
      }
    ) {
      token_data_id
      amount
      current_token_data {
        token_name
        token_uri
        token_properties
        collection_id
        current_collection {
          collection_name
          description
          uri
          creator_address
        }
      }
    }
  }
`;
```

**Implementation Notes:**

- Queries NFTs owned by the user in the "Event Tickets" collection.
- Includes token metadata for matching with ticket data.
- Uses a variable for the owner address.

### TODO 10: Implement Transfer Function

Implement the `handleTransfer` function:

```tsx
const handleTransfer = async (ticket: Ticket, recipientAddress: string) => {
  let nft = onChainNftData.find((nft) => nft.event_image === ticket.image);
  if (!nft) {
    console.error("NFT not found in onChainNftData");
    return;
  }

  if (!recipientAddress) {
    console.error("Recipient address is empty");
    return;
  }

  // Check if the recipient address starts with "0x"
  if (recipientAddress.startsWith("0x")) {
    recipientAddress = recipientAddress.slice(2);
  }

  try {
    // Call the transfer function from the smart contract
    await submitTransaction("transfer_ticket", [
      `0x${recipientAddress}`,
      nft.token_address,
    ]);

    console.log(`Transferring ticket ${ticket.ticket_id} to ${recipientAddress}`);
    setRecipientAddress("");
  } catch (error) {
    console.error("Transfer error:", error);
  }
};
```

**Implementation Notes:**

- Matches the ticket with on-chain NFT data using the image URL.
- Normalizes the recipient address format.
- Submits the transfer transaction using the `submitTransaction` hook.
- Clears the input field on success.

### TODO 11: Implement Transfer Button Handler

Connect the transfer button to the `handleTransfer` function:

```tsx
<Button
  disabled={!recipientAddress}
  onClick={() => handleTransfer(ticket, recipientAddress)}
  className="bg-gradient-to-r from-blue-500 to-blue-700"
>
  Transfer
</Button>
```

**Implementation Notes:**

- Disables the button when no recipient address is entered.
- Triggers the transfer process on click.

## Testing the Application

### 1. Running the Application Locally

Start the development server:

```bash
npm run dev
```

This will launch the frontend at [http://localhost:3000](http://localhost:3000).

### 2. Testing Features

- **Connect Wallet**: Ensure wallet connection works and displays the correct address.
- **Purchase Ticket**: Buy both transferrable and soulbound tickets; verify transactions and QR code generation.
- **Dashboard**: Check that purchased tickets appear with correct event details.
- **Transfer Ticket**: For transferrable tickets, test sending to another wallet address.
- **Error Handling**: Simulate errors (e.g., invalid address, failed transaction) and verify user feedback.

## Best Practices Implemented

- **Error Handling**: Toast notifications provide feedback for successful and failed transactions (e.g., in `handleBuyTicket` and `handleTransfer`).
- **Loading States**: The `isLoading` state in `/app/ticket/[id]/page.tsx` shows a loading indicator during data fetching.
- **Responsive Design**: Tailwind CSS ensures a responsive layout (e.g., `grid-cols-1 md:grid-cols-2 lg:grid-cols-3`).
- **Wallet Connection**: Checks for `account?.address` prevent errors when no wallet is connected.

## Conclusion

You've successfully built a complete NFT ticketing frontend integrated with a Move-based smart contract. Key features include:

1. **Ticket Purchasing**: Minting transferrable or soulbound NFT tickets.
2. **QR Code Storage**: Generating and storing QR codes on IPFS via Pinata.
3. **User Dashboard**: Displaying owned tickets with event details.
4. **Ticket Transfers**: Enabling secure ticket transfers for non-soulbound tickets.

**Key Takeaways:**

- Combining blockchain and IPFS provides cost-effective, decentralized storage.
- Scaffold hooks simplify blockchain interactions.
- User feedback and state management are critical for dApp usability.
- Smart contract design impacts frontend implementation.

**Next Steps:**
Extend the system with features like:

- Ticket validation at events.
- Secondary market integration.
- Event reminders.
- Multi-ticket purchases.
- Ticket categories.

Next Tutorial: [Testing the NFT Ticketing DApp](./testing-nft-ticketing.md)
