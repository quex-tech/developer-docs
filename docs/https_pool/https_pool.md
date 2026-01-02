# Quex request oracle pool descriptive guide

## Pool actions

The actions of this pool represent the combination of the public part of the
request, its private encrypted part, post-processing filter and ABI schema for the post-processed result:

```solidity
struct RequestAction {
    HTTPRequest request;
    HTTPPrivatePatch patch;
    string responseSchema;
    string jqFilter;
}
```

The actions are normally added with `addAction(RequestAction memory requestAction)` call. This method returns the id of the
action recorded. However, if you have multiple actions with reusable constituents, you may choose adding it by parts via
`addActionByParts(bytes32 requestId, bytes32 patchId, bytes32 schemaId, bytes32 filterId)`. In order to register the
parts of the action and obtain their ids, use the methods
+ `addRequest(HTTPRequest memory request)`
+ `addPrivatePatch(HTTPPrivatePatch memory privatePatch)`
+ `addJqFilter(string memory jqFilter)`
+ `addResponseSchema(string memory responseSchema)`

## HTTPRequest

Below is the list of the structures needed to define an `HTTPRequest`
```solidity
enum RequestMethod {
    Get,
    Post,
    Put,
    Patch,
    Delete,
    Options,
    Trace
}

struct RequestHeader {
    string key;
    string value;
}

struct QueryParameter {
    string key;
    string value;
}

struct HTTPRequest {
    RequestMethod method;
    string host;
    string path;
    RequestHeader[] headers;
    QueryParameter[] parameters;
    bytes body;
}
```
These structures bear common HTTP semantics

## HTTPPrivatePatch

`HTTPPrivatePatch` is the structure used to hold the data encrypted per TD, such as API keys or security tokens.

```solidity
struct RequestHeaderPatch {
    string key;
    bytes ciphertext;
}

struct QueryParameterPatch {
    string key;
    bytes ciphertext;
}

struct HTTPPrivatePatch {
    bytes pathSuffix;
    RequestHeaderPatch[] headers;
    QueryParameterPatch[] parameters;
    bytes body;
    address tdAddress;
}
```
The byte field are encrypted for the TD using DHKE in combination with AES-GCM. The format of the ciphertexts is
```
ephemeral_key | nonce | tag | ciphertext
```

Here `ephemeral_key` is x-coordinate of the point followed by the y-coordinate of the point.

The symmetric key for the ciphertext is generated as 32-byte SHA256-HKDF of concatenation of ephemeral key and shared EC point.
Both elliptic curve points are encoded and uncompressed (prefixed with `0x04` byte).

## jqFilter

`jqFilter` is the string representing the filter in the language `jq` to be applied to the response prior to data return. Quex Request
Oracles support a subset of `jq`, which is described on the corresponding [page](jq_subset.md)

## responseSchema

`responseSchema` field contains a plaintext description of the ABI schema for the returned data. The data post-processed
with `jq` must represent the mixed-type array. This array is interpreted by the oracle as tuple to be encoded with
`responseSchema`

## Action IDs

Action Ids for Request Oracle Pool are computed as `keccak256` of the ABI-encoded `RequestAction`
