---
title: "Build a Token Vesting dApp with Movement"
description: "A practical guide to building a token vesting dApp on Movement's blockchain, enabling projects to create customizable vesting schedules for gradual token distribution to users."
date: "2025-04-04"
---

# Introduction
Token vesting is a powerful mechanism in the blockchain world, ensuring fair distribution and long-term commitment by locking tokens
for a set period before they can be accessed. Whether you're building a project for a decentralized team, a startup, or a community,
vesting helps align incentives and prevent premature sell-offs. In this tutorial, we’ll walk you through creating a token vesting
decentralized application (dApp) using the Move programming language on the Movement testnet.

In this guide, we'll walk you through building a token vesting decentralized application (dApp) using the Move programming language on the Movement
testnet. This dApp will enable projects to establish vesting schedules for team members, investors, and other stakeholders, ensuring a controlled and
gradual token release based on predefined conditions.


# Prerequisites

Before diving into the implementation, ensure you have the following prerequisites set up:

- **[Movement CLI](https://developer.movementnetwork.xyz/learning-paths/basic-concepts/01-install-movement-cli)**: Essential for interacting with the Movement blockchain.
- **[Aptos CLI](https://aptos.dev/cli-tools/aptos-cli-tool/install-aptos-cli)**: Required for certain operations compatible with Movement.
- **[Scaffold-Move](https://github.com/arjanjohan/scaffold-move)**: A development framework that streamlines building Move applications.
- **[Aptos SDK](https://aptos.dev/en/build/sdks/ts-sdk)**: Set up the Aptos SDK for interacting with the Movement testnet and managing blockchain operations.
- **[Node.js](https://nodejs.org/en/download/)**: Required for running Next.js and related frontend dependencies (version 16 or later recommended).
- An **IDE**: Use an integrated development environment like **[VS Code](https://code.visualstudio.com/)** or **[IntelliJ IDEA](https://www.jetbrains.com/idea/)** for writing and debugging code.
- **Basic familiarity with [Move](https://developer.movementnetwork.xyz/learning-paths/basic-concepts)**: A foundational understanding of the Move programming language is helpful.
- **Familiarity with [Scaffold-Move Hooks](https://scaffold-move-docs.vercel.app/hooks/)**: Understanding how to use the custom hooks provided by Scaffold-Move for blockchain interactions.
The complete code for this guide is available on GitHub [here](https://github.com/dumbdevss/vesting-tutorial).

## TL;DR

- Set up a Scaffold-Move project for streamlined Move development on the Movement testnet.
- Write a Move module to create and manage time-based vesting schedules.
- Develop a responsive Next.js dashboard for users to view and withdraw vested tokens.
- Deploy the vesting dApp to the Movement testnet with test cases using the Aptos SDK.

# Setting up Development Environment

To begin building our Token Vesting dApp, we'll set up an isolated and efficient development environment using a pre-bootstrapped repository. Follow these steps to get started:

1. **Clone the Repository**  
   Clone the tutorial repository, which has been bootstrapped with `scaffold-move` for Movement development. The `main` branch is the default:  
   ```bash
   git clone https://github.com/dumbdevss/vesting-tutorial
   cd vesting-tutorial
   ```

2. **Install Dependencies**
   Install the necessary project dependencies using Yarn:
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
   blockchain = "<your_generated_account_address>"
   ```

   > **Note**: If you encounter an error, navigate to the `packages/move` directory and use `movement init` or `aptos init`, depending on the CLI installed.

## Understanding the Project Structure

The project is structured to clearly separate the smart contract logic from the user interface components. Here's a breakdown:

- **`packages/move`**: This directory houses the Move smart contracts.
    - **`sources`**: Contains the core Move modules that define the logic for creating and managing NFT tickets.

- **`/frontend`**: This directory contains the Next.js application, which provides the user interface for interacting with the smart contracts.
    - **`/components`**: Reusable React components used throughout the application.
    - **`/app`**: Defines the application's routes and pages.
    - **`/hooks/scaffold-move`**: Custom React hooks that facilitate interaction with the Movement blockchain, leveraging `scaffold-move`.
    - **`/utils`**: Helper functions and utilities to support the frontend functionality.

This separation of concerns promotes a more maintainable and scalable codebase.

## Next Steps: Developing the Vesting Smart Contract

Now that we've set up our development environment and understood the project structure, let's move on to developing the core smart contract for our Token Vesting solution. In the following section, we'll write the Move code that will power our decentralized vesting platform, enabling projects to create customizable vesting schedules for gradual token distribution.