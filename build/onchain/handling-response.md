# Handling the Response

The response is automatically verified on-chain by validating the cryptographic signature. If a callback function is specified, it will be executed to perform the dApp logic associated with the data update.

Once the data is published on-chain, it is accessible to any contract through the `IV1QuexLogReader` interface. The following example demonstrates how to retrieve the latest BTC price from the Quex log:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.22;

import "https://github.com/quex-tech/quex-v1-interfaces/blob/master/interfaces/IV1QuexLogReader.sol";

bytes32 constant BTC_FEED_ID = 0xabs123...;
address constant QUEX_LOG_ADDRESS = 0x3959148FF37f2d5c5F7a4A9c2E12dA4057B9C38A;

contract C {
    IV1QuexLogReader qreader;

    constructor () {
        qreader = IV1QuexLogReader(QUEX_LOG_ADDRESS);
    }

    function getLastBTCPriceInCents() public view returns (int256) {
        (uint256 id, int256 value, uint256 timestamp) = qreader.getLastData(BTC_FEED_ID);
        return (value / 10000) + (value % 10000 >= 5000 ? int256(1) : int256(0));
    }
}
```

This implementation enables the use of Quex logs for regularly updated price feeds, functioning similarly to other oracle networks such as Chainlink or Pyth.
