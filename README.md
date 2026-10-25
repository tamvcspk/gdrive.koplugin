# gdrive.koplugin

A [KOReader](https://koreader.rocks/) plugin for reading books from Google Drive on e-readers, built for small-storage, low-power devices (developed and tested on Kobo).

> **Status:** planning / early development. Nothing is released yet.

## Goals

- **Offline-first:** browse your whole Drive library from a local catalog with Wi-Fi off; the network is used only when you press *Sync*.
- **Download on demand:** fetch a book when you want to read it, or queue it for the next sync.
- **Multiple sources:** My Drive, *Shared with me* and Shared drives folders, each added by the user.
- **Storage-aware:** user-defined quota, LRU eviction, pinned books.
- **Read-only access:** uses the `drive.readonly` scope; it never modifies your Drive.
- Comics (cbz/cbr, image folders) and regular book formats supported.

See the full design in [docs/plan.md](docs/plan.md) (Vietnamese).

## Privacy

The plugin talks only to Google's APIs; nothing is sent to any other server. See [PRIVACY.md](PRIVACY.md).

## License

[MIT](LICENSE)
