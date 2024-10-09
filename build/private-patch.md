# Accessing Credential-Protected Data

In previous sections, we provided examples of accessing public API routes. However, in most cases, you may need to access credentials-protected data. For example, if we try to get cryptocurrency prices from CoinMarketCap instead of Binance, we must include an API key in the request.

## Example: Fetching Data from CoinMarketCap

To get the latest price of Bitcoin (BTC) from CoinMarketCap, we will use the following request:
- **URL**: `https://pro-api.coinmarketcap.com/v2/cryptocurrency/quotes/latest`
- **Parameters**: `{ "id": "1" }` (where `"1"` is the ID for Bitcoin)
- **Headers**: `X-CMC_PRO_API_KEY: "abc123-..."` (where `"abc123-..."` is your private API key)

The JQ filter we'll use to extract the price and format it for on-chain use is:
```jq
(.data["1"].quote.USD.price * 1000000) | round
```

To securely make this request, we need to hide the private API key from public view. We will achieve this by creating a patch to the HTTPS request that includes the encrypted API key.

The public key required for encryption is presented within the enclave's attestation report, which is signed with a certified chain. Refer to the relevant section of the documentation for details on how to obtain and verify the attestation report.

## Creating a Secure Patch

Quex allows you to include sensitive data, such as API keys, securely in your request by using an encrypted patch. The patch ensures that the private information is not exposed and can only be decrypted within the TEE-protected environment. Here’s how you can modify the request:

```bash
curl -X POST https://quex-api-endpoint/query \
-H "Content-Type: application/json" \
-d '{
      "request": {
        "method": "Get",
        "host": "pro-api.coinmarketcap.com",
        "path": "/v2/cryptocurrency/quotes/latest",
        "headers": [],
        "parameters": [
          {
            "key": "id",
            "value": "1"
          }
        ]
      },
      "patch": {
        "headers": [
          {
            "key": "X-CMC_PRO_API_KEY",
            "ciphertext": "encrypted_api_key_here"
          }
        ],
        "parameters": null,
        "path_suffix": null,
        "body": null
      },
      "filter": "(.data[\"1\"].quote.USD.price * 1000000) | round",
      "schema": "int256"
    }'
```

In this request:
- The `"ciphertext"` for the `X-CMC_PRO_API_KEY` is an encrypted version of your API key, ensuring that the private key is never exposed in plaintext.
- The patch provides the encrypted API key, which will be decrypted and used only within the TEE-protected enclave during the request.


This approach is generic and can be used to access any credential-protected data while keeping private information hidden from public view. The encrypted data is represented under the `feedID` hash, ensuring that any modifications to the encrypted content will result in a different `feedID`, thus preserving data integrity and confidentiality.