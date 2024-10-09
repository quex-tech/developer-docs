# Select Data

Let's start with an example where you need to get cryptocurrency prices on-chain. In this case, we'll use Binance’s public API to fetch the data.

The prices can be accessed off-chain by making a `GET` request to the following API route: `https://www.binance.com/api/v3/ticker/price`. The response will contain the latest prices of all traded pairs in the following format:

```json
[
  {"symbol": "ETHBTC", "price": "0.03914000"},
  {"symbol": "LTCBTC", "price": "0.00105700"},
  ...
]
```

To make this request certified by Quex, we have to send the request to the TEE-protected module using the following `curl` command:

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
      "filter": null,
      "schema": "string"
    }'
```

After submitting this request, the response will include a signed result in JSON format:

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
    "value": [
      {"symbol": "ETHBTC", "price": "0.03914000"},
      {"symbol": "LTCBTC", "price": "0.00105700"},
      ...
    ]
  }
}
```

In this signed response:

- The feedID serves as a unique identifier for the request and is calculated as a keccak256 hash of the entire request structure. This ensures that any change to the request, such as modifying a parameter or the path, will result in a different feedID, ensuring the integrity of the data.
- The cryptographic signature covers the entire response, including the feedID, the timestamp, and the data itself, providing strong guarantees that the data was securely fetched and signed by the TEE at the moment of the request.


Note that in this section, we omitted the information about the quex-api-endpoint and the method for providing the response on-chain. These details will be covered later in this documentation.