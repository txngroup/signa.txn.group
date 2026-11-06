# Connection Troubleshooting

If a dApp will not connect to SignA, or it connects to the wrong accounts, a
stale grant is usually the cause. Grants are held per origin and are cleared
when the wallet locks.

## Clear the connection

1. Press any **Disconnect** button the dApp offers. Most wallet libraries
   forget their side of the connection this way.
2. Open the gear menu in SignA and choose **Connections**. Find the site under
   the network you are on and revoke it.
3. Reload the dApp tab and connect again.

Logging out no longer clears connections. They are stored per network and
survive a lock and a browser restart, so the Connections page is the way to
drop one. A revoked site stops seeing your addresses and has to ask again.

## If it still fails

- **Check the network.** A dApp requesting a specific genesis ID will be
  refused if the wallet is on a different network. Switch networks in the
  header and reconnect.
- **Check the accounts.** The connection prompt shares only the accounts you
  tick. If the dApp cannot see an address, reconnect and include it.
- **Check the origin.** Grants are keyed to the exact origin, so
  `https://example.com` and `https://www.example.com` are separate
  connections.
- **Reload the extension.** From `chrome://extensions/` (or
  `about:debugging` on Firefox), reload SignA and try once more.
- **Check the account is visible on that network.** An account can be hidden
  from a network on its account page, and a hidden account is dropped from any
  connection on that network. Re-enable it there, then reconnect.
- Failing all of that, clear the site's cookies and storage in your browser
  and reconnect from a fresh page load.
