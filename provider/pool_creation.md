# Pool Creation

In case you are willing to create your own oracle pool utilizing Quex, the following interface must be implemented:
```solidity
interface IOraclePool {
    function isInPool(uint256 tdId) external view returns (bool);
    function getAction(uint256 actionId) external view returns (bytes memory);
    function getActionFee(uint256 actionId) external view returns (uint256);
    function getTreasury() external view returns (address);
}
```

Here we describe the intended functionality 

## isInPool(uint256 tdId)

The function must receive id of the Trust Domain registered with Quex Core and return `true` in case this particular
Trust Domain has the authority to response to the pool actions. You can arrange the set of policies which fits your pool
model best. Consider few examples

### TD Whitelist

In this scenario, the pool explicitly enumerates the Trust Domains, which can participate in it. This is the simplest
use case (and often the most adequate one) which corresponds to the fully private oracle pools

### Fixed Payload TDs

In this scenario, you may wish to provide the right of response to any Trust Domain, given that they run exactly the same
software. For this purpose, the `isInPool` function can query `getTD(uint256 tdId)` method of Quex Core receiving the TD Quote for the
given TD, and verifying whether the `MRTD` and `RTMR` registers match the prescribed values

### Controlling CPU version

In this scenario, in addition to verifying the measurement registers, you may wish to opt-in for the on-chain
verification of CPU version and Quoting Enclave, up to inspection of Provisioning Certification Keys. For example, you
can choose to allow pool participation for those Trust Domains which operate specific versions of TDX module and have
latest microcode updates applied. Apart from `getTD`, your contract may retrieve id of the quoting enclave via
`getQEId(uint256 tdId)`, and inspect the quote with `getQE(uint256 qeId)`.

## getAction(uint256 actionId)

This method must return the byte array that can be accepted by trust domains in your pool as an action. The Quex
oracles consume actions in a REST endpoint as POST request with body
```json
{
    "action": "base64-encoded-action-bytestring"
}
```

We recommend complying to this pattern for ease of interoperability and access universality. However, you may choose
another means of action processing. It is important, however, that the Trust Domain must be able to reproduce action Id
by the action byte array, since action Id is among the fields that must be signed by a TD as a response.

The semantics of the actions can vary. Ids may be sequential numbers or hash of some data structures. For example, in
our HTTPS oracle pool, the action Id is the hash of the full HTTP request structure together with the post-processing
prescription, and the response ABI scheme.

## getActionFee(uint256 actionId)

This method must return the fee your pool assigns to the given action Id. It is important in case your pool operates in
an on-chain request-based fashion. In this case, the initiator of the request will have to add your fee to the request
during creation. This fee will be delivered to your treasury address by Quex Core once the request is fulfilled.

## getTreasury()

This method must return the address of your treasury. That is, the address where the action fees are to be transferred on
request fulfillment
