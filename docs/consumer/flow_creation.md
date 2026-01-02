# Data Flow

The data flow corresponds to the route the specific action takes from the pool to the data consumer contract. In Quex,
the general flow structure is the following

```solidity
struct Flow {
    uint256 gasLimit;
    uint256 actionId;
    address pool;
    address consumer;
    bytes4 callback;
}
```

Here the `gasLimit` is the gas limit for the `callback` function on the consumer contract. Once created, the flow will
be used every time your contract needs to consume the data corresponding to the `actionId` from the oracle `pool`. Note
that for request-based processing, the `gasLimit` contributes to amount which must be attached to the subsequent
requests. All the data is verified by Quex Core prior to the delivery, and the `callback` on `consumer` contract is
invoked with the designated `gasLimit`.

# Flow Creation

A flow can be created on Quex Core using the `createFlow(Flow memory flow)` call, which returns a `flowId` for future reference. This ID can also be captured from the `FlowAdded(uint256 flowId)` event emitted during flow creation.

In many cases, you might want to construct the flow directly within your contract to explicitly define each step. To simplify this, Quex provides the [`FlowBuilder`](https://github.com/quex-tech/quex-v1-interfaces/blob/master/src/libraries/FlowBuilder.sol) library, which requires defining only the necessary fields.

Here's an example from our tutorial demonstrating how to set up a data flow using `FlowBuilder`:

```solidity
    import "quex-v1-interfaces/src/libraries/QuexRequestManager.sol";
    using FlowBuilder for FlowBuilder.FlowConfig;
    ...

    /**
     * Creates a new flow to fetch TVL data from the DeFi Llama API for dydx, multiplies it by 1e18,
     * and rounds to the nearest integer.
     */
    function setUpFlow(address quexCore, address oraclePool) private onlyOwner {
        FlowBuilder.FlowConfig memory config = FlowBuilder.create(quexCore, oraclePool, "api.llama.fi", "/tvl/dydx");
        config = config.withFilter(". * 1000000000000000000 | round");
        config = config.withSchema("uint256");
        config = config.withCallback(address(this), this.processResponse.selector);
        registerFlow(config);
    }
}
```

However, in many cases, you may prefer building the flow off-chain and registering it on-chain afterward. For instance, this approach is necessary when only part of the data flow is public, and some request data (e.g., API credentials) must remain encrypted. For these scenarios, we've developed a [standalone Python tool](https://github.com/quex-tech/quex-v1-interfaces/tree/master/tools/create_flow) that allows you to define the entire Flow structure in plaintext, securely encrypt the private data, and register the flow on-chain. Once registered, the encrypted data is securely included in the request within the TD and will never again be available in plaintext.