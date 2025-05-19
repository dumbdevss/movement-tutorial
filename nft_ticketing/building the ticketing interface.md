# NFT Ticketing Contract Tutorial - Frontend Integration

This tutorial will guide you through integrating your frontend application with our Move-based NFT ticketing smart contract. You'll learn how to implement ticket purchasing functionality using Scaffold hooks to create a seamless connection between your frontend and the blockchain.

## Implementing the Ticket Purchase Flow

### Step 1: Configure Pinata IPFS Storage

Add the following environment variables to your `.env` file:

```
NEXT_PUBLIC_PINATA_JWT = "your-pinata-jwt-token"
NEXT_PUBLIC_PINATA_GATEWAY = "your-pinata-gateway"
```

### Step 2: Set Up Required Imports

Navigate to `/app/events/page.tsx` and add the following imports:

```tsx
import useSubmitTransaction from '~~/hooks/scaffold-move/useSubmitTransaction';
import { useWallet } from '@aptos-labs/wallet-adapter-react';
import { useToast } from '~~/hooks/use-toast';
import { QRCodeSVG } from 'qrcode.react';
import ReactDOMServer from 'react-dom/server';
import PinataSDK from '@pinata/sdk';
```

### Step 3: Initialize Hooks and Services

Set up the required hooks to access the wallet, transaction submission, and toast notifications:

```tsx
const { submitTransaction, transactionResponse, transactionInProcess } = useSubmitTransaction("ticketing");
const { account } = useWallet();
const { toast } = useToast();
const pinata = new PinataSDK({
  pinataJwt: process.env.NEXT_PUBLIC_PINATA_JWT,
  pinataGateway: process.env.NEXT_PUBLIC_PINATA_GATEWAY,
});
```

### Step 4: Implement QR Code Generation and IPFS Upload

Create a function to generate a QR code and upload it to Pinata IPFS:

```tsx
const generateQRAndUploadToPinataIPFS = async (data: any): Promise<string> => {
  try {
    // Create QR code SVG element
    const qrCodeSvg = (
      <QRCodeSVG
        value={typeof data === 'string' ? data : JSON.stringify(data)}
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
    return url; // Returns the IPFS URL
  } catch (error) {
    console.error('Error generating QR code and uploading to IPFS:', error);
    throw error;
  }
};
```

### Step 5: Implement the Ticket Purchase Function

Create a function to handle the ticket purchase process:

```tsx
const handleBuyTicket = async (event: Event) => {
  const selectedType = selectedTicketTypes[event.id];
  const ticketTypeInfo = event.ticketTypes.find(t => t.type === selectedType);

  if (!ticketTypeInfo) return;

  try {
    // Determine which transaction type to use based on ticket type
    const transactionType =
      selectedType === "transferrable"
        ? "mint_transferable_ticket"
        : "mint_soulbound_ticket";

    // Generate QR code with ticket data and upload to IPFS
    let qrCodeUrl = await generateQRAndUploadToPinataIPFS({
      eventId: event.id,
      ticketType: selectedType,
      userAddress: account?.address,
    });
      
    // Submit the transaction to mint the NFT ticket
    await submitTransaction(transactionType, [
      parseInt(event.id),
      qrCodeUrl,
      `${event.name} ticket for user ${account?.address}`,
      `${event.image}`,
    ]);
    
    // Display success message
    toast({
      title: "Purchase Successful",
      description: `Successfully purchased ${selectedType} ticket for ${event.name} at ${ticketTypeInfo.price}`,
      duration: 5000,
    });
    
  } catch (error) {
    // Handle errors
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

## Implementing the User Dashboard

The user dashboard allows ticket holders to view and manage their NFT tickets. This component should be implemented in a separate file such as `/app/dashboard/page.tsx`.

### Key Features to Implement:

1. **Fetch User's NFT Tickets**: Query the blockchain for tickets owned by the current user
2. **Display Ticket Information**: Show ticket details including event name, date, and type 
3. **QR Code Display**: Allow users to view their ticket QR codes for event entry
4. **Transfer Options**: For transferrable tickets, implement transfer functionality

## Ticket Validation Process

For event organizers, implement a ticket validation component that:

1. Scans the QR code from attendees
2. Verifies ticket authenticity on the blockchain
3. Marks tickets as "used" to prevent double entry
4. Provides clear validation feedback to event staff

## Best Practices

- **Error Handling**: Always provide clear feedback for failed transactions
- **Loading States**: Implement loading indicators during blockchain interactions
- **Responsive Design**: Ensure your UI works well on both desktop and mobile devices
- **Wallet Connection**: Add fallback options when users don't have a wallet connected

## Conclusion

This frontend integration provides a complete user e