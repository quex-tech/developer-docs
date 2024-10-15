## Example 3: Retrieveing Complex Data

In this example we illustrate how to pass complex structures for on-chain usage with non-trivial schema.
Suppose the on-chain handler requires five bids and asks from Binance for the symbol `BTCUSDT` together with the value
of `lastUpdateId` for extra on-chain checks. Suppose the corresponding Solidity structures are
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
Binance returns prices and quantities as string-encoded fixed-point numbers. So assume also that multiplication by 10<sup>8</sup>
is required prior to typecasting to `uint256` to prevent precision loss.

In this case, the schema of the desired result is `(uint256,(uint256,uint256)[5],(uint256,uint256)[5])`. The structures
are ABI-encoded as tuples. The JSON resulting from the filter, therefore, must be a mixed-type array of length 3 (outer
tuple type, three fields in `OrderBook` structure) with first element being a number, second and third elements being
arrays of length 5 (`(uint256,uint256)[5]`). Every element of the second and the third entry is, in turn, two element
array (pair of unsigned integers corresponding to the structure `Order`).

The example of Binance API response is
```json
{
  "lastUpdateId": 52933493429,
  "bids": [
    [
      "65615.89000000",
      "3.36758000"
    ],
    ...
  ],
  "asks": [
    [
      "65615.90000000",
      "3.52297000"
    ],
    ...
  ]
}
```
We start building the filter step by step. 
1. To cast a number from string to desired format, `tonumber*100000000 | floor` can be used. 
2. Now, the filter `.bids[0] | map(tonumber*100000000 | floor)` would yield the first bid converted to the
right format
3. To convert all the bids to the necessary format, apply this map as nested: `.bids | map(map(tonumber*100000000 | floor))`
4. To reuse this filter for asks, process bids and asks as an array:
`[.bids, .asks] | map(map(map(tonumber*100000000 | floor)))`. Now we have two arrays adhering to the encoding.
5. Finally, prepend it with the value of `lastUpdateId`:
`[.lastUpdateId] + ([.bids, .asks] | map(map(map(tonumber*100000000|floor))))`

The request to the oracle is therefore
```bash
curl -X POST https://quex-api-endpoint/query \
-H "Content-Type: application/json" \
-d '{
      "request": {
        "method": "Get",
        "host": "www.binance.com",
        "path": "/api/v3/depth",
        "headers": [],
        "parameters": [
          {
            "key": "symbol",
            "value": "BTCUSDT"
          },
          {
            "key": "limit",
            "value": 5
          }
        ]
      },
      "patch": null,
      "filter": "[.lastUpdateId] + ([.bids, .asks] | map(map(map(tonumber*100000000|floor))))"
      "schema": "(uint256,(uint256,uint256)[5],(uint256,uint256)[5])"
    }'
```
The certified response will be returned in the following format:
```json
{
  "signature": {
    "r": "0x...",
    "s": "0x...",
    "v": 27
  },
  "data": {
    "timestamp": 1696523461,
    "feedID": "0xabc123...",
    "value": "0x..."
  }
}
```
Here the `value` is ABI-encoded resulting structure.
The on-chain handler can now use Solidity ABI to decode the data and seamlessly use the struct:
```solidity
//... Inside the handler; binaryData is the content of the oracle response "value" field
OrderBook memory orderBook = abi.decode(binaryData, (OrderBook));
//... use orderBook object
```
