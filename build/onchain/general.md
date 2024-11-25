# General Workflow

The general workflow with Quex oracle consists of the following steps:

1. Specify the actions to be performed by the oracle and register them on-chain. Retrieve the corresponding `feedID` as
   a result. This step is data feed creation.
2. Deploy the smart-contract capable of accepting the data output from the oracle
3. Every time new data is needed, request it from Quex smart-contract providing `feedID`, callback of the receiving
   contract, gas limit for the callback and the fee to cover this gas limit.
4. After the data is fetched, it is sent back to the callback with specified gas limit in a separate transaction

## Datafeed Creation

1. Define the request to TLS-secured endpoint of the data provider. It can by any third-party service given that TLS
   version is no lower than 1.2, and the response is provided as JSON file.
2. Define the private fields of the request to data provider endpoint. For example, API keys, access tokens etc. These
   will appear on-chain in encrypted form. The decryption will occur inside Trust Domain only.
3. Define the post-processing routine for the received JSON. The post-processing is a script defined as
   [jq](https://jqlang.github.io/jq/manual/) syntax. Quex oracle supports the subset of jq. See Supported Filters for
   the details. The script must yield the mixed-type JSON array in order to be packed into Solidity structure.
4. Annotate the types of fields in the array returned from the previous step.

## Deploying Client Contract

The client contract is your own contract which will handle the data. It must implement the method for accepting the data
from `V1RequestLogic` contract. Note that it is the responsibility of smart-contract developer to
implement the control that only `V1RequestLogic` has the right of this method invocation. The returned data contains
ID of the particular request (needed to distinguish between requests made concurrently) and `DataItem` object. Note also
that one must never rely on `requestId` solely. Always check `feedId` as well, as `feedId` is the field is under the
signature of
Trust Domain, whereas `requestId` is not.

## Data Querying

Data querying consists of calling `sendRequest(feedId, callbackAddress, callbackMethod, gasLimit)` on
`V1RequestRegistry` contract. Once this method is called, the Quex relayer collects the details of the request and
post-processing using `feedId` and queries the signing service residing in Trust Domain for the data. Signing service
forms HTTPS request, applies post-processing, ABI-encodes the resulting object to `DataItem.value` field, signs the
response, and sends it back to relayer. Relayer passes the data to Quex contracts, which verify the validity of signature
and permissions of the particular TD, and finally pass the data to the callback provided in `sendRequest`.
