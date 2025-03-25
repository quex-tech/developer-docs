# Contract addresses

## Mainnets

| Network         | Quex Core                                    | Request Oracle Pool                          |
| --------------- | -------------------------------------------- | -------------------------------------------- |
| Arbitrum One    | `0xD8a37e96117816D43949e72B90F73061A868b387` | `0x957E16D5bfa78799d79b86bBb84b3Ca34D986439` |
| Avalanche       | `0xD8a37e96117816D43949e72B90F73061A868b387` | `0x957E16D5bfa78799d79b86bBb84b3Ca34D986439` |
| Berachain       | `0xD8a37e96117816D43949e72B90F73061A868b387` | `0x957E16D5bfa78799d79b86bBb84b3Ca34D986439` |
| BNB Smart Chain | `0xD8a37e96117816D43949e72B90F73061A868b387` | `0x957E16D5bfa78799d79b86bBb84b3Ca34D986439` |
| Celo            | `0x48f15775Bc2d83BA18485FE19D4BC6a7ad90293c` | `0x3606B159e59040172a01D691b5abC3C0c9349A2f` |
| Hedera*         | `0x29b25EC9760e7a4CDbb7539Ae671C585E4028618` | `0x76E91bD9F9d13693a76609b51991E81B796e72D0` |
| Redbelly        | `0x946E499e8b6A7a745FC1c3d8EeC9e3d69C6Ec117` | `0x971911B8634d96e362482a8E8c08BcaA26feB17e` |
| XDC             | `0xD8a37e96117816D43949e72B90F73061A868b387` | `0x957E16D5bfa78799d79b86bBb84b3Ca34D986439` |

TD pubkey for Request Oracle Pool: `0x610e7707b0dbbed2bc6f1c66e9674955bae124e7a406ee64133d77c8bf0595c3ce6dcd75b4e289b14caf5ad4a15868a8fad54388ae091dd165e00bd5260206af`

## Testnets

| Network                 | Quex Core                                    | Request Oracle Pool                          |
| ----------------------- | -------------------------------------------- | -------------------------------------------- |
| Arbitrum Sepolia        | `0xD8a37e96117816D43949e72B90F73061A868b387` | `0x957E16D5bfa78799d79b86bBb84b3Ca34D986439` |
| Avalanche Fuji          | `0xD8a37e96117816D43949e72B90F73061A868b387` | `0x957E16D5bfa78799d79b86bBb84b3Ca34D986439` |
| Berachain Bepolia       | `0x29b25EC9760e7a4CDbb7539Ae671C585E4028618` | `0x57DBA934A01A584300d68668729F7c83161b854C` |
| BNB Smart Chain Testnet | `0xD8a37e96117816D43949e72B90F73061A868b387` | `0x957E16D5bfa78799d79b86bBb84b3Ca34D986439` |
| Celo Alfajores          | `0xD8a37e96117816D43949e72B90F73061A868b387` | `0x957E16D5bfa78799d79b86bBb84b3Ca34D986439` |
| Hedera Testnet*           | `0x4d14714Ec77d7300011873fD4c828cbbc6c4d9EB` | `0x42701a7ADC7eF01d222A236C7d6498878109698f` |
| Redbelly Testnet        | `0x213fcba4caCa2d1675Ae4859f2966455d9e5bE94` | `0x7B666588EE93232887a447f7D212Dff7A4854933` |
| XDC Apothem             | `0xD8a37e96117816D43949e72B90F73061A868b387` | `0x957E16D5bfa78799d79b86bBb84b3Ca34D986439` |

TD pubkey** for Request Oracle Pool: `0xb23974e9267308bd821c34038e00072bf1e297f308227d98de387deb50f9ca2ebed328af1471f291e53eff602130f5ab79d006ee040553016775d79261362770`

---
\* In Hedera network the [IQuexActionRegistry.getRequestFee](https://github.com/quex-tech/quex-v1-interfaces/blob/master/src/interfaces/core/IQuexActionRegistry.sol) function returns `nativeFee` value in tinybar (tℏ). In some cases you need to convert it from 8 decimal places value to 18 decimal places value. More info [here](https://docs.hedera.com/hedera/sdks-and-apis/sdks/hbars#hbar-decimal-places)      
\** TD-specific public keys for testnet request oracle pools are may be changed without notice
