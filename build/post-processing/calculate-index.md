## Example 2: Calculating an Index Price Based on ETH and LTC

In this example, we will calculate the price of an index composed of 60% `ETHBTC` and 40% `LTCBTC`. This involves fetching the prices for both pairs, performing weighted averaging, and converting the result into an integer value for on-chain use.

1. **Modify the Request**

   To calculate the weighted average of the `ETHBTC` and `LTCBTC` prices, modify the request to the Quex TEE module as follows:

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
         "filter": "((.[] | select(.symbol == \"ETHBTC\") | .price | tonumber) * 0.6 + (.[] | select(.symbol == \"LTCBTC\") | .price | tonumber) * 0.4) * 100000000 | floor",
         "schema": "int256"
       }'
   ```

- The JQ filter specified in the `"filter"` parameter: `((.[] | select(.symbol == "ETHBTC") | .price | tonumber) * 0.6 + (.[] | select(.symbol == "LTCBTC") | .price | tonumber) * 0.4) * 100000000 | floor` performs the following steps:
    - Selects the prices for both `ETHBTC` and `LTCBTC` from the JSON array.
    - Converts each price to a number.
    - Calculates the weighted average: 60% of `ETHBTC` and 40% of `LTCBTC`.
    - Multiplies the result by 10<sup>8</sup> and rounds it down to an integer.
- The `"schema": "int256"` indicates that the processed result will be returned as an integer value.

2. **Certified Response Structure**

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
       "value": 2389480
     }
   }
   ```

   The `value` field now contains the calculated index price, with the weighted average of `ETHBTC` and `LTCBTC` prices multiplied by 10<sup>8</sup> for convenient on-chain use. Note that only the final number is available on-chain, omitting all intermediate steps such as fetching the prices and performing the calculations. However, all these steps are encoded within the `feedID` and secured with the signature generated within the TEE module, ensuring the result's integrity and preventing manipulation.
