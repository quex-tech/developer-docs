## Example 1: Extracting ETH Price and Converting to an Integer

In this example, we will extract the `ETHBTC` price from the raw JSON response and multiply it by 10^8, rounding it
to an integer value. This approach makes the data more manageable and suitable for on-chain use.

To extract the `ETHBTC` price and process it into an integer, modify the request to the Quex TEE module as follows:

   ```bash
   curl -X POST https://quex-api-endpoint/query \
   -H "Content-Type: application/json" \
   -d '{
         "request": {
           "method": "Get",
           "host": "www.binance.com",
           "path": "/api/v3/ticker/price",
           "headers": [],
           "parameters": []
         },
         "patch": null,
         "filter": ".[] | select(.symbol == \"ETHBTC\") | (.price | tonumber * 100000000 | floor)",
         "schema": "int256"
       }'
   ```

- The JQ filter specified in the `"filter"`
  parameter: `.[] | select(.symbol == "ETHBTC") | (.price | tonumber * 100000000 | floor)` performs the following steps:
    - Selects the `ETHBTC` price from the JSON array.
    - Converts the price to a number.
    - Multiplies it by 10^8 and rounds down to an integer.
- The `"schema": "int256"` indicates that the processed result will be returned as an integer value.

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
    "value": 3914000
  }
}
```

The `value` field now contains the extracted `ETHBTC` price, multiplied by 10^8 to make it suitable for on-chain use. Note that only the final number is available on-chain, omitting all the intermediate steps such as making the request to Binance and processing the result. However, all these steps are encoded within the `feedID` and are secured with the signature, which is generated within the TEE module, preventing any possibility of manipulating the result.
