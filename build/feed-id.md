### Feed Id Calculation

In this section, we describe the method used to calculate the `feedID` within Quex, ensuring it remains deterministic and secure. This process involves applying a `feed_id_filter` to the input JSON before generating a hash. This method allows sensitive information, such as API keys, to be excluded from the hash, ensuring the `feedID` remains consistent even when such data changes, like during API key rotations.

#### Applying the Feed Id Filter

The first step in computing the `feedID` is to apply the `feed_id_filter`, which removes any fields that should not contribute to the `feedID`. For instance, when dealing with credential-protected data, fields such as API keys or other sensitive information are excluded from the final hash. By using a `feed_id_filter`, only the stable and relevant parts of the input data are retained for the next steps in the process. 

This filtering ensures that the `feedID` remains unchanged even if encrypted fields like API keys are rotated, allowing the feed to stay consistent while keeping sensitive information secure.

#### Sorting Keys for Deterministic Feed Id

After the `feed_id_filter` has been applied, the remaining data structure is sorted by its keys to ensure determinism. This guarantees that the same input, irrespective of the order of its fields, will always produce the same `feedID`. Sorting the input structure ensures that any variations in the field order do not affect the hash calculation, providing a consistent result across different executions.

#### Example of Input Data

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

In this example, the input includes request parameters to fetch cryptocurrency data from CoinMarketCap. The `patch.headers` field contains an encrypted API key that should not affect the `feedID` calculation. To handle this, the `feed_id_filter` is applied to remove the `ciphertext` from the headers.

#### Creating Bytes for Hash Calculation

After applying the `feed_id_filter` and sorting the remaining fields by their keys, the input JSON is serialized into a byte string. This serialized representation is then used to compute the hash. By sorting the keys and filtering out sensitive data, the system ensures that the same input, regardless of the order or the presence of dynamic elements like encrypted API keys, consistently produces the same byte sequence. This byte sequence is then hashed using a cryptographic function (Keccak), producing the final `feedID` used in the Quex system.

This method ensures that the `feedID` remains stable and secure, even when sensitive data is rotated or updated, while maintaining determinism in the calculation process.