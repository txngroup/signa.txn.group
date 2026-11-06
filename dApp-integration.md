# Integrating SignA

SignA injects a JavaScript library into every page the browser visits, so a
site can connect to the wallet, ask which accounts it holds, request a
signature for a transaction the site built, and post signed transactions to
the network. The wallet never gives out a key; only signatures cross the
boundary.

This guide covers `window.algorand`, the ARC-0001 style object. Two other
routes into the same signing implementation exist:

- **ARC-0027 / AVM web provider** (provider id
  `5e79c4b3-a2a4-4a2a-8bcb-09d6a4c6e0d1`). If you use a dApp library that
  discovers wallets - use-wallet v5, or anything on
  `@agoralabs-sh/avm-web-provider` - it finds SignA without any of the code
  below. The provider answers `enable` and `signTransactions` by delegating to
  the methods documented here, so behaviour and validation are identical.
- **`window.AlgoSigner`**, the pre-1.10.0 object. Still present for existing
  integrations; see the [legacy integration guide](legacy-dApp-integration.md).
  Do not start new work there.

## Methods

- [algorand.enable(enableOpts)](#algorandenableenableopts-enableopts)
- [algorand.signTxns([txnObjects, ...])](#algorandsigntxnstxnobjects-txnobject--txnobject)
- [algorand.postTxns([signedTxns, ...])](#algorandposttxnsstxns-signedtxn--signedtxn)
- [algorand.signAndPostTxns([txnObjects, ...])](#algorandsignandposttxnstxnobjects-txnobject--txnobject)

## Misc

[Rejection Messages](#rejection-messages)

[Working with Custom Networks](#custom-networks)

[Helper Encoding Functions](#encoding-functions)

## Method Detail

### algorand.enable(enableOpts: EnableOpts)

In order for dApps to interact with SignA, they must first request access by calling `algorand.enable()`. This will prompt the user to select which accounts to share with the dApp as well as the network they wish to operate on.

The accounts offered are the intersection of two things: what the wallet holds, and what is visible on the requested network. An account exists once in SignA and the user chooses which networks it appears on, so an address they hold may legitimately not be offered on the network you asked for.

A connection is scoped to one origin and one network, and it persists until the user revokes it from the wallet's Connections page. It survives a lock and a browser restart, so a returning user may be pre-authorized. It never confers signing authority: every signature still requires the user's passphrase at the moment of signing.

After the user selects which network and accounts to share with the dApp, they'll be returned by SignA as the response to the `algorand.enable()` call.


```
export type EnableResponse = {
  genesisID:    specific genesis ID shared by the user,
  genesisHash:  specific genesis hash shared by the user,
  accounts:     array of specific accounts shared by the user
};
```

**Request**

```js
await algorand.enable();
```

**Response**

```json
{
  "genesisID": "NETWORK_ID",
  "genesisHash": "NETWORK_HASH",
  "accounts": [
    "ACCOUNT_1",
    "ACCOUNT_2",
  ],
}
```

In cases were the dApp wishes to request specific accounts, a specific network or both; they will be able to do so by providing additional parameters to the `algorand.enable()` call.

```
export type EnableOpts = {
  genesisID?:   [optional] specific genesis ID requested by the dApp,
  genesisHash?: [optional] specific genesis hash requested by the dApp,
  accounts?:    [optional] array of specific accounts requested by the dApp,
};
```
If either `EnableOpts.genesisID` or `EnableOpts.genesisHash` are provided, they must match one of the networks available in SignA. Where there is ambiguity, such as two networks sharing a `genesisID`, the user will be prompted to choose between the available matching networks.

If neither `EnableOpts.genesisID` nor `EnableOpts.genesisHash` are provided, the user will be able to select which network they want to grant access to for the dApp.

If `EnableOpts.accounts` is provided, the requested accounts will appear as required for the user and the user will be prompted to grant control over the specified accounts; these accounts will be positioned at the start of the array in the response. The user may share additional accounts than those requested by the dApp, in which case the accounts will be appended at the end of the returning account array. The user may also choose to share fewer accounts than those requested, in which case the promise will be rejected and the rejected accounts will be found inside the `data.accounts` property of the error.

If `EnableOpts.accounts` is not provided, the user will be prompted to select which accounts they wish to share with the dApp.

**Request**

```js
await algorand.enable({
  genesisID: 'mainnet-v1.0',
  accounts: ['REQUESTED_ACCOUNT'],
});
```

**Response**

User granting access to an additional account:
```json
{
  "genesisID": "mainnet-v1.0",
  "genesisHash": "USER_SELECTED_HASH",
  "accounts": [
    "REQUESTED_ACCOUNT",
    "ADDITIONAL_USER_SELECTED_ACCOUNT",
  ],
}
```

User not granting access to an account requested by the dApp:
```json
{
  "code": 4400,
  "message": "...",
  "data": {
    "accounts": [
      "REQUESTED_ACCOUNT",
    ],
  },
}
```

#### Regarding multiple calls to `algorand.enable()`

In cases where an enable call is made after a user has already authorized the dApp there will be additional checks before prompting the user. There is an ephemeral single network cached history and if the requested network from the dApp matches the previous one and the accounts are the same or a subset of the cached approved accounts the cache will be returned instead of prompting the user. 

If the network is different or there are additional accounts requested compared to the cache the user will be prompted to re-authorize the request.

## Working with Transactions

Transactions provided to SignA will be validated against the Algorand JS SDK transaction types. Field names must match and the whole transaction will be rejected otherwise. 

#### Transaction Requirements

Transactions objects need to be presented with the following structure:

```
export type TxnObject = {
  txn:        Base64-encoded string of a transaction binary,
  signers?:   [optional] array of addresses to sign with (defaults to the sender),
  stxn?:      [optional] Base64-encoded string of a signed transaction binary,
  multisig?:  [optional] extra metadata needed for multisig transactions,
  authAddr?:  [optional] used to specify which account is doing the signing when dealing with rekeyed accounts,
};
```

In order to facilitate conversion between different formats and encodings, [helper encoding functions](#encoding-functions) are available on the `algorand.encoding.*` namespace.

Also available on transactions built with the JS SDK is the `.toByte()` method that converts the SDK transaction object into it's binary format.

### algorand.signTxns(txnObjects: TxnObject[] | TxnObject[][])

Send transaction objects, conforming to the Algorand JS SDK, to SignA for approval. The *network* is determined from the `genesis-id` property.
If approved, the response is an array of base64-encoded signed transaction objects.

**Request**

```js
algorand.signTxns([
  {
    txn: 'iqNhbXRko2ZlZc0D6KJmds4A259Go2dlbqx0ZXN0bmV0LXYxLjCiZ2jEIEhjtRiks8hOyBDyLU8QgcsPcfBZp6wg3sYvf3DlCToio2dycMQgdsLAGqgrtwqqQS4UEN7O8CZHjfhPTwLHrB1A2pXwvKGibHbOANujLqNyY3bEIK0TEDcptY0uFvk2V5LDVzRfdz7O4freYHEuZbpI+6hMo3NuZMQglmyhKUPeU2KALzt/Jcs0GQ55k2vsqZ4pGeNlzpnYLbukdHlwZaNwYXk=',
  },
]);
```

**NOTE:** Even though the method accepts an array of transactions, it requires atomic transactions that share a groupId and will error on non-atomic groups.

**Response**

```json
[
  "gqNzaWfEQL6mW/7ss2HKAqsuHN/7ePx11wKSAvFocw5QEDvzSvrvJdzWYvT7ua8Lc0SS0zOmUDDaHQC/pGJ0PNqnu7W3qQKjdHhuiaNhbXQGo2ZlZc4AA7U4omZ2zgB+OrujZ2VurHRlc3RuZXQtdjEuMKJnaMQgSGO1GKSzyE7IEPItTxCByw9x8FmnrCDexi9/cOUJOiKibHbOAH4+o6NyY3bEIHhydylNDQQhpD9QdKWejLCMBgb5UYJTGCfDW3KgLsI+o3NuZMQgZM5ZNuFgR8pz2dHBgDlmHolfGgF96zX/X4x2bnAJ3aqkdHlwZaNwYXk=",
]
```

**Example**

```js
// Connect to SignA
await algorand.enable();

// Create an Algod client to get suggested transaction params
let client = new algosdk.Algodv2(token, server, port, headers);
let suggestedParams = await client.getTransactionParams().do();

// Use the JS SDK to build a Transaction
let sdkTxn = new algosdk.Transaction({
  to: 'RECEIVER_ADDRESS',
  from: 'SENDER_ADDRESS',
  amount: 100,
  ...suggestedParams,
});

// Get the binary and base64 encode it
let binaryTxn = sdkTxn.toByte();
let base64Txn = algorand.encoding.msgpackToBase64(binaryTxn);

let signedTxns = await algorand.signTxns([
  {
    txn: base64Txn,
  },
]);
```

Alternatively, you can provide multiple arrays of transactions at once. Same rules regarding the contents of the groups apply.

**Request**

```js
algorand.signTxns([
  [
    {
      txn: 'iqNhbXRko2ZlZc0D6KJmds4A259Go2dlbqx0ZXN0bmV0LXYxLjCiZ2jEIEhjtRiks8hOyBDyLU8QgcsPcfBZp6wg3sYvf3DlCToio2dycMQgdsLAGqgrtwqqQS4UEN7O8CZHjfhPTwLHrB1A2pXwvKGibHbOANujLqNyY3bEIK0TEDcptY0uFvk2V5LDVzRfdz7O4freYHEuZbpI+6hMo3NuZMQglmyhKUPeU2KALzt/Jcs0GQ55k2vsqZ4pGeNlzpnYLbukdHlwZaNwYXk=',
    },
  ],
  [
    {
      txn: 'iaRhZnJ6w6RmYWRkxCCWbKEpQ95TYoAvO38lyzQZDnmTa+ypnikZ42XOmdgtu6RmYWlkzgDITmiiZnbOAQNjIKNnZW6sdGVzdG5ldC12MS4womdoxCBIY7UYpLPITsgQ8i1PEIHLD3HwWaesIN7GL39w5Qk6IqJsds4BA2cIo3NuZMQglmyhKUPeU2KALzt/Jcs0GQ55k2vsqZ4pGeNlzpnYLbukdHlwZaRhZnJ6',
    },
  ],
]);
```

**Response**

```json
[
  [
    "gqNzaWfEQL6mW/7ss2HKAqsuHN/7ePx11wKSAvFocw5QEDvzSvrvJdzWYvT7ua8Lc0SS0zOmUDDaHQC/pGJ0PNqnu7W3qQKjdHhuiaNhbXQGo2ZlZc4AA7U4omZ2zgB+OrujZ2VurHRlc3RuZXQtdjEuMKJnaMQgSGO1GKSzyE7IEPItTxCByw9x8FmnrCDexi9/cOUJOiKibHbOAH4+o6NyY3bEIHhydylNDQQhpD9QdKWejLCMBgb5UYJTGCfDW3KgLsI+o3NuZMQgZM5ZNuFgR8pz2dHBgDlmHolfGgF96zX/X4x2bnAJ3aqkdHlwZaNwYXk=",
  ],
  [
    "gqNzaWfEQC8ZIPYimAypJD2TmEQjuWxEEk8/gJbBegEHdtyKr6TuA78otKIEB9PYQimgMLGn87YOEB6GgRe5vjWRTuWGsAqjdHhuiaRhZnJ6w6RmYWRkxCCWbKEpQ95TYoAvO38lyzQZDnmTa+ypnikZ42XOmdgtu6RmYWlkzgDITmiiZnbOAQNjO6NnZW6sdGVzdG5ldC12MS4womdoxCBIY7UYpLPITsgQ8i1PEIHLD3HwWaesIN7GL39w5Qk6IqJsds4BA2cjo3NuZMQglmyhKUPeU2KALzt/Jcs0GQ55k2vsqZ4pGeNlzpnYLbukdHlwZaRhZnJ6",
  ]
]
```

The signed transactions can then be sent using the SDK (example below) or using the [algorand.postTxns()](#algorandposttxnsstxns-signedtxn--signedtxn) method.

```js
// Get the base64 encoded signed transaction and convert it to binary
let binarySignedTxn = algorand.encoding.base64ToMsgpack(signedTxns[0]);

// Send the transaction through the SDK client
await client.sendRawTransaction(binarySignedTxn).do();
```

### Atomic Transactions

For Atomic transactions, provide an array of transaction objects with the same group ID, _provided in the same order as when the group was assigned_.

**Example**

```js
let txn1 = new algosdk.Transaction({
  to: 'SECOND_ADDRESS',
  from: 'FIRST_ADDRESS',
  amount: 100,
  ...suggestedParams,
});
let txn2 = new algosdk.Transaction({
  to: 'FIRST_ADDRESS',
  from: 'SECOND_ADDRESS',
  amount: 100,
  ...suggestedParams,
});

// Assign a Group ID to the transactions using the SDK
algosdk.assignGroupID([txn1, txn2]);

let binaryTxns = [txn1.toByte(), txn2.toByte()];
let base64Txns = binaryTxns.map((binary) => algorand.encoding.msgpackToBase64(binary));

let signedTxns = await algorand.signTxns([
  {
    txn: base64Txns[0],
  },
  {
    txn: base64Txns[1],
  },
]);
```

The signed transaction array can then be sent using the SDK.

```js
let binarySignedTxns = signedTxns.map((stxn) => algorand.encoding.base64ToMsgpack(stxn));
await client.sendRawTransaction(binarySignedTxns).do();
```

#### Reference Atomic transactions

In case not all group transactions belong to accounts on SignA, you can set the `signers` field of the transaction object as an empty array to specify that it's only being sent to SignA for reference and group validation, not for signing. Reference transactions should look like this:

```js
{
  txn: 'B64_TXN',
  signers: [],
  // This tells SignA that this transaction is not meant to be signed
}
```

`algorand.signTxns()` will return `null` in the position(s) where reference transactions were provided. In these instances, you'd have to sign the missing transaction(s) by your own means before they can be sent. This is useful for transactions that require external signing, like `lsig` transactions.

#### Providing Signed reference transaction(s)

You can provide a signed reference transaction to SignA via the `stxn` field of the transaction object for it to be validated and returned as part of the group. For the transaction(s) where `stxn` was provided, `algorand.signTxns()` will return the `stxn` string in the same position of the response array as the corresponding reference transaction(s) instead of `null`.

**Example**

```js
let txn1 = new algosdk.Transaction({
  to: 'EXTERNAL_ACCOUNT',
  from: 'ACCOUNT_IN_SIGNA',
  amount: 100,
  ...suggestedParams,
});
let txn2 = new algosdk.Transaction({
  to: 'ACCOUNT_IN_SIGNA',
  from: 'EXTERNAL_ACCOUNT',
  amount: 100,
  ...suggestedParams,
});

algosdk.assignGroupID([txn1, txn2]);
let binaryTxns = [txn1.toByte(), txn2.toByte()];
let base64Txns = binaryTxns.map((binary) => algorand.encoding.msgpackToBase64(binary));

let signedTxns = await algorand.signTxns([
  {
    txn: base64Txns[0],
  },
  {
    txn: base64Txns[1],
    signers: [],
    stxn: 'MANUALLY_SIGNED_SECOND_TXN_B64',
  },
]);
```

**Response**

```js
[
  'SIGNA_SIGNED_B64',
  'MANUALLY_SIGNED_SECOND_TXN_B64',
];
```

#### Signing reference transactions manually
 
In case you can't or don't want to provide the `stxn`, the provided transaction should look like this:

**Example**

```js
let signedTxns = await algorand.signTxns([
  {
    txn: base64Txns[0],
  },
  {
    txn: base64Txns[1],
    signers: [],
  },
]);
```

**Response**

```js
[
  'SIGNA_SIGNED_B64',
  null,
];
```

Afterwards, you can sign and send the remaining transaction(s) with the SDK:

```js
// Convert first transaction to binary from the response
let signedTxn1Binary = algorand.encoding.base64ToMsgpack(signedTxns[0]);
// Sign leftover transaction with the SDK
let externalAccount = algosdk.mnemonicToSecretKey('EXTERNAL_ACCOUNT_MNEMONIC');
let signedTxn2Binary = txn2.signTxn(externalAccount.sk);

await client.sendRawTransaction([signedTxn1Binary, signedTxn2Binary]).do();
```

#### Multisig Transactions

For Multisig transactions, an additional metadata object is required that adheres to the [Algorand multisig parameters](https://developer.algorand.org/docs/features/accounts/create/#how-to-generate-a-multisignature-account) structure when creating a new multisig account:

```js
{
  version: number,
  threshold: number,
  addrs: string[]
}
```

`algorand.signTxns()` will validate that the resulting multisig address made from the provided parameters matches the sender address and try to sign with every account on the `addrs` array that is also on SignA.

**NOTE:** `algorand.signTxns()` only accepts unsigned multisig transactions. In case you need to add more signatures to partially signed multisig transactions, please use the SDK.

**Example**

```js
let multisigParams = {
  version: 1,
  threshold: 1,
  addrs: ['FIRST_ADDRESS', 'SECOND_ADDRESS', 'ADDRESS_NOT_IN_SIGNA'],
};

let multisigAddress = algosdk.multisigAddress(multisigParams);

let multisigTxn = new algosdk.Transaction({
  to: 'RECEIVER_ADDRESS',
  from: multisigAddress,
  amount: 100,
  ...suggestedParams,
});

// Get the binary and base64 encode it
let binaryMultisigTxn = multisigTxn.toByte();
let base64MultisigTxn = algorand.encoding.msgpackToBase64(binaryMultisigTxn);

// This returns a partially signed Multisig Transaction with signatures for FIRST_ADDRESS and SECOND_ADDRESS
let signedTxns = await algorand.signTxns([
  {
    txn: base64MultisigTxn,
    msig: multisigParams,
  },
]);
```

In case you want to specify a subset of addresses to sign with, you can add them to the `signers` list on the transaction object, like so:

```js
// This returns a partially signed Multisig Transaction with signatures for SECOND_ADDRESS
let signedTxns = await algorand.signTxns([
  {
    txn: base64MultisigTxn,
    msig: multisigParams,
    signers: ['SECOND_ADDRESS'],
  },
]);
```
#### Authorized Addresses

When dealing with rekeyed accounts, the authorized address to be used to sign the transaction differs from the rekeyed account. The `TxnObject.authAddr` field allows to specify a different sender address in those cases.

**NOTE:** If specified, SignA will sign the transaction using this authorized address even if it sees the sender address was not rekeyed to authAddr. This is because the sender may be rekeyed before the transaction is committed.

**Example**

```js
let txn = new algosdk.Transaction({
  to: 'REKEYED_ACCOUNT',
  from: 'REKEYED_ACCOUNT',
  amount: 100,
  ...suggestedParams,
});

let base64Txn = algorand.encoding.msgpackToBase64(txn.toByte());

let signedTxns = await algorand.signTxns([
  {
    txn: base64Txn,
    authAddr: 'AUTHORIZED_ADDRESS_FOR_REKEY',
  },
]);
```

**Special Considerations**

In cases where both `TxnObject.authAddr` and `TxnObject.msig` are provided, both addresses need to match or the transaction will be rejected.

In cases where both `TxnObject.authAddr` and a `TxnObject.signers` array with a single address are provided, but no `TxnObject.msig` was provided; `authAddr` needs to match the `signer` or the transaction will be rejected.

### algorand.postTxns(stxns: SignedTxn[] | SignedTxn[][])

Send a base64 encoded signed transaction blob to SignA to transmit to the Network.

**Request**

```js
algorand.postTxns([signedTx]);
```

**Response**

```
export type PostResult = {
  txnIDs: string[] | string[][],
};
```

```json
{
  "txnIDs": [
    "OKU6A2QYMRSZAUEJUZL3PW5XKLTA6TKWQHIIBXDCO3KT5OHCULBA"
  ]
}
```

When providing more than one group of transactions at the same time, all transactions will be sent to the network simultaneously. In case some of the groups were unable to be commited to the network, the returning error will provide information on which ones were successful, as well as the reason behind the failed ones.

```
export type PartialPostError = {
  successTxnIDs: (string | null)[][],
  data: (string | null)[][],
  code: number,
  message: string,
};
```
The `successTxnIDs` property includes the IDs of the transactions that were succesfully commited to the network while the `data` property includes the reasons behind the failing ones. Both arrays are *ordered* respecting the original positions of the provided groups of transactions.

**Request**

```js
algorand.postTxns([successfulTxn], [rejectedTxn]);
```

**Response**

```json
{
  "successTxnIDs": [
    "OKU6A2QYMRSZAUEJUZL3PW5XKLTA6TKWQHIIBXDCO3KT5OHCULBA",
    null,
  ],
  "data": [
    null,
    "Reason behind network refusal",
  ],
  "code": 4400,
  "message": "...",
}
```

### algorand.signAndPostTxns(txnObjects: TxnObject[] | TxnObject[][])

This methods takes the input from [algorand.signTxns()](#algorandsigntxnstxnobjects-txnobject--txnobject) and internally posts the signed transactions to the network before returning the response of [algorand.postTxns()](#algorandposttxnsstxns-signedtxn--signedtxn) to the dApp.

**Request**

```js
algorand.signTxns([
  {
    txn: 'iqNhbXRko2ZlZc0D6KJmds4A259Go2dlbqx0ZXN0bmV0LXYxLjCiZ2jEIEhjtRiks8hOyBDyLU8QgcsPcfBZp6wg3sYvf3DlCToio2dycMQgdsLAGqgrtwqqQS4UEN7O8CZHjfhPTwLHrB1A2pXwvKGibHbOANujLqNyY3bEIK0TEDcptY0uFvk2V5LDVzRfdz7O4freYHEuZbpI+6hMo3NuZMQglmyhKUPeU2KALzt/Jcs0GQ55k2vsqZ4pGeNlzpnYLbukdHlwZaNwYXk=',
  },
]);
```

**Response**

```json
{
  "txnIDs": [
    "OKU6A2QYMRSZAUEJUZL3PW5XKLTA6TKWQHIIBXDCO3KT5OHCULBA"
  ]
}
```

## Custom Networks

MainNet, TestNet, and BetaNet are built in. Users may add any other network, so do not assume the set is fixed. A setup guide is in [add-network.md](add-network.md).

- [algorand.enable()](#algorandenableenableopts-enableopts) calls accept genesis IDs/hashes that have been added to the user's custom network list as valid networks.
  - Non-matching `genesisID` or `genesisHash` will result in a error.
- Transaction requests will require a valid matching `genesisID`, even for custom networks.

## Rejection Messages

Some of the following error codes may be returned when interacting with SignA. When available, any additional info regarding the error will be found on the `data` property of the error.

| Error Code | Description | Additional notes |
| ---------- | ----------- | ---------------- |
| 4000 | An unknown error occured. | N/A |
| 4001 | The user rejected the signature request. | N/A |
| 4100 | The requested operation and/or account has not been authorized by the user. | This is usually due to the connection between the dApp and the wallet becoming stale and the user [needing to reconnect](connection-issues.md). Otherwise, it may signal that you are trying to sign with private keys not found on SignA. | 
| 4200 | The wallet does not support the requested operation. | Users need to have imported or created an account on SignA before connecting to dApps, as well as succesfully having configured any custom networks required. |
| 4201 | The wallet does not support signing that many transactions at a time. | The max number of transactions per group is 16. For Ledger devices, they currently can't sign more than one transaction at the same time. |
| 4202 | The wallet was not initialized properly beforehand. | The extension user has not authorized requests from this website. |
| 4300 | The input provided is invalid. | SignA rejected some of the transactions due to invalid fields. |
| 4400 | Some transactions were not sent properly. | Some, but not all of the transactions were able to be posted to the network. The IDs of the succesfully posted transactions as well as information on the failing ones are provided on the error.

Additional information, if available, would be provided in the `data` field of the error object.

Returned errors have the following object structure:

```
{
  message: string;
  code: number;
  name: string;
  data?: any;
}
```

Errors may be passed back to the dApp from the Algorand JS SDK if a transaction is valid, but has some other issue - for example, insufficient funds in the sending account.

## Encoding Functions

`algorand.enconding.*` contains a few different methods in order to help with the different formats and encodings that are needed when working with dApps and the SDK.

```
  algorand.encoding.msgpackToBase64(): receives a binary object (as a Uint8Array) and returns the corresponding base64 encoded string,
  algorand.encoding.base64ToMsgpack(): receives a base64 encoded string and returns the corresponding binary object (as a Uint8Array),
  algorand.encoding.stringToByteArray(): receives a plain unencoded string and returns the corresponding binary object (as a Uint8Array),
  algorand.encoding.byteArrayToString(): receives a binary object (as a Uint8Array) and returns the corresponding plain unencoded string,
```
