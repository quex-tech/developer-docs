# Quex Data Oracle Tutorial

Here we present a step-by-step illustration of Quex data oracle usage. This tutorial passes through all the actions needed
to create your own data feed on Quex Data Oracle query and use the data.

For the sake of example, we consider the part of dApp that collects and stores five best bids and five best asks on
BTC-USDT pair on Binance together with their `lastUpdateId`. Normally you would not need the persistent storage of this
data due to high gas consumption and corresponding transaction fees. You would rather make some aggregation and
implement some decision logic with as few persistent storage as possible. However, this example is good in demonstrating
most of Quex capabilities. Namely
+ Complex HTTP query to a third-party API
+ Non-trivial result post-processing, including arithmetics, array mapping, and selecting fields of JSON
+ Ability to receive the data on-chain as a ready-to-use user-defined structure without the hassle of reencoding,
  repacking, or introducing large argument lists

### Register the request on-chain

Clone the interfaces repository with the helper script to register the new request
```bash
git clone https://github.com/quex-tech/quex-v1-interfaces
```

Cd to the directory with the helper script
```bash
cd quex-v1-interfaces/tools/create_feed
```

Edit the data feed description (`quex_feed.json`) to meet your desired request. In our case, it will look like
```json
{
  "request": {
    "method": "GET",
    "host": "www.binance.com",
    "path": "/api/v3/depth",
    "headers": [],
    "parameters": [{"key": "symbol", "value": "BTCUSDT"}, {"key": "limit", "value": "5"}],
    "body": null
  },
  "patch": null,
  "schema": "(uint256,(uint256,uint256)[5],(uint256,uint256)[5])",
  "filter": "[.lastUpdateId]+([.bids,.asks]|map(map(map(tonumber*100000000|floor))))"
}
```
The details on how these fields are constructed can be found [here](../post-processing/structs.md). Briefly, `request`
has the structure of an HTTP request to be made by an oracle, `patch` is designated to contain the encrypted private
fields of the request, `schema` is the type of the structure which is to be ABI-encoded and passed on-chain, `filter` is
a `jq` filter, used to convert response JSON to `schema`. Quex uses the subset of jq language for
post-processing. The list of supported operations can be found [here](./jq-subset.md).

In `config.json` provide your RPC URL, the address of Quex `FeedRegistry` contract, and the path to the file with your
data feed description (`quex_feed.json`)
```json
{
  "rpc_url": "https://governors.testnet.redbelly.network",
  "feed_registry": "0xE35fBA13d96a9A26166cbD2aC8Df12CD842e1408",
  "feed_file": "quex_feed.json"
}

```
The address of the `FeedRegistry` contract together with other user-facing contracts can be found
[here](./contract-addresses.md).

Prepare Python virtual environment and install dependencies
```bash
python -m venv venv
source ./venv/bin/activate
pip install -r requirements.txt
```

Run the script to register the request. It will send corresponding transactions to the configured chain. Hence, it
requires the private key of the account you use for on-chain request registration
```bash
SECRET_KEY=<your private key here> python create_feed.py
```
The output will look like
```
request_id:    0xd7623cc955950e5ddcad849873a251ab645d71fe6ff146401908cbbc4548c015
patch_id:      0x0000000000000000000000000000000000000000000000000000000000000000
schema_id:     0x50b18bc35b93e3457523fc4f65c329ae0667fb1fece8f0ac00b7e794943fa214
filter_id:     0x96cef62829ee2b193bfef8d779ca070706428adf9e0866dae067fb653fc9e2bc
feed_id:       0x6279d8a0a0ae2db6a6ad495ec2b4c7dfc4b482cd3495035b4053e8dd8b359a6c
```
The most important field here is feed ID. Take a note of it

You can see the corresponding transactions in the explorer. The parts of the data feed description are reusable. So, in case you will
be registering new data feeds manually, you can reuse `request_id`, `patch_id`, `schema_id`, and `filter_id` for new
data feeds without sending already recorded data on-chain.

### Create a Smart-Contract

Create your contract which utilizes the data from the oracle. Here is an example for our particular case. Note that
`FEED_ID` is taken as in the output of `create_feed.py`.
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.22;

// Import IV1RequestRegistry interface and DataItem structure
import "https://github.com/quex-tech/quex-v1-interfaces/blob/master/interfaces/IV1RequestRegistry.sol";

address constant QUEX_REQUEST_REGISTRY_ADDRESS = 0x9afa1fd478705a1C4663F1590674500293f01e24;
address constant QUEX_REQUEST_LOGIC_ADDRESS = 0xD81F8f8B7007e9A82cE42bcE96D0572b36bEa77a;
bytes32 constant FEED_ID = 0x6279d8a0a0ae2db6a6ad495ec2b4c7dfc4b482cd3495035b4053e8dd8b359a6c;

struct Order {
    uint256 price;
    uint256 quantity;
}

struct OrderBook {
    uint256 lastUpdateId;
    Order[5] bids;
    Order[5] asks;
}

contract C {
    IV1RequestRegistry quexRequests;
    bytes32 requestId;
    OrderBook[] orderBooks;

    constructor () {
        // The requests will be made to the QuexRequestRegistry contract
        quexRequests = IV1RequestRegistry(QUEX_REQUEST_REGISTRY_ADDRESS);
    }

    // We have two options: either pass the exact compensation for gas to QuexRequestRegistry
    // (callbackGasLimit*tx.gasprice)
    // or be able to receive the change from QuexRequestRegistry. Here we choose the second option.
    // Therefore, receive() must be implemented
    receive() external payable {}

    // The method creates the request to be processed by Quex oracle
    function request(uint32 callbackGasLimit) public payable returns(bytes32) {
        // send the request and store generated request ID.
        // We must pay the compensation for the future callback invocation
        (requestId, ) = quexRequests.sendRequest{value: msg.value}(
            FEED_ID,                       // ID of priorly created feed
            address(this),                 // Address of the contract receiving the response
            this.processResponse.selector, // selector of the callback receiving the response
            callbackGasLimit               // gas limit for the callback
        );

        // Return the change to sender
        payable(msg.sender).transfer(address(this).balance);
        return requestId;
    }

    // The callback which is receiving and processing the response
    // The callback must accept (bytes32,DataItem memory) as its arguments
    function processResponse(bytes32 receivedRequestId, DataItem memory response) external {
        // verify that the data comes from genuine source
        require(msg.sender == QUEX_REQUEST_LOGIC_ADDRESS, "Only Quex Logic can push data");

        // verify that the response is for the request created by ourselves
        require(receivedRequestId == requestId, "Unknown request ID");

        // since we have recorded and verified the request ID, verifying the feed ID is not necessary in this particular case
        // `value` field of DataItem contains the ABI-encoded structure we requested

        // decode the data and process it (push to the array, for example)
        orderBooks.push(abi.decode(response.value, (OrderBook)));
    }

    // Two getter methods to view stored data
    function getOrderBooks() external view returns (OrderBook[] memory) {
        return orderBooks;
    }

    function getLastBid() external view returns (Order memory) {
        require(orderBooks.length >= 1, "No order books recorded");
        return orderBooks[orderBooks.length - 1].bids[0];
    }
}
```
Deploy your contract to the network where Quex data oracles exist. This particular contract can be compiled with
`"viaIR": true` option to allow structure array copying.

### Query Data

Every time we wish to query the data, we can use the `request` method of the contract above. It will pass the request to
`QuexRequestRegistry` contract, and after the data is fetched and post-processed it will be returned to
`processResponse` method. As `processResponse` can contain arbitrary logic, the `callbackGasLimit` must be passed to
`request` which must account for the gas used by `processResponse`. Additionally, the amount of native coins not less
than `tx.gasprice*callbackGasLimit` must be paid in the `request` call to cover these gas expenses.

Do not worry about guessing the precise value of the gas price. The `QuexRequestRegistry` contract calculates the exact
price and returns the change within the same transaction, so you can safely transfer a bit more than needed.

In our example the callback has to store large structures of `OrderBook`, and a gas consumption is a bit less than 600k
per call. So when trying out this contract in the test network, you can set `callbackGasLimit` to 600000, and set the
transaction value to cover this gas (at the time of writing, 0.5 RBNT on Redbelly testnet will suffice for all this
logic).

After you trigger `request`, and receive the response, you can check it with the getters provided.
