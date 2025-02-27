# Getting Started with Quex

## Project Overview

In this tutorial we are creating a DApp on Arbitrum Sepolia, which consumes the data from Binance public API. To see the
basic capabilities, we are also going to use the post-processing of the data on the Quex Data Oracle Side making a
simple script (or filter in `jq` terms) for this purpose, and illustrate the non-trivial return structure.

Suppose the DApp collects the order books for BTC/USDT pair for its logic. It needs the following data from Binance:
+ The sequential number of the update to keep track of the ordering (Binance returns it as `lastUpdateId`)
+ Five best bids 
+ Five best asks
Both bid and ask are required to be tuples of integer numbers (price, quantity). The precision is required to be 8th digit
after decimal point. That is, the prices are to be multiplied by 100,000,000 and returned as `uint256`.

## Design the Data Structures

In line with the problem statement, our contract will work with the following data structures:
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

## Deploy Receiving Contract

Here is an example of a simple contract keeping track of the last created request and storing the response data.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.22;

// Import IQuexActionRegistry interface and DataItem structure
import "https://github.com/quex-tech/quex-v1-interfaces/blob/fad2ceb5bff350b1eece52bdb74a2e01984f333e/interfaces/core/IQuexActionRegistry.sol";
import "@openzeppelin/contracts@4.5.0/access/Ownable.sol";

address constant QUEX_CORE = 0xD8a37e96117816D43949e72B90F73061A868b387;
IQuexActionRegistry constant quexCore = IQuexActionRegistry(QUEX_CORE);

struct Order {
    uint256 price;
    uint256 quantity;
}

struct OrderBook {
    uint256 lastUpdateId;
    Order[5] bids;
    Order[5] asks;
}

contract C is Ownable {
    uint256 requestId;
    OrderBook[] orderBooks;

    // We will track the requests performed by the unique request Id assigned by Quex
    // Only keep the latest request Id
    function request(uint256 flowId) public payable onlyOwner returns(uint256) {
        requestId = quexCore.createRequest{value:msg.value}(flowId);
        return requestId;
    }

    // On request() call this contract may receive the change from Quex Core. Therefore, receive() method must be
    // implemented
    receive() external payable {
        payable(owner()).call{value: msg.value}("");
    }

    // Callback handling the data processing logic
    function processResponse(uint256 receivedRequestId, DataItem memory response, IdType idType) external {
        // Verify that the sender is indeed Quex
        require(msg.sender == QUEX_CORE, "Only Quex Proxy can push data");
        // Veify that the request was initiated on-chain, rather than off-chain
        require(idType == IdType.RequestId, "Return type mismatch");
        // Verify that the response corresponds to our request
        require(receivedRequestId == requestId, "Unknown request ID");
        // Use the data. In this case we just store them
        orderBooks.push(abi.decode(response.value, (OrderBook)));
        return;
    }

    // Simple view function to see the results
    function getOrderBooks() external view returns (OrderBook[] memory) {
        return orderBooks;
    }

    // Another convenience view
    function getLastBid() external view returns (Order memory) {
        require(orderBooks.length >= 1, "No order books recorded");
        return orderBooks[orderBooks.length - 1].bids[0];
    }
}
```

## Register Action

According to the Quex architecture, the two things need to be done for data to be shipped. First, get the Action Id from
the oracle pool. In our case, the pool is the Quex Request Pool. The action must consist in performing HTTPS request to
Binance open API. Since this action is quite specific, the pool does not know it in advance. So we need to register this action
on the pool contract and get its id. If you are interested in the specifics of this process, consult the [Request Pool
Description](../https_pool/https_pool.md). In this tutorial we use the helper tool to create both action and flow
simultaneously, so let us go through the idea behind the flow creation first.

## Create Flow

After the action id is known, it is time to define the route of data delivery. That is, to tell Quex Core what oracle
pool is the data supplier, what action is expected to be performed by it for the particular demand, what is the address
of the data consumer (including callback selector), and what gas consumption to expect from the callback (for relayer
reimbursement). As a result, we will get flow id which can be used for making requests and pushing the data without
passing the specific details every time. For the structures involved, please consult [Flow Creation](flow_creation.md).

To save time on these contract interactions, we will use the [Flow Creation
Tool](https://github.com/quex-tech/quex-v1-interfaces/tree/master/tools/create_flow). So, let us pass to this part

## Using Flow Creation Script

### Configure Settings
First, edit `config.json`. Compare the addresses of oracle pool and Quex Core with the ones you can find
[here](../general/addresses). Verify that `rpc_url` indeed points to Arbitrum Sepolia RPC. We can see from Remix IDE
that gas limit of 700k should be more than enough for `processResponse` call. The value of `td_pubkey` can be found
either in our Core Contract (see [`ITrustDomainRegistry`](../provider/td_registration), `REPORT_DATA` field of the TD
Quote), or on [addresses](../general/addresses) page. In case your request will not have private data, the `td_pubkey`
does not matter.

Make sure that `consumer` points to your contract, and the callback selector points to your method. If you use Remix
IDE, you can find the selectors in `Solidity Compiler->Compilation Details->Function Hashes`.

### Configure Request

Now, we edit `request.json`. It defines the structure that will be passed to `addAction` call. The `request` field has
the general structure of HTTP request that will be performed by the oracle. However, there is also similarly looking
`patch` field. This field contains the private data which will be added to the request inside the TD. The TD accepts it
in encrypted form. Why is it in plaintext here then? The flow creation script encrypts it prior to sending using the
`td_pubkey` previously specified in the config. The `pathSuffix` is concatenated to the path in the `request`, the
headers and parameters are added to those of `request`, the body in `patch`. In case the body in `patch` is non-empty,
it will replace the `body` from the `request`.

The query we are tailoring accesses the host `www.binance.com`, path `/api/v3/depth`, has header `Content-Type:
application/json`, and parameters `limit=5` and `symbol=BTCUSDT`. If one tries this query, the response from Binance API
is like
```
{
  "lastUpdateId": 62019703469,
  "bids": [
    [
      "86319.98000000",
      "2.90207000"
    ],
  ...
  ],
  "asks": [
    [
      "86319.99000000",
      "0.58125000"
    ],
  ...
  ]
}
```
Now, we need to let the oracle know how to convert this JSON file to solidity structs used by our contract. To do so,
first define `responseSchema` as Solidity ABI schema for the `OrderBook` structure. Namely,
`(uint256,(uint256,uint256)[5],(uint256,uint256)[5])`.  Now we need instruct the oracle to post-process response in a
mixed-type array which can be cast to this type. Quex Request Oracle Pool uses a subset of [jq](https://jqlang.org)
language for JSON post-processing. Jq programs are also called filters. We start building the filter step by step.
1. To cast a number from string to desired format, `tonumber*100000000 | floor` can be used.
2. Now, the filter `.bids[0] | map(tonumber*100000000 | floor)` would yield the first bid converted to the
right format
3. To convert all the bids to the necessary format, apply this map as nested: `.bids | map(map(tonumber*100000000 | floor))`
4. To reuse this filter for asks, process bids and asks as an array:
`[.bids, .asks] | map(map(map(tonumber*100000000 | floor)))`. Now we have two arrays adhering to the encoding.
5. Finally, prepend it with the value of `lastUpdateId`:
`[.lastUpdateId] + ([.bids, .asks] | map(map(map(tonumber*100000000|floor))))`

Combining it all together, the `request.json` file may now look as follows:
```json
{
    "request": {
        "method": "GET",
        "host": "www.binance.com",
        "path": "/api/v3/depth",
        "headers": [
            {
                "key": "Content-Type",
                "value": "application/json"
            }
        ],
        "body": "",
        "parameters": [
            {
                "key": "limit",
                "value": "5"
            },
            {
                "key": "symbol",
                "value": "BTCUSDT"
            }
        ]
    },
    "jqFilter": "[.lastUpdateId]+([.bids,.asks]|map(map(map(tonumber*100000000|floor))))",
    "responseSchema": "(uint256,(uint256,uint256)[5],(uint256,uint256)[5])",
}
```

Note that we did not include any patch as we do not need the private data in this particular case.

### Run the Flow Creation Script

In order to initiate the transactions, the script needs access to the secret key. It is easiest to pass it as an
environment variable. However, you can also add it to `.env` or to `config.json` (see readme for the script)
```bash
SECRET_KEY=deadbeef... python create_flow.py config.json
```

The script will output the id of registered action; flow id, the fee per request in native coins, and the amount of gas
to be covered per request.

## Estimate Fee

In our case, the tool has already shown the fee values. However, if we needed to access them from other project, we
could use `getRequestFee(uint256 flowId)` method of the Quex Core which returns this tuple. The value which must be
attached to the transaction is `nativeFee + gasPrice*gas`. Suppose, the call returned `30000000000000` Wei as
`nativeFee` and `810000` as gas. Suppose also that gas price is 0.1 GWei That means, the request creating transaction
must have at least `110000` GWei in value. It is safe to round this value up, as Quex Core returns the change.

## Send Request

Once the value is estimated, the request can be created by calling `request` function on our contract with the value
taken from the first step. This transaction submits the on-chain request that is captured by the pool relayer,
transferred to the oracle in the pool. After the oracle completes the task, the post-processed data are relayed to Quex
Core contract. Quex Core verifies the authority of the signing Trust Domain for this particular action, checks the
signature, and sends the data to our callback.

## Check the Result

Check out your view functions to see that order books are indeed delivered.
