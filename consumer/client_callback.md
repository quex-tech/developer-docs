# Data Consumer Callback

The callback for data consuming contract must have the following type signature (the method name can be arbitrary)
```solidity
function processResponse(uint256 receivedRequestId, DataItem memory response, IdType idType) external;
```

Here `receivedRequestId` is either id of the previously created request (in case of on-chain initiation), or id of the
data flow (in case of off-chain initiation). These two cases can be distinguished by looking at the `idType`. `response`
contains the data from the oracle. As a data consumer, upon the data receiving, one must ensure that
+ The method was called by the Quex Core contract
+ `idType` has the expected value
+ `receivedRequestId` has the expected value

The optional checks include
+ Verify that `error` code in `response` is zero (depending on the particular oracle pool capabilities)
+ Verify that the `timestamp` in the `response` is within tolerance bound (depending on the oracle pool policies, data
  transfer mode, and business logic)
