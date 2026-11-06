# Ledger Hardware Accounts

> **Untested on this codebase.** The Ledger path was ported to algosdk 3.x and
> current transport libraries and it compiles, but no signing has been
> exercised against a physical device since the fork. Verify every byte on the
> device screen, and try TestNet before MainNet.

SignA can hold Ledger accounts alongside accounts whose keys live in the
encrypted vault. The device keeps the signing key; the wallet only ever
receives a signature. It works for transactions started in the wallet and for
transactions sent by a dApp.

## Requirements

- **A Chromium-based browser.** The connection uses WebHID, which Firefox does
  not implement. Firefox reports this rather than failing silently.
- The Algorand app installed and open on the device, and the device unlocked

## Adding an address

1. Select the network you want the account on. Accounts are per network, so
   the same device address has to be added to each network you use it on.
2. From the account list, choose **Add Account**, then **Link hardware device
   account**
3. The wallet opens a new tab to talk to the device. WebHID needs a real page
   with a user gesture, which a side panel cannot always provide.
4. Give the account a name, confirm the device is connected and unlocked with
   the Algorand app open, then retrieve addresses and pick one
5. Save. The tab can be closed once the address appears in your account list.

## Signing

Hardware accounts look like any other account in the interface. When you send
from one, or a dApp asks one to sign:

1. Enter your passphrase at the prompt as usual
2. The wallet recognises the address as a hardware account and opens a tab
   showing the transaction
3. Review it in the tab, then press **Send to device**
4. **Verify every field on the device screen.** The tab is a convenience; the
   device display is what you are actually approving.
5. Approve on the device

For a wallet-initiated transaction, the signed transaction is submitted and
the transaction ID is shown. For a dApp transaction, the signed bytes are
returned to the dApp tab exactly as a software signature would be.

## Limits

- Groups are supported, and multiple transactions in one group are signed in
  sequence on the device
- Multisig accounts cannot be signed for with a hardware account
- If the device screen is blank or you get a "not supported" error, the
  Algorand app is not open. Open it and retry.
