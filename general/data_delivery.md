# Data delivery modes

Quex Core supports two mechanisms of data delivery:
+ On-chain initiated (request-based)
+ Off-chain initiated (request-based, event-based, push)

Both of these mechanisms utilize Flows as routing abstraction. To differentiate between the two on the data consumer
side, `IdType` is passed to the consumer callback:

```solidity
enum IdType {
    RequestId,
    FlowId
}
```

`IdType` of `RequestId` means that the data was delivered as a result of the on-chain request, and the `id` passed to
the callback is the request id. `IdType` of `FlowId` means that the data was delivered as a result of the off-chain
initiation, and delivered solely by flow id (which is passed to the callback as an id in this case).

## On-Chain initiated data transfer

Having the flow and subscription registered, an on-chain data request can be created by calling `createRequest(uint256 flowId, uint256 subscriptionId)` on Quex Core. At this point, the amount of tokens required to cover all potential fees (Quex, oracle pool, callback gas consumption) is reserved within the subscription until the request is fulfilled or canceled.

Once the request is created, it is transferred to an oracle. The signed oracle response is delivered by the relayer to `fulfillRequest(OracleMessage memory message, ETHSignature memory signature, uint256 requestId, uint256 tdId)`. At this stage, oracle pool policies are checked, the signature is verified, and the data is dispatched to the consumer callback. Since the actual gas consumed by the client's callback is known at this point, the reserved tokens are released from the subscription, and the actual fees are distributed among Quex Core, the oracle pool, and the relayer.

## Off-chain initiated data transfer

This case is way simpler, as the only method invoked on-chain is `pushData(OracleMessage memory message, ETHSignature
memory signature, uint256 flowId, uint256 tdId)`. Note that this method is payable, and the attached value can be
obtained via `getQuexFee(uint256 flowId)` method. The difference is that in this case the relaying party pays only the Quex
Fee, whereas in previous case the relayer was reimbursed for the gas spent from the subscription.
