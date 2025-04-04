# Data Encoding Format

For EVM networks, Quex Request Oracles support all data structures compatible with [ABI encoding](https://docs.soliditylang.org/en/develop/abi-spec.html). A Solidity ABI schema is specified within Quex Request Oracles as a string, and it must match the data structure returned by the jq filter and used in your smart contracts.

Below is a list of typical or interesting schemas that might assist you when creating your flow:

**Integer: `uint256`**

The simplest case is fetching a single numeric value from an API. In such cases, use `uint256` as your schema. When using the `FlowBuilder` tool, set your schema during flow creation as follows:

```solidity
config = config.withSchema("uint256");
```

Later, decode the response while processing the oracle’s data:

```solidity
uint256 responseValue = abi.decode(response.value, (uint256));
```

**Nested structures**

Let's consider a more complex scenario. Suppose the DApp collects the order books for BTC/USDT pair for its logic. It needs the following data from Binance:
+ The sequential number of the update to keep track of the ordering (Binance returns it as `lastUpdateId`)
+ Five best bids
+ Five best asks

Both bids and asks should be represented as tuples of integers (price, quantity) with precision up to 8 digits after the decimal point.

To handle this scenario, we can define the following Solidity structures:

```solidity
struct Order {
    uint256 price;
    uint256 quantity;
}

struct OrderBook {
    uint256 lastUpdateId;
    Order[5] bids;
    Order[5] asks;
}
```

Then, define your ABI schema as `(uint256,(uint256,uint256)[5],(uint256,uint256)[5])`. When using the `FlowBuilder` tool, set the schema during flow creation:

```solidity
config = config.withSchema("(uint256,(uint256,uint256)[5],(uint256,uint256)[5])");
```

Finally, decoding the response during processing is straightforward:

```solidity
orderbook = abi.decode(response.value, (OrderBook))
```

As illustrated by these examples, Quex Request Oracles support a wide range of data schemas - from simple integers to complex nested structures. By leveraging ABI encoding, you can flexibly and securely integrate off-chain data into your smart contracts, enabling sophisticated, data-driven logic within your dApps.

