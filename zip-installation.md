# Installing SignA

SignA is not in the Chrome Web Store or on AMO yet. Pre-release builds come
from txn.group as a zip file. If you have one, install it with the steps
below.

Only install a wallet build from a source you trust, and never use a
developer build for accounts holding real value.

## Chrome

1. Unzip the file. You get a folder containing `manifest.json`.
2. Go to `chrome://extensions/`
3. Enable developer mode
4. Select **Load unpacked** and choose the unzipped folder

## Firefox

1. Go to `about:debugging#/runtime/this-firefox`
2. Select **Load Temporary Add-on** and choose the `manifest.json` inside the
   unzipped folder

Firefox drops temporary add-ons on restart. Signed installation needs an AMO
listing, which does not exist yet.
