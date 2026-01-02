# TD oracle requirements

It is recommended that your TD accepts the actions as byte arrays (maybe encoded). And it is mandatory that your TD
outputs the data signed with the key which public counterpart is registered in the corresponding TD Quote.

The output from your oracle must contain `OracleMessage` and signature over this ABI-encoded structure. Since it affects
the delivery routine and signature verification, the `OracleMessage` is specified as a Solidity structure:
```solidity
struct OracleMessage {
    uint256 actionId;
    DataItem dataItem;
    address relayer;
}
```

Here `actionId` is the action Id assigned to the action by your pool. Note, that it is the responsibility of you as the
TD provider to make sure that the `actionId` under signature corresponds to the actual action being performed. The
action Id must correspond to the one defined in your [oracle pool](pool_creation.md).
`relayer` is the address of the party that will deliver signed data on-chain and is fixed here to prevent MEV attacks.
`DataItem` struct member is specified as

```solidity
struct DataItem {
    uint256 timestamp;
    uint256 error;
    bytes value;
}
```

`timestamp` is the timestamp of the data retrieval. `error` is the error code (zero if no error occured), `value` is the
actual data. We recommend `value` to be ABI-encoded struct convenient for the further use by your data consumers.

The signature over the `OracleMessage` is the struct representing usual Ethereum signature

```solidity
struct ETHSignature {
    bytes32 r;
    bytes32 s;
    uint8 v;
}
```
