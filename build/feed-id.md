### Feed Id Calculation

In this section, we describe the method used to calculate the `feedID` within Quex, ensuring it remains deterministic and secure.

In a previous section, we introduced a method to patch requests with encrypted data, allowing sensitive information like API keys to be hidden. While this is useful for security, it introduces a new problem: the `feedID` will vary between different nodes accessing the same data sources because they may use different API keys. Even for a single node, rotating API keys could result in different `feedID`s, which would cause issues for the on-chain clients of Quex.

To solve this, we introduce the `feed_id_filter` parameter—a JQ program that is applied to the entire input request JSON before calculating the `feedID`. For example, a filter like `"feed_id_filter": "del(.patch.headers[]?.ciphertext)"` removes the `ciphertext` fields from the headers in the `patch` section. As a result, the filtered JSON will be identical to the original request, except that the sensitive `ciphertext` fields have been excluded. The `feed_id_filter` itself is included in this JSON, ensuring transparency about how the request was processed before calculating the `feedID`.

Once the `feed_id_filter` has been applied, the resulting JSON is sorted by keys and serialized into a compact single-line string. This ensures a consistent representation of the request data.

Finally, the Keccak cryptographic hash function is applied to the byte representation of this string. This guarantees that every unique request generates a unique `feedID`, while filtering out irrelevant data like API keys to maintain consistency across nodes.

#### Example

Consider the following input structure:

```json
{
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
        ]
    },
    "filter": "(.data[\"1\"].quote.USD.price * 1000000) | round",
    "feed_id_filter": "del(.patch.headers[]?.ciphertext)",
    "schema": "int256"
}
```

In this example, the `feed_id_filter` removes the `ciphertext` field from the `patch.headers`. The resulting JSON, minus the sensitive data, is then sorted by keys and serialized into a compact string. This string is then hashed using the Keccak function to produce the final `feedID`.

This method ensures that the `feedID` remains consistent and secure, even when API keys or other sensitive data are updated.
