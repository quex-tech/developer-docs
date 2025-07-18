# Getting Started with Quex

# Introduction

Quex provides a protocol and infrastructure for decentralized data transfer based on confidential computing.  The most simple of these for developers is data oracle, and that’s what we’ll be learning about today.
In this guide, we’ll be conceptualizing, understanding and building a basic parametric emission token, which will give you some idea of how to use Quex for accessing verifiable data on-chain.

## Document Structure

This tutorial is organized into the following sections:

- **Introduction**: Provides an overview of Quex and explains what we're building.
- **Try it Out**: Quick start instructions to deploy and run the provided example.
- **Let’s Create a Quex-based Contract**: Step-by-step instructions to set up your environment and implement smart contracts leveraging Quex.
    - **Prepare Environment**: How to configure your development environment.
    - **Token Emission**: Detailed implementation of ERC-20 token emission logic.
    - **Create Flow**: Defines how data requests are structured and handled by Quex oracle pools.
    - **Deploy and Run**: Guide to deploying the contracts and executing the request to mint tokens.
- **Next Steps**: Recommendations on further exploration and advanced use cases.

## What Are We Building?

Let’s begin by clearly defining what we’re building and why. Imagine you’re issuing a token for a DeFi protocol and want to design tokenomics that incentivize the foundation to actively increase the protocol’s Total Value Locked (TVL). One effective approach is awarding the foundation tokens proportional to the protocol's TVL, as reported by [DeFiLlama](https://defillama.com).

Specifically, we'll design a token where the foundation receives freshly minted tokens daily, equal in number to the USD-denominated TVL of the protocol across all supported chains and protocol versions. For clarity and practical demonstration, we'll use TVL data from the [DyDx protocol](https://defillama.com/protocol/dydx). Fortunately, DeFiLlama provides an [API endpoint](https://api.llama.fi/tvl/dydx) that returns exactly this number. However, because this data is provided off-chain, achieving our goal requires an oracle to securely transfer the data on-chain.

# Try it Out

The codebase for this example, including a prepared Foundry environment, is available in [our examples repository](https://github.com/quex-tech/quex-v1-examples). You can easily explore it by cloning the repository and installing its dependencies:

```shell
git clone https://github.com/quex-tech/quex-v1-examples
cd quex-v1-examples/tvl-emission
forge install
```

This repository also provides scripts that help you deploy the contracts and make requests to the Quex data oracle, triggering token minting. To deploy these contracts, you’ll need the private key of a wallet holding sufficient gas tokens for your chosen network. Set your private key as an environment variable for convenience:

```shell
export SECRET=<0xYourPrivateKey>
```

## Build and Deploy Contracts

Run the DeployTVLEmissionScript to build and deploy the ERC-20 token, the token emission contract, and to set up the data flow for verifiable Quex HTTPS responses:

```shell
forge script script/DeployTVLEmissionScript.s.sol --broadcast
```

You’ll need the deployed contract address for making requests. Store the deployed contract’s address as an environment variable:

```shell
export CONTRACT_ADDRESS=<DEPLOYED_CONTRACT_ADDRESS>
```

## Make a Request

Next, make a request using the provided script:

```shell
forge script script/Request.s.sol --broadcast
```

## Check Your Balance

Now, verify your wallet balance—you should receive freshly minted TVLT tokens, equal in amount to the current TVL of the DyDx protocol!

Congratulations! You've successfully deployed your first contract that relies on confidential computing proofs to bring off-chain data to your contract. In the following sections, we'll provide a detailed explanation of what's happening under the hood of our contracts. This will give you insights and ideas for building your own Quex-based smart contracts.

# Let’s Create a Quex-based Contract!

## Prepare Environment

We'll build our example using [Foundry](https://book.getfoundry.sh) (Forge)—ensure it's correctly installed and up to date. Update Foundry by running:

```shell
foundryup
```

We’ll also use the ERC-20 implementation from OpenZeppelin and Quex libraries to simplify our development:

```shell
forge install openzeppelin/openzeppelin-contracts
forge install quex-tech/quex-v1-interfaces
```

## Token emission

Next, let's create a new contract `ParametricToken.sol` in src folder with the following code:

```solidity
pragma solidity ^0.8.22;

import "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
import "openzeppelin-contracts/contracts/access/Ownable.sol";

contract ParametricToken is ERC20, Ownable {
    constructor() ERC20("Parametric Emission Token", "TVLT") Ownable(msg.sender) {}

    function mint(address to, uint256 amount) external onlyOwner {
        _mint(to, amount);
    }
}
```

As you can see, this is a standard ERC-20 token with one modification: only the contract owner can mint new tokens. In our scenario, ownership of this token contract will belong to another contract.

Let’s create that contract, `TVLEmission.sol`, next:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.22;

import "./ParametricToken.sol";
import "quex-v1-interfaces/src/libraries/QuexRequestManager.sol";
/**
 * @title TVLEmission
 * @dev This contract manages token emissions based on Total Value Locked (TVL) data retrieved from Quex.
 * It ensures that emissions can only be processed once per day to prevent excessive token minting.
 */
contract TVLEmission is QuexRequestManager {
    address private _treasuryAddress;
    ParametricToken public parametricToken;
    uint256 public lastRequestTime;
    uint256 private constant REQUEST_COOLDOWN = 1 days;

    constructor(address treasuryAddress, address quexCore, address oraclePool) QuexRequestManager(quexCore) {
        parametricToken = new ParametricToken();
        _treasuryAddress = treasuryAddress;
    }

    /**
     * @notice Processes a response from Quex and mints tokens based on TVL.
     * @dev Ensures a minimum of 24 hours has passed since the last emission before minting new tokens.
     * @param receivedRequestId The ID of the request that is being processed.
     * @param response The response data from Quex, expected to contain the latest TVL value.
     */
    function processResponse(uint256 receivedRequestId, DataItem memory response, IdType idType)
        external
        verifyResponse(receivedRequestId, idType)
    {
        require(block.timestamp >= lastRequestTime + REQUEST_COOLDOWN, "Request cooldown active");
        uint256 lastTVL = abi.decode(response.value, (uint256));
        parametricToken.mint(_treasuryAddress, lastTVL);
        lastRequestTime = block.timestamp;
    }
}
```

Let’s briefly summarize what’s happening here:
- The contract imports and inherits from `QuexRequestManager`, which simplifies interactions with Quex oracle pools and performs the necessary response verification.
- The constructor is initialized with addresses for the token treasury, Quex core, and the [HTTPS oracle pool](../https_pool/https_pool.md). We also deploy the `ParametricToken` contract within the constructor, becoming its owner and gaining the ability to mint tokens.
- The `processResponse` method is implemented as a callback, which securely processes responses from the Quex oracle. It includes a cooldown period to limit request frequency and uses the `verifyResponse` modifier to protect against oracle manipulation attacks.

At this point, we should make a side note regarding Quex architecture. Although Quex supports classic pull- and push-based data oracles, under the hood, all of them rely on callback mechanics. In our example, we use the fastest and most cost-efficient callback scenario, particularly useful when you need to process data directly upon receiving it.

However, though we've defined a `processResponse` function to verify proofs and process responses, we haven't yet defined the data flow—the exact HTTPS request the oracle pool should perform and the necessary supporting information. We'll cover this in the next section.

## Create Flow

In Quex terminology, a `Flow` is a combination of the recipient contract address, recipient contract callback, callback gas limit, oracle pool address, and the ID of the action to be performed by the oracle. While there are multiple ways to define a `Flow`, for clarity in our example, we'll define it directly within our contract. For more options, explore [Flow creation](flow_creation.md) section of this documentation.

To achieve this, create a `setUpFlow()` function as follows and invoke it from the constructor:

```solidity
...
using FlowBuilder for FlowBuilder.FlowConfig;

contract TVLEmission is QuexRequestManager {
    constructor(address treasuryAddress, address quexCore, address oraclePool) QuexRequestManager(quexCore) {
        ...
        setUpFlow();
    }

    function setUpFlow(address quexCore, address oraclePool) private onlyOwner {
        FlowBuilder.FlowConfig memory config = FlowBuilder.create(quexCore, oraclePool, "api.llama.fi", "/tvl/dydx");
        config = config.withFilter(". * 1000000000000000000 | round");
        config = config.withSchema("uint256");
        config = config.withCallback(address(this), this.processResponse.selector);
        registerFlow(config);
    }
    
    ...
}
```

Let's go through the `setUpFlow()` method step by step. We'll use the `FlowBuilder` helper here, which simplifies flow registration by asking us to define only the necessary fields.

First, we define the [HTTPS request](../https_pool/https_pool.md#httprequest) we’d like to perform—in our case, it's [https://api.llama.fi/tvl/dydx](https://api.llama.fi/tvl/dydx), which returns the USD-denominated TVL of the DyDx protocol required for our task. Although our example is straightforward, Quex supports more advanced requests: you can specify headers, parameters, HTTP methods, and request bodies as needed. Additionally, Quex allows the use of private data (such as API credentials) by passing encrypted data securely to our oracles. While we don't directly demonstrate these advanced features in this tutorial, you can explore them in a [dedicated section](private_patch.md) of our documentation. These additional options provide flexibility when working with more complex APIs.

Second, we define a [filter](../https_pool/https_pool.md#jqfilter)—a script written in the [jq](https://jqlang.org) programming language—for response post-processing. The DeFiLlama API from the previous step returns a floating-point number, e.g., `284291310.4518468`. However, standard ERC20 tokens follow a convention where token amounts are represented as integers, scaled by 1e18. To achieve this, the jq script `". * 1000000000000000000 | round"` multiplies the response by 1e18 and rounds it to an integer.

Third, we define a response [schema](../https_pool/https_pool.md#responseschema)—this is the format for encoding the oracle response. In our example, the response (an unsigned numeric value) can be easily encoded as a Solidity `uint256`. However, you're not limited by this choice and can define more complex schemas if needed, learn more at our [shemas page](../https_pool/data_scheme.md). If you wish to explore example using more complex data structures, check out [this tutorial](./complex_structures_tutorial.md).

Finally, we need to define what happens when the response is ready. We do this using three fields: the oracle pool's address (`consumer`), the callback function (`callback`) to handle the incoming data, and a `gasLimit` for executing the callback. After defining these parameters, we call `createFlow()` to register our flow in the Quex registry, and store the returned identifier via `setFlowId()` for verifying incoming data later.

## Set up subscription

To simplify the money flow during request creation and fulfillment, Quex uses **subscriptions** - pre-deposited native tokens managed by the `IDepositManager` contract, which handle covering all necessary fees when fulfilling requests. `QuexRequestManager` already has a built-in `createSubscription()` method, which performs the necessary steps to create and set up a subscription. The only parameter you need to provide is the initial deposit amount. You can always replenish the subscription using the `deposit()` method from `IDepositManager` or withdraw remaining funds using the `withdraw()` method. For more details, you can refer to [Subscription management](../consumer/subscription_management.md) section of this documentation.


```solidity
...

contract TVLEmission is QuexRequestManager {
    constructor(address treasuryAddress, address quexCore, address oraclePool) QuexRequestManager(quexCore) {
        ...
        createSubscription();
    }
    
    ...
}
```

## Deploy and Run

That's it!

Now, all you need is to deploy the `TVLEmission` contract and call the built-in `request()` method to fetch your data and mint tokens. You can do this using any tools you prefer—particularly, you may use our prepared Foundry scripts available in the [examples repository](https://github.com/quex-tech/quex-v1-examples/tree/main/tvl-emission).


## Next Steps

Congratulations on deploying your very first contract that relies on data secured by confidential computing proofs! You can explore further by visiting other sections of our documentation to learn how to build more complex examples and create the next generation of DApps connected to real-world data.

Also, join our [Community](../community.md) to stay updated on Quex developments and engage with other developers!
