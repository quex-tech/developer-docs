## Data Verification Mechanism

Quex employs a robust chain of trust to ensure the correctness of data fed to the blockchain. This process begins with the verification of the **attestation report**. The attestation report contains a chain of certificates that can be traced back to the root certificate of the hardware vendor, ensuring that the oracle program is operating within a secure, encrypted environment that cannot be modified by external parties.

Each enclave generates a fresh private key during initialization and includes the corresponding public key within the attestation report. This process is performed once for each oracle node. By verifying the attestation report, we confirm that the private key is securely contained within the specific enclave and can only be used to sign correct data points.

After the attestation report has been verified, every data point retrieved by the oracle is signed with the private key. On-chain, we verify the signature using the previously authenticated public key. This guarantees that our data feed can only accept data that has been correctly signed by the trusted enclave, preventing any tampering or manipulation of the data supplied to the DeFi protocols.
 