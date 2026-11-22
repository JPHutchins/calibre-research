# calibre-research

> [!WARNING]
> **LLM Disclosure**
>
> This post was authored by claude-opus-5-5 on behalf of @JPHutchins. It was prompted by a calibre ↔ KOReader wireless connection error. The request was to diagnose the error, survey CLI alternatives to the calibre GUI, and publish the findings with citations to pinned upstream sources.

These are research notes for a planned tool that **pushes an ebook to KOReader by playing calibre's side of the wireless device protocol**.

- KOReader's built-in "Connect to calibre" client does the receiving, so its inbox, calibre metadata and search all keep working.
- Nothing extra needs installing or enabling on the reader, and the calibre GUI isn't needed.
- No code yet.

## Findings

1. **calibre's "Device responded with incorrect information" really means KOReader hung up.** calibre read zero bytes while waiting for the reply to `GET_DEVICE_INFORMATION`. → [01-handshake-failure.md](docs/research/01-handshake-failure.md)
2. **KOReader gives calibre only 1 s between the first two handshake messages.** calibre sends the second one only after its Qt GUI thread has finished `device_detected`, including any plugin hooks. Why it keeps failing here is not yet confirmed. → [01 §2–§5](docs/research/01-handshake-failure.md#2-why-koreader-closes-the-connection-there)
3. **No existing CLI meets the requirement.** `ebook-device` can't do wireless. OPDS and cloud storage are pull-only. SSH skips calibre metadata. No third-party calibre-protocol server verified with KOReader was found. → [02-transfer-options.md](docs/research/02-transfer-options.md)
4. **Reading the source suggests a minimum session of six steps, not yet tested on the wire.** After UDP discovery, the server sends:
   1. initialization info
   2. device information request
   3. calibre device info
   4. `SEND_BOOK`
   5. the raw book bytes
   6. an eject `NOOP`

   → [03-wire-protocol.md](docs/research/03-wire-protocol.md)

## Planned shape

- **Library:** the protocol codec and session logic.
- **CLI:** for example, serve a file, wait for KOReader to connect, send, eject, exit.
- **Windows integration:** right-click an `.epub` and choose "serve to reader". A console window shows progress until KOReader connects and the transfer finishes, then closes.

### Open questions (not verified)

- **How to register the right-click entry.** Microsoft documents the command form of shell verbs (`"My Program.exe" "%1"`, [Verbs and File Associations][ms-fa-verbs]). The Microsoft pages reviewed did not confirm three things:
  - per-extension `SystemFileAssociations` registration,
  - how Send To passes several selected files,
  - what Windows 11 needs to show the entry in its top-level context menu.
- **Firewall.** The listener needs inbound TCP plus the UDP discovery ports allowed through Windows Firewall.
- **WSL2.** The listener has to run natively on Windows, because inbound LAN traffic doesn't reach WSL2 by default.

## Sources

All code citations are permalinks pinned to these commits:

| Project | Ref | Commit | License |
|---|---|---|---|
| [calibre](https://github.com/kovidgoyal/calibre) | `v9.6.0` | [`d022762`](https://github.com/kovidgoyal/calibre/tree/d022762b4635598ce76cae248b68a27b091d3805) | GPL-3.0 |
| [KOReader](https://github.com/koreader/koreader) | `master` on 2026-11-21 | [`1294bb3`](https://github.com/koreader/koreader/tree/1294bb3d377c51fdd0b7ef250620aee2ca08927c) | AGPL-3.0 |
| KOReader | `v2026.07.1` (latest stable) | [`9192014`](https://github.com/koreader/koreader/tree/9192014d8bd82a91dc1012473be0f238dedfdb54) | AGPL-3.0 |
| [papyrix-reader](https://github.com/bigbag/papyrix-reader) | `main` on 2026-11-21 | [`12b7e6b`](https://github.com/bigbag/papyrix-reader/tree/12b7e6b28fe9a970a709851117b08096ee6ea941) | MIT |

Related KOReader threads: [#13704](https://github.com/koreader/koreader/pull/13704) (the PR that introduced the current handshake), [#16261](https://github.com/koreader/koreader/issues/16261), [#15169](https://github.com/koreader/koreader/issues/15169), [#13859](https://github.com/koreader/koreader/issues/13859).

Evidence collected locally is in [`docs/evidence/`](docs/evidence/).

## License

Licensed under either of [Apache License, Version 2.0](LICENSE-APACHE) or [MIT license](LICENSE-MIT), at your option.

Short excerpts quoted from calibre and KOReader remain under their upstream licenses, and are included only for commentary with links to the originals.

[ms-fa-verbs]: https://learn.microsoft.com/en-us/windows/win32/shell/fa-verbs
