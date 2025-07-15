# Overview

Quex implements the unified protocol of TEE-based data transfer to the blockchain. Currently, Quex supports EVM-compatible
chains, and its TEE backbone is based on Intel TDX technology providing secure, fast, reliable and scalable environment for
arbitrary logic execution and verification.

Intel TDX allows one to create hardware-isolated virtual machines (Trust Domains, TDs), which can be verifiably protected
from intervention both from host and VM creator. The authenticity of the TD output can be verified by leveraging remote
attestation protocols, resulting in the output data being signed with a key, with
+ Private key being bound to the particular Trust Domain on a particular CPU
+ Public key effectively certified with chain of trust rooting in Intel SGX Root of Trust CA
+ Protected from reading by any party including the TD creator, and the hardware owner

TD itself can perform any actions an ordinary VM can. For example, if the network interface is present, the TD can issue
the HTTP requests to external resources via TLS, verify the TLS certificates and post-process the data. Or, it can
contain the software running as a light node in a blockchain, capturing events from the chain. Or, it can perform some
heavy data manipulation, such as running LLM, which is prohibitively computation-intense for the on-chain execution (or
even on-chain verification of the corresponding ZK proofs).

In all these cases, the result of the TD execution for Quex is the output data structure, conveniently packed for the
on-chain usage and signed with a unique key known solely by TD itself. 

The public counter-part of the key is written on the blockchain once per TD together with all the necessary information
regarding the TD (including hashed measurements of the memory layout, firmware, kernel image, initramfs, certificate
chain, CPU information). This public key is later used for the on-chain signature verification of the data received from
the TD. We call these TDs **Oracles**.

Any oracle can perform a prescribed set of **actions**. For example, querying and external REST endpoint and
post-processing the result is an action. An oracle receives **command** for an action, executes it and returns the signed
result.

To provide flexibility and fault tolerance, the oracles are united in **Oracle Pools**, normally by the set of actions
they can perform. When the end user needs some data, they address the command to the Oracle Pool. Any oracle from the
pool can execute the command, and the signed result is shipped to the chain. Quex Core on-chain logic verifies the
signature, checks the permissions of the responding oracle, and delivers the result to the customer-defined receiving
smart contract.

For common scenarios the action ID, the oracle pool ID, and the receiver address are reused multiple times. To
simplify the usage and reduce the fees in the long run, Quex uses **data flows**. A data flow is a combination of
recipient contract address, recipient contract callback, callback gas limit, oracle pool address, and ID of the action to be
performed by the oracle. Once created, the data flow is stored on-chain, and the subsequent requests to the oracle pools
are done solely by flow ID without any extra data.

Recipient contract callback may charge an unpredictable amount of gas fees, which must be paid by the relayer. To simplify the money flow during request creation and fulfillment, Quex introduces **subscriptions** - pre-deposited native tokens in a special contract used to cover all necessary fees when fulfilling requests.
