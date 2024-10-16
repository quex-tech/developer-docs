# Post-Processing

In the [Retrieve Data](../retrieve-data.md) section, we demonstrated how to retrieve a certified response from the Binance API, which returns data in the following format:

```json
[
  {"symbol": "ETHBTC", "price": "0.03914000"},
  {"symbol": "LTCBTC", "price": "0.00105700"},
  ...
]
```

However, using this raw data directly within smart contracts is not suitable due to its structure and redundancy. Thus, in this section, we explore how Quex allows you to perform verifiable computations for post-processing on fetched data, making it more suitable for on-chain use by extracting relevant information and performing computations.

## Examples

[Extracting Integer Values](extract-int.md)
> Learn how to extract a specific price and convert it to an integer value for efficient on-chain processing.

[Index Calculation](calculate-index)
> See how to calculate a weighted index price using multiple assets and make the result available on-chain.
