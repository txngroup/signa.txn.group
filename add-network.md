# Adding a Network

SignA ships with MainNet, TestNet, and BetaNet. Those three are locked: their
endpoints are compiled into the extension and cannot be edited or deleted from
the interface. Everything else is added by hand.

Common reasons to add one: a local [AlgoKit](https://github.com/algorandfoundation/algokit-cli)
or sandbox node, a private network, or a hosted provider you have an API key
for.

## What you need

- **Algod URL**, including the port if the node is not on 443
- **Indexer URL** - optional. Leave it blank to run against algod alone.
  Balances, signing, and dApp connections all work without one; transaction
  history does not, and the wallet says so rather than erroring.
- **API headers** - only if your provider requires them

The genesis ID and genesis hash are fetched from the node when you save, so
you do not enter them. This is what lets a custom network match an
`algorand.enable()` request from a dApp.

## Steps

1. Open the wallet and select the gear icon in the header
2. Choose **Network Configuration**
3. Choose **New Network**
4. Give it a display name. The name has to be unique, and it is how the
   network appears in the network switcher and in dApp prompts.
5. Enter the algod URL, and the indexer URL if you have one
6. Add headers if your provider needs them (see below)
7. Press **Check URLs** to confirm both endpoints respond before committing
8. **Save**, and enter your passphrase

Accounts are per network, so a newly added network starts empty. Import or
create accounts on it the usual way.

## Network headers

For a node that uses the standard Algorand API token, paste the token straight
into the field. It is sent as the `x-algo-api-token` header.

If algod and indexer need different keys, or the provider uses a header other
than the standard token, enter JSON instead:

```json
{ "Algod": { "x-api-key": "xxxxxxxxx" }, "Indexer": { "x-api-key": "xxxxxxxxx" } }
```

Any header name works; the object is merged into every request to that
endpoint. The JSON must be on a single line.

Headers are stored with the network in the encrypted vault and are only sent
to the URLs configured on that network.

## Editing and deleting

Custom networks can be renamed, repointed, or deleted from the same screen.
Renaming a network carries its accounts across. Deleting one removes its
accounts from the wallet, so export any mnemonics you still need first.
