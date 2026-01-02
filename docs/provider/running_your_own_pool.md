# Running Your Own Pool

This sections contains the steps necessary to run you own Trust Domain pool on Quex.

## Determine the set of actions to be performed by the oracle pool

An Oracle Pool is an abstraction used by Quex. The pool unites the set of Trust Domains capable of performing the same
(or similar) set of actions. The actions can include external data retrieval, complex processing, random number
generation and so on. Every action must have a unique Id associated to it and an encoding as a byte array.

## Determine the set of policies for pool participation

The pool can contain fixed number of oracles owned by you, or can contain unlimited set of oracles united by the same
codebase involving other parties, or it can also have some hardware requirements verified on-chain via Quex Core. At
this point you need to decide what will be the rules for your pool participation. Please check the [Pool
Creation](pool_creation.md) for more details

## Deploy your oracle code inside TDX

As Quex currently relies on Intel TDX, your oracles must run inside trusted execution environment. At the moment, Quex
does not offer options of hosting, so you need ether your own, or rented hardware for the purpose. It is important that
your Trust Domains must have their own private keys for signing the output data. These keys must never leave the TDs, as
well as there must be no efficient means for any party (including your devs, admins and yourself) to extract these keys.
At the moment, it is your responsibility to instantiate VMs in a convincing and verifiable way. Quex can assist with the
correct tools and approach for this purpose, but it cannot take the responsibility for what is happening inside your own
code. See [TD Oracle Requirements](td_oracle_requirements.md) for the details.

## Deploy your pool contract

The contract will be responsible for action enumeration, TDs in your pool, and your fee management. That is, you will
define which policies are applied to your Trust Domains, which fees are assigned by you for on-chain initiated requests,
and where will Quex transfer these fees. In case your pool operates in a push-based model, which is absolutely in line
with Quex infrastructure, it is assumed that you collect your revenue with another means, and the fee you assign in the
pool contract is less relevant. Please, consult [Pool Creation](pool_creation.md) for description and examples.

## Register your TDs with Quex

In order or Quex to be capable of your TD data verification, and delivery to the end customer, TDs from your pool must
be registered on chain with Quex Core. See [TD Registration](td_registration.md).

## Start shipping the data

At this point, all the necessary preliminaries are met, and all you need is to start shipping the data. In case of
request-based model, make sure that you have relayers passing the actions from the chain to your TDs. In case of
push-based model, make sure that a daemon software is set up to transfer the signed data from your TDs to Quex Core via
`pushData` method.
