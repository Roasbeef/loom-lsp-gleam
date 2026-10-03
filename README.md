# lsp_gleam

The Gleam language server, `gleam lsp`, as a Loom language profile
([Loom's ADR-016](https://github.com/Roasbeef/loom/blob/main/docs/adr/016-language-profiles.md)). It is the maintained version of the `[lsp.gleam]` example in
[`docs/examples/loom.toml`](https://github.com/Roasbeef/loom/blob/main/docs/examples/loom.toml).

```sh
loomd ext install https://github.com/Roasbeef/loom-lsp-gleam --rev codex/dependency-preparation
loomd ext check lsp_gleam
```

This 0.2.0 update requires Loom with protocol 064 (the dependency-preparation
PR). The branch above is the review version; pin its reviewed commit when
installing before a release tag exists. Older Loom versions refuse the new
profile key rather than ignoring its authority.

The host needs `gleam` on the daemon's `PATH` (or code mode's located
toolchain, which a bare `gleam` means in a session). Bare-name questions also
require `rg`; the shipped checks supply a path and line and need no search. The server writes
`manifest.toml` and `build/` into the project, so the project is
writable. Before each cold start, Loom runs the same Gleam executable with
`deps download` in that package. Installation explicitly approves full network
access for this setup call, bounded by 60 seconds wall and CPU time and 1 MiB
per output stream. The server itself stays offline.

Setup and LSP share a private HOME and cache. A fresh worktree requires no
manual dependency command. Workspace-local sibling packages remain readable
only within the session's permissions. Changes to their configurations or the
selected package's dependency records restart the server and rerun setup.
Registry or dependency failures refuse startup with an actionable error; an
offline server is never asked to finish the failed download.

The previous v0.1.0 install keeps its existing authority. Update explicitly,
then start a new Loom session. `loomd ext install` does not overwrite an existing
name, so remove `lsp_gleam` before installing the reviewed update.

`fixture/` is a two-module project, and the two `[[check]]`s in
`extension.toml` are a definition at `src/util.gleam:1` and the
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
