## How it Works

### 1. Why Another Oracle?

Data oracles have become a prime target for attackers in the DeFi ecosystem, with significant financial losses stemming from oracle manipulation attacks. A report by Chainalysis shows that one in every three attacks on DeFi protocols in 2023 was linked to oracles, leading to annual losses of no less than $400 million. [Source](https://www.chainalysis.com/blog/oracle-manipulation-attacks-rising/).

A key reason oracles are vulnerable is the economic tradeoff between attack cost and potential profit. The cost of launching an attack often depends on purchasing enough oracle tokens to influence the outcome or bribing node operators. This makes current oracle designs reliant on fluctuating market conditions, leaving them susceptible to unpredictable attacks. Moreover, as the Total Value Locked (TVL) of DeFi protocols rises, the financial incentive for manipulation increases.

The need for better oracle solutions becomes particularly pressing with the rise of Real World Assets (RWA). The tokenization of real-world assets like real estate, securities, and commodities requires accurate and tamper-resistant data to be fed on-chain. This expansion of the Web3 market demands new mechanisms that can reliably connect traditional Web2 data sources with blockchain technology.

### 2. How Does Quex's Oracle Work?

Quex oracles leverage Trusted Execution Environments (TEE) to retrieve off-chain data with cryptographic guarantees. TEEs (we currently rely on Intel's chips with TDX support), create secure enclaves where data is processed and signed within a hardware-protected environment. These secure enclaves ensure that the private keys used for data signing never leave the chip, preventing any tampering with the data retrieval process.

When a Quex oracle is initialized, it generates a private key within the TEE. The key is used to sign any data retrieved from external sources, ensuring its authenticity. By using TLS protocols within the same enclave, the system guarantees that the retrieved data remains unmodified, as the entire process—from fetching to signing—occurs within the isolated, secure environment of the TEE. This makes Quex's oracle as secure as the underlying chip it operates on, providing a hardware-level defense against manipulation.

### 3. Data Verification Mechanism

Quex employs a robust chain of trust to ensure the correctness of data fed to the blockchain. This process begins with the verification of the **attestation report**. The attestation report contains a chain of certificates that can be traced back to the root certificate of the hardware vendor, ensuring that the oracle program is operating within a secure, encrypted environment that cannot be modified by external parties.

Each enclave generates a fresh private key during initialization and includes the corresponding public key within the attestation report. This process is performed once for each oracle node. By verifying the attestation report, we confirm that the private key is securely contained within the specific enclave and can only be used to sign correct data points.

After the attestation report has been verified, every data point retrieved by the oracle is signed with the private key. On-chain, we verify the signature using the previously authenticated public key. This guarantees that our data feed can only accept data that has been correctly signed by the trusted enclave, preventing any tampering or manipulation of the data supplied to the DeFi protocols.
 