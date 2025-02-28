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

Having the flow registered, the on-chain data request is created by calling `createRequest(uint256 flowId)` on Quex
Core. Note that this function is payable, and the value of native coins must be attached to it. This value consists of
the following parts:
+ Native coin fees (by Quex Core and by Oracle Pool)
+ Gas consumed by Quex Core and callback

Both of these parts can be accounted for by querying view `getRequestFee(uint256 flowId)` in Quex Core. This method
returns the tuple `(uint256 nativeFee, uint256 gasFee)`. The value to be attached to the request creating transaction
can be computed as `nativeFee + tx.gasPrice*gasFee`. In case larger value is transferred, the change will be returned
from Quex Core to the sender.

Once the request is created, it is transferred to an oracle. The signed oracle response is delivered by the relayer to
`fulfillRequest(OracleMessage memory message, ETHSignature memory signature, uint256 requestId, uint256 tdId)`. At this
point, the oracle pool policies are checked, signature is verified, and the data is dispatched to the consumer callback.

## Off-chain initiated data transfer

This case is way simpler, as the only method invoked on-chain is `pushData(OracleMessage memory message, ETHSignature
memory signature, uint256 flowId, uint256 tdId)`. Note that this method is again payable, and the attached value can be
obtained via `getQuexFee(uint256 flowId)` method. The difference is that in this case the relaying party pays the Quex
Fee, whereas in previous case the request initiating party paid, and the relayer was reimbursed for the gas spent.
