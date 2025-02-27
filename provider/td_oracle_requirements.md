# TD oracle requirements

It is recommended that your TD accepts the actions as byte arrays (maybe encoded). And it is mandatory that your TD
outputs the data signed with the key which public counterpart is registered in the corresponding TD Quote.

The output from your oracle must contain `OracleMessage` and signature over this ABI-encoded structure. Since it affects
the delivery routine and signature verification, the `OracleMessage` is specified as a Solidity structure:
```solidity
struct OracleMessage {
    uint256 actionId;
    DataItem dataItem;
}
```

Here `actionId` is the action Id assigned to the action by your pool. It must be reproducible from within your TD, and
`DataItem` is specified as

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
