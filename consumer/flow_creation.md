# Flow creation

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

The flow can be created on Quex Core with `createFlow(Flow memory flow)` call which returns `flowId` for later
reference. This id can also be recorded from the `FlowAdded(uint256 flowId)` event emitted at flow creation.
