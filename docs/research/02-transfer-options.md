# 02: Ways to get a book onto KOReader without the calibre GUI

> [!WARNING]
> **LLM Disclosure**
>
> This post was authored by claude-opus-5-5 on behalf of @JPHutchins. The question was whether a small CLI could replace the calibre GUI for the usual task of sending one book to KOReader. Three researcher agents surveyed the options, and their claims were then checked against pinned sources.

**Status (2026-11-21):** survey done. No existing tool fits. The chosen direction is a new server that speaks calibre's wireless protocol (see the [README](../../README.md)).

## Requirement

**Reuse KOReader's built-in calibre client** ("Connect to calibre") so you keep its conveniences:

- received books go into the configured inbox directory,
- the metadata sent with each book is added to KOReader's `metadata.calibre` ([`CalibreMetadata:addBook`, wireless.lua L761][ko-761]), which the plugin's calibre search uses,
- the reader shows a "Received file" notice.

## Summary

| Route | Push or pull | Uses KOReader's calibre client | Verdict |
|---|---|---|---|
| `ebook-device` (calibre CLI) | push | n/a | **No.** It never starts the wireless listener. |
| `calibre-debug -e` script that drives calibre's wireless driver | push | yes | **Plausible, untested.** |
| `calibre-server` and KOReader's OPDS browser | pull | no | **Works**, but needs several taps on the reader per book. |
| KOReader SSH server plugin | push | no | **Works on supported devices.** It skips calibre's metadata. |
| KOReader cloud storage (Dropbox, FTP, WebDAV) | pull | no | **Works.** Taps on the reader per book. |
| Existing third-party calibre-protocol servers | push | yes | **None found that's verified with KOReader.** |
| New server implementing calibre's wireless protocol | push | yes | **Chosen.** See [03-wire-protocol.md](03-wire-protocol.md). |

## Details

### `ebook-device`

- It calls `startup()` on each plugin, then `detect_managed_devices` for plugins that manage their own device presence ([`devices/cli.py` L208–L212][cal-cli-208]).
- The wireless driver's `startup()` sets `listen_socket = None` ([driver.py L1854–L1855][cal-driver-1854]). Only `_startup_on_demand` creates the listener ([L1858][cal-driver-1858]), and `ebook-device` never calls it.
- The result is "Unable to find a connected ebook reader." ([L225][cal-cli-225]).

### `calibre-debug -e` script

- The GUI's sequence is: `start_plugin` ([L2016][cal-driver-2016]), poll `detect_managed_devices`, `open`, `get_device_information`, then `upload_books`.
- A script could reproduce that and send `GET_DEVICE_INFORMATION` immediately, but this is **untested**.
- The calibre GUI must be closed because both bind the same ports.
- It still depends on a full calibre install.

### `calibre-server` and OPDS

- The content server exposes OPDS at `/opds` ([`srv/opds.py` L470][cal-opds-470]), and KOReader's OPDS browser can download from it.
- Adding books while the server is running must go through the server: `calibredb --with-library <server-url>`. Opening the library directly fails with "Another calibre program … is running" ([`db/cli/main.py` L160–L165][cal-dbcli-160]).

### KOReader SSH plugin

- It runs a bundled `dropbear` only if the binary is present ([`SSH.koplugin/main.lua` L21][ko-ssh-21]).
- The default port is `2222` ([L32][ko-ssh-32]). Keys go in `settings/SSH/authorized_keys` ([L18][ko-ssh-18]).
- Device home directories:
  - Kobo: `/mnt/onboard` ([L190][ko-kobo-190])
  - Kindle: `/mnt/us` ([L405][ko-kindle-405])
  - PocketBook: `/mnt/ext1` ([L40][ko-pb-40])
- A transfer looks like `cat book.epub | ssh -p 2222 root@<reader> 'cat > /mnt/onboard/book.epub'`. This is untested here, and the login user is taken from the [KOReader SSH wiki][ko-wiki-ssh] rather than the source.
- Trade-offs: you have to start the SSH server on the reader each session, and no calibre metadata or search integration is kept.
- Not chosen. It works on this reader, but it doesn't meet the requirement above.

### KOReader cloud storage

- Providers `dropbox.lua`, `ftp.lua` and `webdav.lua` exist under [`plugins/cloudstorage.koplugin/providers`][ko-cloud].
- You upload with any CLI (for example `rclone`) and download on the reader, which means taps per book.

### Third-party calibre-protocol servers

- The closest match is [`scripts/calibre-simulator.mjs` in bigbag/papyrix-reader][pap].
  - It is a Node.js simulator written to test a different e-reader's firmware.
  - Its `SEND_BOOK` payload has no `metadata` field ([L378–L389][pap-378]). KOReader refuses that with "Invalid book metadata" ([wireless.lua L694][ko-694]).
- Other repositories found were device-side clients (Kindle tools, KOReader plugins), not servers.
- The search covered GitHub repo and code search plus a web search. It was **not exhaustive**: PyPI, crates.io and npm were not searched directly.

[ko-761]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L761
[ko-694]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L694
[ko-ssh-18]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/SSH.koplugin/main.lua#L18
[ko-ssh-21]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/SSH.koplugin/main.lua#L21
[ko-ssh-32]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/SSH.koplugin/main.lua#L32
[ko-kobo-190]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/frontend/device/kobo/device.lua#L190
[ko-kindle-405]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/frontend/device/kindle/device.lua#L405
[ko-pb-40]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/frontend/device/pocketbook/device.lua#L40
[ko-cloud]: https://github.com/koreader/koreader/tree/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/cloudstorage.koplugin/providers
[ko-wiki-ssh]: https://github.com/koreader/koreader/wiki/SSH
[cal-cli-208]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/cli.py#L208-L212
[cal-cli-225]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/cli.py#L225
[cal-driver-1854]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L1854-L1855
[cal-driver-1858]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L1858
[cal-driver-2016]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L2016
[cal-opds-470]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/srv/opds.py#L470
[cal-dbcli-160]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/db/cli/main.py#L160-L165
[pap]: https://github.com/bigbag/papyrix-reader/blob/12b7e6b28fe9a970a709851117b08096ee6ea941/scripts/calibre-simulator.mjs
[pap-378]: https://github.com/bigbag/papyrix-reader/blob/12b7e6b28fe9a970a709851117b08096ee6ea941/scripts/calibre-simulator.mjs#L378-L389
