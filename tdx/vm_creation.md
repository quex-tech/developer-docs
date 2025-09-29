# VM creation and start-up

## Creating VM

Using the Quex toolchain, one can easily create and attest Trust Domains running the payload inside containers.

To build a unified kernel image with a minimal kernel, single network adapter, and key management service (`quex-vault`)
integration, please use [quex-pack](https://github.com/quex-tech/quex-pack). For example, to create a VM running
`hello-world` container, first pull the container to your local machine
```sh
docker pull hello-world
```
Then create reproducible build of the UKI
```sh
./quex-pack.sh -o hello_world.efi docker-daemon:hello-world
```
Here `hello_world.uki` is the name of the target image. For more details on command line arguments and their meaning,
please refer to [quex-pack](https://github.com/quex-tech/quex-pack) readme.

This utility creates a minimal kernel image inside isolated environment. The image only contains minimum necessary to
run docker and a network adapter. For example, it does not have console devices, or ssh. The `init` process is
customized to mount necessary filesystems, enable networking, start key enrollment routine, and run docker afterwards.

After the image is created, you can obtain the measurements of the TD which will run this image. You can do so either by
using our [online tool](https://verify.quex.tech), or with our cli tool
[tdx-measurement-verify-cli](https://github.com/quex-tech/tdx-measurement-verify-cli). First, determine the firmware you
intend to use for your TD (in our case, `OVMF.fd`), and VM configuration, which is significant for ACPI tables
measurements. Then verification is as easy as running
```sh
./verify.js --cpu-cores 2 --memory 2048 --disks 0 hello_world.efi OVMF.fd
```
This command will print the expected measurements of the TD running with given firmware and UKI, 2 virtual CPUs, 2048
MiB RAM and no disk storage (running directly from initramfs).
Since it is a `js` script, you may need to install Node.js and run `npm i` in the `tdx-measurement-verify-cli`
directory to install dependencies.

## Generating the seed

**WARNING. Be cautious! You only need to generate encrypted seed once. quex-vault generates it at random, so make sure
you do not overwrite seed.dat unintentionally**.
In case you have **not** generated the seed for your [vault](https://github.com/quex-tech/quex-v1-vault) yet, run
```sh
vault -s > seed.dat
```

## Instantiating the secret key for your TD

The methods of TD instantiation may vary. In case you are on Ubuntu, please consult the [official
repository](https://github.com/canonical/tdx) for the detailed steps to set up TDX support and run TDs. Once your TD is
running, its key enrollment service opens TCP port `24516`. Upon the connection, it sends key request to the connected
party in binary. In case you are curious, the structure being sent is `td_key_request_t` described
[here](https://github.com/quex-tech/quex-v1-vault/blob/master/include/common.h). Save this struct as `key_req.dat`.
Currently, vault app is operating with files with default names. It reads `seed.dat` and `key_req.dat`, and writes
`quote.dat` together with `key_msg.dat`. So when you run
```sh
vault -s
```
you receive `quote.dat` and `key_msg.dat` binaries. Wrap them into
[quoted_td_key_response_t](https://github.com/quex-tech/quex-pack/blob/main/src/init/types.h) and send back to the
socket of the TD (within the same session, of course). Now the key is instantiated.

The secret key is now waiting for your docker code in `TD_SECRET_KEY` environment variable as hex. And the TDX quote is
in `/var/data/quote.txt`. Its `REPORTDATA` field contains the public key for this private key. So you can verify that the
measurements coincide with the ones you received before (if you serve this file from your container, of course).

## Conclusions

Now you have TD which you built yourself in a robust and reproducible way, initiated an SGX-based key vault,
instantiated a private key on the machine not known to any party but that machine, and received the TDX quote
conatining verifiable measurements of your TD together with the public key. The quote is fully verifiable, including the
CPU details, Intel signatures, and your measurements. Now your TD can sign data with this key, and you can be sure that
(if code and hardware have no vulnerabilities) everything signed with this key originates from your VM indeed, and
nobody can interfere with the execution flow. 

Note that if you switch the machine off or reboot it, its `init` will wait
for the encrypted key from the vault again. That's because secure key persistence relies on SGX sealing. Make sure to use the same seed file.
