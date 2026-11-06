# Decentralized Applications

SignA gives a web page a way to ask for Algorand accounts and signatures
without ever handing it a private key. The dApp builds a transaction, the
wallet shows the user what it does, and only a signature comes back.

For users, that means account secrets stay encrypted in the extension. Every
connection is scoped to one origin and one network, and every signature is
approved individually.

## Three ways to connect

The extension injects all of these into every page. They share one signing
implementation, so which you use is a matter of what your tooling expects.

- **ARC-0027 / AVM web provider.** The discovery-and-messaging standard that
  modern dApp libraries integrate through. If you use
  [use-wallet](https://github.com/TxnLab/use-wallet) v5 or anything else built
  on `@agoralabs-sh/avm-web-provider`, this is what it will find, with no
  wallet-specific code on your side.
- **`window.algorand`.** The ARC-0001 style object: `enable()`, `signTxns()`,
  `postTxns()`, `signAndPostTxns()`. See the
  [dApp integration guide](dApp-integration.md).
- **`window.AlgoSigner`.** The original object, kept for dApps written against
  AlgoSigner before 1.10.0. See the
  [legacy integration guide](legacy-dApp-integration.md). It still works, but
  new integrations should not start here.

## Networks

A connection is bound to a network. `enable()` can request one by genesis ID
or genesis hash; without either, the user picks. MainNet, TestNet, and BetaNet
are built in, and users can add their own, so do not assume the set of
networks is fixed. See [Working with Custom Networks](dApp-integration.md#custom-networks).
