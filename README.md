# lsp_gleam

The Gleam language server, `gleam lsp`, as a Loom language profile
([Loom's ADR-016](https://github.com/Roasbeef/loom/blob/main/docs/adr/016-language-profiles.md)). It is the maintained version of the `[lsp.gleam]` example in
[`docs/examples/loom.toml`](https://github.com/Roasbeef/loom/blob/main/docs/examples/loom.toml).

```sh
loomd ext install https://github.com/Roasbeef/loom-lsp-gleam --rev v0.1.0
loomd ext check lsp_gleam
```

The host needs `gleam` on the daemon's `PATH` (or code mode's located
toolchain, which a bare `gleam` means in a session) and `rg`, which a
bare-name question searches the project with. The server writes
`manifest.toml` and `build/` into the project, so the project is
writable; it needs nothing outside it.

`fixture/` is a two-module project, and the two `[[check]]`s in
`extension.toml` are a qualified definition (`util.greet`) and the
references across both modules. A `loom.toml` table named `gleam`
replaces this profile whole.

[docs/how-this-profile-works.md](docs/how-this-profile-works.md) walks
through `extension.toml` key by key, what the checks prove, and what the
CI does.

## Maintenance

This repository is the maintained `lsp_gleam` profile. Its CI
(`.github/workflows/check.yml`) builds [Loom](https://github.com/Roasbeef/loom)
at a pinned revision, installs this repository from its checkout with
`loomd ext install <checkout path>`, and runs `loomd ext check lsp_gleam`, which
runs the two `[[check]]`s against `fixture/` through the jail. The pinned
revision is `LOOM_REV` in that workflow; changing it re-proves the profile
against a newer Loom.

To use the `[lsp.gleam]` table without installing the extension, copy the
matching example from [`docs/examples/loom.toml`](https://github.com/Roasbeef/loom/blob/main/docs/examples/loom.toml) into your `loom.toml`.
