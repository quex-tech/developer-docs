# Registering your Trust Domain

When running your own pool with custom actions, you will need Trust Domains performing these actions. In order for your
TDs to be attested and registered on-chain, they must be submitted to Quex Core contract. Once registered, they are
associated to their public key counterparts, and the data verification from the TD boils down to policies check (via
`isInPool` call to your oracle pool) and signature verification. The full registration flow is the following:

+ Register certificate of your platform CA key in case it is not registered yet `addPlatformCAKey`
+ Register certificate of your PCK signed with platform CA `addPCK`
+ Register the report from your Quoting Enclave compliant to Intel TDX DCAP signed with PCK `addQE`
+ Register your TD Quote which is associated to the report from QE `addTD`. At this point your TD receives a unique id
  in Quex

At each step, Quex Core contract verifies the data you submit, and stores it on-chain possibility of reuse in pool
policies. This routine usually costly due to high storage consumption, but is normally done once per TD.

**Important** The data contained in `REPORT_DATA` field of your TD Quote will be interpreted as the public key of your
Trust Domain. Hence, every time the data is passed with the given TD Id, the signature will be verified against this
key. It is your responsibility, as the pool creator, to arrange the key management inside the TD, and the pool policies
in a way suitable for your data consumers. Ideally, the keys must never leave the trust domain.
