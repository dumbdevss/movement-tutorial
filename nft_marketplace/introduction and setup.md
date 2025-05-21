# Build an NFT Marketplace on Movement

---
title: "Build an NFT Marketplace on Movement"
description: "A practical guide to building a decentralized NFT marketplace on Movement's blockchain, enabling listing, buying,
selling, and auctioning of digital assets."
date: "2025-05-21"
---

## Introduction

The emergence of NFT marketplaces has transformed how we think about digital asset ownership. These platforms create new opportunities for creators to monetize their work and for collectors to acquire verifiable, unique digital items. By leveraging Movement's blockchain technology, we can build marketplaces that improve upon existing solutions - offering stronger security guarantees, reduced transaction costs, and enhanced performance for trading digital assets.

In this guide, we'll walk you through building an NFT marketplace decentralized application (dApp) using the Move programming language on the Movement testnet. This marketplace will support instant purchases, auctions, listing management, and secure ownership transfers.

## Prerequisites

Before diving into the implementation, ensure you have the following prerequisites set up:

- **[Movement CLI](https://developer.movementnetwork.xyz/learning-paths/basic-concepts/01-install-movement-cli)**: Essential for interacting with the Movement blockchain
- **[Aptos CLI](https://aptos.dev/cli-tools/aptos-cli-tool/install-aptos-cli)**: Required for certain operations compatible with Movement
- **[Scaffold-Move](https://github.com/arjanjohan/scaffold-move)**: Development framework that streamlines building Move applications
- **[Aptos SDK](https://aptos.dev/en/build/sdks/ts-sdk)**: For interacting with the Movement testnet and managing blockchain operations
- **[Node.js](https://nodejs.org/en/download/)**: Required for running Next.js and related frontend dependencies (version 16 or later recommended)
- **[IPFS/Pinata](https://www.pinata.cloud/)**: For decentralized storage of NFT metadata and content
- An **IDE**: Use VS Code or IntelliJ IDEA for writing and debugging code
- **Basic familiarity with [Move](https://developer.movementnetwork.xyz/learning-paths/basic-concepts)**: A foundational understanding of the Move programming language
- **Familiarity with [Scaffold-Move Hooks](https://scaffold-move-docs.vercel.app/hooks/)**: Understanding how to use the custom hooks for blockchain interactions

The complete code for this guide is available on GitHub [here](https://github.com/dumbdevss/nft-marketplace).

## TL;DR

- Set up a Scaffold-Move project for streamlined Move development on the Movement testnet
- Write a Move module to create and manage an NFT marketplace with instant buy, auction, listing, and delisting functionalities
- Develop a responsive Next.js dashboard for users to mint, list, bid on, and purchase NFTs
- Deploy the NFT marketplace dApp to the Movement testnet with comprehensive test cases

## Setting up Development Environment

To begin building our NFT marketplace dApp, we'll set up an efficient development environment using a pre-bootstrapped repository:

1. **Clone the Repository**  
   Clone the tutorial repository, which has been bootstrapped with `scaffold-move` for Movement development:

   ```bash
   git clone https://github.com/dumbdevss/nft-marketplace
   cd nft-marketplace
   ```

2. **Install Dependencies**
   Install the necessary project dependencies:

   ```bash
   yarn install
   ```

3. **Generate a Deployment Account**
   Create a local account for interacting with the Movement testnet:

   ```bash
   yarn account
   ```

   After initialization, note the generated account address and update the `Move.toml` file:

   ```toml
   [addresses]
   nft_marketplace = "<your_generated_account_address>"
   ```

   > **Note**: If you encounter an error, navigate to the `packages/move` directory and use:

   ```bash
   movement init --network custom --rest-url https://testnet.bardock.movementnetwork.xyz/v1 --faucet-url https://faucet.testnet.bardock.movementnetwork.xyz/
   ```

## Understanding the Project Structure

The project is structured to clearly separate smart contract logic from user interface components:

- **`packages/move`**: Contains the Move smart contracts
  - **`sources`**: Core Move modules defining the logic for NFT marketplace operations

- **`/nextjs`**: Next.js application providing the user interface
  - **`/components`**: Reusable React components
  - **`/app`**: Application routes and pages
  - **`/hooks/scaffold-move`**: Custom React hooks for blockchain interaction
  - **`/utils`**: Helper functions and utilities

This separation of concerns promotes a maintainable and scalable codebase.

## Next Steps: Developing the Smart Contract

With our development environment set up and project structure understood, we'll next develop the core smart contract for our NFT marketplace. In the following section, we'll write the Move code that will power our decentralized marketplace, implementing features such as:

- NFT minting and metadata storage
- Instant purchase functionality
- Auction mechanisms with bidding
- Listing and delisting operations
- Secure ownership transfers

Next Tutorial: [Building the NFT Marketplace Smart Contract](./developing_smart_contract)

Stay tuned as we build a robust NFT marketplace on Movement's blockchain infrastructure.
