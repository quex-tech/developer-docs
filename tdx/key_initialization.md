# Key initialization

In Quex we provide a toolchain for building reproducible virtual machines which are supposed to be run as Quex oracles.

Every VM (trust domain) can posess its own secret key material. In order to provide secure persistent storage we utilize
SGX functionality on the same physical machine.

[quex-vault](https://github.com/quex-tech/quex-v1-vault) is an SGX application whose main goal is secure generation of
the master key material, and its secure storage. We use the SGX ESEAL functionality for the purpose. The key material
(seed) is generated inside the enclave, never leaves it in plaintext form and can only be output in the encrypted form
for persistent storage.

This seed is used for the per-VM secret derivation. To obtain the key material for a specific TD, `quex-vault` requires
the following input:
+ previously encrypted seed 
+ quote from the TD running on the same machine, which must contain an ephemeral EC public key as a user data.
+ key mask, which is a bitmap, telling which fields of the TD quote must be taken into account when deriving the key

Using the key mask, one can bind the key for specific measurements of the TD, or measurements together with TCB version
numbers, or MROWNER or any combination of fields. With this functionality one has the fine control over the key policies
(namely which machines are allowed to share the same key material)

Upon receiving the data, `quex-vault` performs local attestation of the TD, and issues the key material for the TD in a
ciphertext encrypted for the ephemeral key. The commitment to the ciphertext together with key mask and received quote
is included in SGX quote. The key material is derived from the masked quote and the seed.

The resulting SGX quote together with all the associated data and ciphertext is forwarded to TD. Which, in turn,
verifies the quote, and, in case of success, enrolls the received key material.
