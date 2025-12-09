# Getting Started with Quex

# Introduction

Quex provides a protocol and infrastructure for decentralized data transfer based on confidential computing.  The most simple of these for developers is data oracle, and that's what we'll be learning about today.
In this guide, we'll explore two examples that demonstrate different aspects of using Quex: a basic parametric emission token that uses public API data, and an advanced integration with OpenAI API that requires secure authentication.

## Document Structure

This tutorial is organized into the following sections:

- **Introduction**: Provides an overview of Quex and explains what we're building.
- **Example 1: TVL-Based Token Emission**: A complete example demonstrating how to build a token that mints based on DeFi protocol TVL data.
    - **Try it Out**: Quick start instructions to deploy and run the provided example.
    - **Let's Create a Quex-based Contract**: Step-by-step instructions to set up your environment and implement smart contracts leveraging Quex.
        - **Prepare Environment**: How to configure your development environment.
        - **Token Emission**: Detailed implementation of ERC-20 token emission logic.
        - **Create Flow**: Defines how data requests are structured and handled by Quex oracle pools.
        - **Deploy and Run**: Guide to deploying the contracts and executing the request to mint tokens.
- **Example 2: OpenAI API Integration**: An advanced example showing how to integrate with APIs that require authentication using encrypted private headers.
    - **Try it Out**: Quick start instructions for the OpenAI integration example.
    - **Let's Build an AI Integration Contract**: Step-by-step guide to integrating OpenAI API with Quex.
        - **Prepare Environment**: Setting up dependencies and API credentials.
        - **OpenAI Integration Contract**: Implementation of a contract that fetches AI-generated sentiment scores.
        - **Create Flow with Private Headers**: How to securely pass API keys using Trust Domain encryption.
        - **Deploy and Run**: Guide to deploying and testing the OpenAI integration.
- **Next Steps**: Recommendations on further exploration and advanced use cases.

# Example 1: TVL-Based Token Emission

## What Are We Building?
Let's begin by clearly defining what we're building and why. Imagine you're issuing a token for a DeFi protocol and want to design tokenomics that incentivize the foundation to actively increase the protocol's Total Value Locked (TVL). One effective approach is awarding the foundation tokens proportional to the protocol's TVL, as reported by [DeFiLlama](https://defillama.com).

Specifically, we'll design a token where the foundation receives freshly minted tokens daily, equal in number to the USD-denominated TVL of the protocol across all supported chains and protocol versions. For clarity and practical demonstration, we'll use TVL data from the [DyDx protocol](https://defillama.com/protocol/dydx). Fortunately, DeFiLlama provides an [API endpoint](https://api.llama.fi/tvl/dydx) that returns exactly this number. However, because this data is provided off-chain, achieving our goal requires an oracle to securely transfer the data on-chain.


## Try it Out

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

## Let's Create a Quex-based Contract for TVL Emission!

### Prepare Environment

We'll build our example using [Foundry](https://book.getfoundry.sh) (Forge)—ensure it's correctly installed and up to date. Update Foundry by running:

```shell
foundryup
```

We’ll also use the ERC-20 implementation from OpenZeppelin and Quex libraries to simplify our development:

```shell
forge install openzeppelin/openzeppelin-contracts
forge install quex-tech/quex-v1-interfaces
```

### Token emission

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

### Create Flow

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

### Set up subscription

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

### Deploy and Run

That's it!

Now, all you need is to deploy the `TVLEmission` contract and call the built-in `request()` method to fetch your data and mint tokens. You can do this using any tools you prefer—particularly, you may use our prepared Foundry scripts available in the [examples repository](https://github.com/quex-tech/quex-v1-examples/tree/main/tvl-emission).

# Example 2: OpenAI API Integration

Now that you've built a contract using public API data, let's explore a more advanced use case: integrating with APIs that require authentication. In this example, we'll build a contract that uses OpenAI's API to perform sentiment analysis and store the results on-chain.

## What Are We Building?

Imagine you want to create a DeFi protocol that adjusts its parameters based on market sentiment. To achieve this, you need to fetch sentiment analysis from an AI service like OpenAI and use that data in your smart contracts. However, OpenAI's API requires authentication via an API key, which must be kept secure and never exposed on-chain.

Quex solves this challenge by allowing you to encrypt sensitive data (like API keys) using Trust Domain encryption. The encrypted credentials are securely passed to the oracles, which can decrypt and use them within the confidential computing environment, ensuring your API keys never appear in plaintext on-chain.

In this example, we'll build a contract that:
- Sends POST requests to OpenAI's chat completions API
- Uses encrypted private headers to securely pass the API key
- Extracts sentiment scores (0-100) from OpenAI responses
- Stores the latest sentiment score on-chain
- Emits events when new responses are received

## Try it Out

The codebase for this example is available in [our examples repository](https://github.com/quex-tech/quex-v1-examples). You can explore it by cloning the repository and installing its dependencies:

```shell
git clone https://github.com/quex-tech/quex-v1-examples
cd quex-v1-examples/openai-api-integration
forge install
```

To deploy these contracts, you'll need:
1. A wallet with gas tokens on your chosen network (e.g., Arbitrum Sepolia)
2. An OpenAI API key
3. A Trust Domain (TD) address configured with your OpenAI API key

Set your private key as an environment variable:

```shell
export SECRET=<0xYourPrivateKey>
```

**Important**: Before deploying, you need to:
1. Register your OpenAI API key with a Trust Domain (TD) to get the encrypted API key
2. Update the `TRUST_DOMAIN` constant in `script/DeployOpenAIScript.s.sol` with actual TD address (see [here](../general/addresses.md))
3. Set the encrypted API key as an environment variable (as hex string):

```shell
export ENCRYPTED_API_KEY=<0xEncryptedApiKeyHex>
```

The encrypted API key should be in the format `"Bearer sk-..."` encrypted by the Trust Domain and provided as a hex string (starting with `0x`).
For details on the encryption format, refer to [this section of the tooling code](https://github.com/quex-tech/quex-v1-interfaces/blob/9d454bfe1e0c632d1b15caadf3a76c5bd0f12f2f/tools/create_flow/create_flow.py#L45).

### Build and Deploy Contracts

Run the `DeployOpenAIScript` to build and deploy the OpenAI integration contract and create a data flow:

```shell
forge script script/DeployOpenAIScript.s.sol --broadcast
```

You'll need the deployed contract address for making requests. Store the deployed contract's address as an environment variable:

```shell
export CONTRACT_ADDRESS=<DEPLOYED_CONTRACT_ADDRESS>
```

### Make a Request

After deployment, make a request using the provided script:

```shell
forge script script/Request.s.sol --broadcast
```

This will send a request to OpenAI API through Quex oracles. The contract will receive the response and store the sentiment score.

### Check the Response

You can check the stored sentiment score by calling the contract's view functions using Foundry's cast tool:

```shell
cast call $CONTRACT_ADDRESS "latestSentimentScore()(uint256)" --rpc-url <YOUR_RPC_URL>
cast call $CONTRACT_ADDRESS "getLatestSentiment()(uint256,uint256)" --rpc-url <YOUR_RPC_URL>
```

Congratulations! You've successfully integrated OpenAI API with Quex, demonstrating how to securely use authenticated APIs in your smart contracts.

## Let's Build an AI Integration Contract!

### Prepare Environment

We'll build our example using [Foundry](https://book.getfoundry.sh) (Forge)—ensure it's correctly installed and up to date. Update Foundry by running:

```shell
foundryup
```

We'll use the Quex libraries to simplify our development:

```shell
forge install quex-tech/quex-v1-interfaces
```

### OpenAI Integration Contract

Let's create a new contract `OpenAIIntegration.sol` in the src folder:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.22;

import "quex-v1-interfaces/src/libraries/QuexRequestManager.sol";

using FlowBuilder for FlowBuilder.FlowConfig;

/**
 * @title OpenAIIntegration
 * @dev This contract demonstrates how to integrate OpenAI API with Quex data oracles.
 * It fetches sentiment analysis scores from OpenAI and stores them on-chain.
 */
contract OpenAIIntegration is QuexRequestManager {
    /// @notice Stores the latest sentiment score received from OpenAI
    uint256 public latestSentimentScore;
    
    /// @notice Stores the timestamp of the last successful response
    uint256 public lastResponseTime;
    
    /// @notice Event emitted when a new sentiment score is received
    event SentimentScoreReceived(uint256 requestId, uint256 score, uint256 timestamp);

    constructor(address quexCore, address oraclePool, address tdAddress, bytes memory encryptedApiKey) payable QuexRequestManager(quexCore) {
        setUp(quexCore, oraclePool, tdAddress, encryptedApiKey);
    }

    /**
     * @notice Processes a response from Quex containing the sentiment score from OpenAI.
     * @param receivedRequestId The ID of the request that is being processed.
     * @param response The response data from Quex, expected to contain the sentiment score (0-100).
     */
    function processResponse(uint256 receivedRequestId, DataItem memory response, IdType idType) 
        external 
        verifyResponse(receivedRequestId, idType) 
    {
        uint256 sentimentScore = abi.decode(response.value, (uint256));
        require(sentimentScore <= 100, "Invalid sentiment score");
        
        latestSentimentScore = sentimentScore;
        lastResponseTime = block.timestamp;
        
        emit SentimentScoreReceived(receivedRequestId, sentimentScore, block.timestamp);
    }

    /**
     * @notice Retrieves the latest sentiment score and timestamp
     * @return score The latest sentiment score (0-100)
     * @return timestamp The timestamp when the score was received
     */
    function getLatestSentiment() external view returns (uint256 score, uint256 timestamp) {
        return (latestSentimentScore, lastResponseTime);
    }
}
```

This contract:
- Inherits from `QuexRequestManager` for simplified interactions with Quex oracle pools
- Stores the latest sentiment score and timestamp
- Implements `processResponse` as a callback to securely process responses from the Quex oracle
- Validates that sentiment scores are between 0-100
- Emits events when new responses are received

### Create Flow with Private Headers

Now, let's add the `setUp()` function that configures the flow with encrypted private headers. This is the key difference from our first example—we'll use Trust Domain encryption to securely pass the API key:

```solidity
/**
 * @notice Creates a new flow to fetch sentiment analysis from OpenAI API.
 * The flow sends a POST request to OpenAI's chat completions endpoint with a prompt
 * asking for a sentiment score (0-100), and extracts the numeric response.
 * @param quexCore Address of the Quex Core contract
 * @param oraclePool Address of the Oracle Pool contract
 * @param tdAddress Address of the Trust Domain that encrypted the API key
 * @param encryptedApiKey The encrypted API key (format: "Bearer sk-...") as bytes
 */
function setUp(address quexCore, address oraclePool, address tdAddress, bytes memory encryptedApiKey) private onlyOwner {
    require(msg.value > 0, "Please attach some Eth to deposit subscription");
    require(encryptedApiKey.length > 0, "Encrypted API key cannot be empty");

    // Set up public headers
    RequestHeader[] memory headers = new RequestHeader[](1);
    headers[0] = RequestHeader({key: "Content-Type", value: "application/json"});

    // Set up private headers for API key (encrypted)
    RequestHeaderPatch[] memory privateHeaders = new RequestHeaderPatch[](1);
    privateHeaders[0] = RequestHeaderPatch({
        key: "Authorization",
        ciphertext: encryptedApiKey
    });

    // Set up request body with prompt for sentiment analysis
    bytes memory body = bytes('{"model":"gpt-5-nano-2025-08-07","messages":[{"role":"user","content":"Rate the sentiment of the following text on a scale of 0-100, where 0 is very negative and 100 is very positive. Return only the number: The cryptocurrency market is showing strong bullish signals today."}]}');

    // Set up flow
    FlowBuilder.FlowConfig memory config = FlowBuilder.create(quexCore, oraclePool, "api.openai.com", "/v1/chat/completions");
    config = config.withMethod(RequestMethod.Post);
    config = config.withHeaders(headers);
    config = config.withBody(body);
    config = config.withTdAddress(tdAddress);
    config = config.withPrivateHeaders(privateHeaders);
    // Extract the numeric sentiment score from the response
    // OpenAI returns: {"choices":[{"message":{"content":"75"}}]}
    // We extract just the number
    config = config.withFilter(".choices[0].message.content | tonumber | round");
    config = config.withSchema("uint256");
    config = config.withCallback(address(this), this.processResponse.selector);
    registerFlow(config);

    // Set up subscription that will be used to charge fees
    createSubscription(msg.value);
}
```

Let's break down the key differences from our first example:

1. **POST Method**: Unlike the TVL example which uses GET, we specify `RequestMethod.Post` since OpenAI's chat completions API requires POST requests.

2. **Request Body**: We include a JSON body with the prompt asking OpenAI to analyze sentiment and return a numeric score.

3. **Private Headers**: This is the crucial security feature. We use `withPrivateHeaders()` to pass the encrypted API key. The `RequestHeaderPatch` structure contains:
   - `key`: The header name ("Authorization")
   - `ciphertext`: The encrypted API key (format: "Bearer sk-..." encrypted by the Trust Domain)

4. **Trust Domain Address**: We specify the Trust Domain address using `withTdAddress()`, which tells Quex which Trust Domain should decrypt the private headers.

5. **jq Filter**: The filter `.choices[0].message.content | tonumber | round` extracts the numeric response from OpenAI's JSON structure and converts it to an integer.

The encrypted API key is never exposed on-chain—it's only decrypted within the confidential computing environment of the Quex oracles, ensuring your credentials remain secure.

### Deploy and Run

That's it! Now you can deploy the `OpenAIIntegration` contract and call the built-in `request()` method to fetch sentiment analysis from OpenAI. You can use our prepared Foundry scripts available in the [examples repository](https://github.com/quex-tech/quex-v1-examples/tree/main/openai-api-integration).

## Next Steps

Congratulations on deploying your very first contract that relies on data secured by confidential computing proofs! You can explore further by visiting other sections of our documentation to learn how to build more complex examples and create the next generation of DApps connected to real-world data.

Also, join our [Community](../community.md) to stay updated on Quex developments and engage with other developers!
