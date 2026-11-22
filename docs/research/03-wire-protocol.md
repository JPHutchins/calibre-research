# 03: calibre wireless device protocol, as KOReader speaks it

> [!WARNING]
> **LLM Disclosure**
>
> This post was authored by claude-opus-5-5 on behalf of @JPHutchins. These are protocol notes for a planned replacement server that pushes one book to KOReader's built-in calibre client. Everything below comes from reading the pinned sources; none of it has been tested on the wire yet.

**Status (2026-11-21):** derived from source and not yet tested. Each step cites the KOReader handler ([`1294bb3`][ko]) and calibre's sender ([`v9.6.0`][cal]).

## Roles

- The **server** is calibre, or a replacement for it. It listens on TCP and drives the whole session.
- The **client** is KOReader's `calibre.koplugin`. It finds the server, connects to it, and only ever replies.

## Discovery

- **KOReader's broadcast:** it binds UDP port `8134` ([L33][ko-33]) and broadcasts `"hello"` to ports `54982, 48123, 39001, 44044, 59678` ([L31][ko-31], [L150–L170][ko-150]). calibre listens on the same ports ([driver.py L249][cal-249]).
- **The server's reply:** `calibre wireless device client (on <hostname>);<content_server_port>,<wireless_port>` ([driver.py L128–L133][cal-128]).
- **What KOReader reads from the reply:** only the host it came from and the last port ([L164][ko-164]).
- **Skipping discovery:** a fixed address saved in KOReader (`calibre_wireless_url`) bypasses it ([L285][ko-285]).

## Framing

- Each control message is `<decimal byte length><JSON array>`, where the array is `[opcode, {args}]`.
  - calibre writes it at [driver.py L645][cal-645]; KOReader writes it at [L486–L490][ko-486].
  - KOReader parses it at [L432–L445][ko-432].
- Replies use opcode `OK` (0) or `ERROR` (20).
- Book bytes are sent raw, with no framing, right after the client's `OK` to `SEND_BOOK` (see step 5).
- Opcode tables: KOReader [L35–L58][ko-35], calibre [L251][cal-251].

## Minimal "send one book" session

| # | Server sends | KOReader does | Sources |
|---|---|---|---|
| 1 | `GET_INITIALIZATION_INFO` (9) with `calibre_version` (array of numbers), optional `passwordChallenge`, and others | Records the version and replies `OK` with `acceptedExtensions`, `passwordHash`, `maxBookContentPacketLen: 4096` and more | calibre [L1053–L1064][cal-1053]; KOReader [L496–L539][ko-496] |
| — | **Next message within 1 s** | Otherwise "handshake timeout" and disconnect | [L326–L335][ko-330], [01](01-handshake-failure.md) |
| 2 | `GET_DEVICE_INFORMATION` (3) `{}` | `OK` with `device_info: {device_store_uuid, device_name}`, `version`, `device_version` | [L580–L591][ko-580] |
| 3 | `SET_CALIBRE_DEVICE_INFO` (1) with the device-info object | Saves `driveinfo.calibre` and replies `OK {}` | [L593–L597][ko-593] |
| 4 | `SEND_BOOK` (8) `{lpath, length, metadata, thisBook, totalBooks, willStreamBooks, willStreamBinary, wantsSendOkToSendbook, canSupportLpathChanges}` | Validates it, replies `OK {}`, and switches the socket to raw mode | calibre [L702–L711][cal-702]; KOReader [L658–L783][ko-658] |
| 5 | `length` raw bytes | Writes the file, adds the metadata to `metadata.calibre`, shows "Received file". Sends no reply after the bytes. | calibre [L722–L729][cal-722]; KOReader [L744–L770][ko-744] |
| 6 | `NOOP` (12) `{ejecting: true}` | `OK {}`, then "Disconnected by calibre" | [L627–L633][ko-627], [L377][ko-377] |

### How KOReader validates `SEND_BOOK`

KOReader refuses the book in each of these cases:

- `lpath` isn't relative, contains `..`, a backslash, a drive letter or NUL, or has an unsupported extension ([`isSafeLpath` L101][ko-101]).
- `metadata` isn't a JSON object ([L694][ko-694]).
- The inbox directory is missing, or the book doesn't fit with a 128 KiB margin.

If the `calibre_version` sent in step 1 is at least 4.18.0, KOReader reports a refusal as an `ERROR {message}` reply ([L666][ko-666]). With an older version, it only shows the refusal on the reader's screen.

### Metadata KOReader keeps

`metadata.calibre` stores only `uuid`, `lpath`, `last_modified`, `size`, `title`, `authors`, `author_sort`, `tags`, `series` and `series_index` ([metadata.lua L17][ko-meta-17], applied by `slim` [L41][ko-meta-41] in `addBook` [L134][ko-meta-134]).

## Not verified

- Whether skipping calibre's other post-handshake messages (`SET_LIBRARY_INFO`, `FREE_SPACE`, `GET_BOOK_COUNT`, and so on) leaves KOReader in a clean state.
- Which of the kept metadata fields KOReader's calibre search actually needs.
- The password challenge/response format.
- Behaviour when `wantsSendOkToSendbook` is false.

The suggested way to settle these: record a real calibre ↔ KOReader session with "Print extra debug information" turned on, and use it for golden test fixtures.

[ko]: https://github.com/koreader/koreader/tree/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin
[cal]: https://github.com/kovidgoyal/calibre/tree/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app
[ko-31]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L31
[ko-33]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L33
[ko-35]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L35-L58
[ko-101]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L101
[ko-150]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L150-L170
[ko-164]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L164
[ko-285]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L285
[ko-330]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L326-L335
[ko-377]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L377
[ko-432]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L432-L445
[ko-486]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L486-L490
[ko-496]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L496-L539
[ko-580]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L580-L591
[ko-593]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L593-L597
[ko-627]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L627-L633
[ko-658]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L658-L783
[ko-666]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L666
[ko-694]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L694
[ko-744]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L744-L770
[ko-meta-17]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/metadata.lua#L17-L28
[ko-meta-41]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/metadata.lua#L41
[ko-meta-134]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/metadata.lua#L134
[cal-128]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L128-L133
[cal-249]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L249
[cal-251]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L251
[cal-645]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L645
[cal-702]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L702-L711
[cal-722]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L722-L729
[cal-1053]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L1053-L1064
