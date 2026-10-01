# How lsp_gleam works

This repository is a Loom language profile for the Gleam language server,
`gleam lsp`. It holds no code. It holds one `extension.toml` that tells
[Loom](https://github.com/Roasbeef/loom) how to run the server in its
jail, a small Gleam project in `fixture/` that the server loads, and a CI
workflow that proves the two agree. This document walks through each of
them so you can read `extension.toml` without having Loom's source open.

For the machinery behind it, see Loom's
[language-server architecture](https://github.com/Roasbeef/loom/blob/main/docs/architecture/lsp.md)
and [ADR-016, language profiles](https://github.com/Roasbeef/loom/blob/main/docs/adr/016-language-profiles.md).
The short version: Loom speaks the Language Server Protocol and knows no
language. A server runs project code (`gleam lsp` compiles the project),
so Loom runs it inside a sandbox, and everything that sandbox must grant,
plus the few facts a language spells differently, comes from a profile
like this one.

## Reading path and design rules

This is the only design document in the repository, because the profile
is one short TOML file and a split into principles and architecture would
repeat itself. Read `extension.toml` first, then `fixture/`, then
`.github/workflows/check.yml`; this document explains each in that order.
Three rules shaped the file.

### Grant what was measured, and nothing wider

The jail starts from nothing, so every grant in the table is there
because something failed without it. The server writes `manifest.toml`
and `build/` into the project it serves, so the project is writable. That
is the only grant. The reason is that a language server runs project
code, and a grant that is wider than the measurement is authority a
hostile project can use.

### Resolve the toolchain the way code mode does

The command is a bare `gleam`. Loom resolves it to the toolchain code mode
located, so the compiler analysing a project is the one that builds the
model's programs, and their versions cannot disagree. The profile states
no path, because a path would pin one machine's layout.

### Put the proof beside the claim

A profile asserts that a server loads a project and answers inside the
jail. The `[[check]]`s and the fixture are that assertion made
executable, and CI runs them against a real Loom, so a claim in
`extension.toml` that stops being true fails a build instead of a
user's session.

## extension.toml, key by key

### `[extension]`

`name = "lsp_gleam"` is what `loomd ext check` takes, and `tier =
"profile"` says this extension is data only. A profile may declare
language-server tables and checks. It may not declare tools, hooks or a
network policy, and Loom refuses the manifest if it does. `version`,
`description` and `license` are ordinary metadata.

### `[lsp.gleam]`

The table name, `gleam`, is the server's name inside Loom. A `loom.toml`
table with the same name replaces this one whole, never field by field.

`command = ["gleam", "lsp"]` is the argv Loom executes. It is never a
shell string. The head, `gleam`, is the one bare name Loom treats
specially: it resolves to the toolchain that code mode located, so the
compiler analysing a project is the one that builds the model's
programs. Where code mode located none, it is an ordinary `PATH` lookup.
Either way Loom mounts the directory holding the executable, read-only,
and nothing wider.

`extensions = [".gleam"]` claims every `.gleam` file for this server. A
file has exactly one owning server, so two profiles claiming `.gleam`
conflict and Loom refuses both. Because `language_id` is not set,
documents are opened with the `languageId` `gleam`, the default (the
first extension without its dot), which is what the server expects.

`root_markers = ["gleam.toml"]` chooses the project: the nearest ancestor
directory of a file that holds a `gleam.toml` is the project root, and it
is the root the server is started on. A file whose real location lies
outside that root is refused before any request is sent.

`project = "writable"` is the one grant this profile makes. The server
compiles the project it serves and writes `manifest.toml` and `build/`
into it, so a read-only project would fail to load. This is also why
the profile asks for nothing else. A project with no dependencies reads
nothing outside itself, and one with dependencies has them under
`build/packages`, which is inside the project. So there are no
`readable` or `writable` roots, no `cache_env` and no `env` names.

`hint` is one line, appended once to `lsp_definition`'s description in
sessions that configure this server. It tells the model to qualify a name
with its module as imported (`probe.greet`), or `pkg/mod.name` for a
nested module. `qualifier_separators` and `module_case` are not set: the
defaults, `.` and as-written, fit Gleam. The qualifier must end the
definition's file path without its extension, so `util` matches
`src/util.gleam` and a nested `pkg/mod` matches `src/pkg/mod.gleam`.

The network is off in every jail, so the server cannot fetch packages. A
project with dependencies needs them fetched before the session starts.

### `[[check]]`

A check is one question asked of a running server through the same door
Loom's tools use, and a list of the sites the answer must equal. Sites
are fixture-relative `path:line`, compared as a set, so two hits on one
line count once and order does not matter. This profile has two:

- **`definition` of `util.greet`, expecting `src/util.gleam:1`.** It
  proves that the server starts under the jail's policy and compiles the
  project (which is why `project` is writable), that Loom's bare-name
  search finds the candidate sites, and that the qualifier `util` narrows
  them to the right module.
- **`references` of `greet`, asked from `src/util.gleam` line 1, expecting
  `src/util.gleam:1`, `src/util.gleam:6` and `src/fixture.gleam:4`.** That
  is the declaration, the two calls in `twice` on line 6 of its own
  module, and the two calls on line 4 of the other. This check names a
  `path` and `line`, so it also proves the narrowed form of a query, where
  Loom finds the symbol on that line instead of searching.

## The fixture

`fixture/` is a Gleam project with no dependencies. `src/util.gleam`
declares `greet` on line 1 and `twice` on line 5, whose body calls
`greet` twice on line 6. `src/fixture.gleam` imports `util` and calls
`util.greet` twice on line 4. With no dependencies the server needs no
network. The checks assert line numbers, so editing a fixture file means
updating `expect`.

## What the CI does

`.github/workflows/check.yml` runs on pushes to `main`, pull requests and
manual dispatch. Its single job proves the profile against a real Loom:

1. **Check out two repositories.** This one goes into `profile/` and Loom
   into `loom/`, at the revision `LOOM_REV`, so the tree that is later
   installed is exactly this repository.
2. **Install the build tools.** Erlang/OTP, Gleam and rebar3 build and
   run `loomd`. Gleam is also the language toolchain here, so no separate
   server install is needed: the `gleam` that built Loom is the one the
   server runs as. Go is installed to build the sandbox helper.
3. **Prepare the runner for the sandbox.** Install `bubblewrap` (the
   namespace and mount work) and `ripgrep` (what a bare-name question
   searches with). Lift Ubuntu 24.04's AppArmor restriction on
   unprivileged user namespaces, and delegate a cgroup v2 base to the
   helper with probes that prove the delegation is real.
4. **Build Loom.** `make -C loom sandbox` builds the helper, and `make -C
   loom server-shipment` builds `loomd`, retrying Hex fetches.
5. **Install the profile and check it.** `loomd ext install` on the
   `profile/` checkout, then `loomd ext check lsp_gleam`. The job passes
   exactly when that command exits 0.

### Re-proving against a newer Loom

`LOOM_REV` in the workflow's `env` block is the Loom revision the profile
is proven against: a commit SHA on Loom's `main`, starting at the merge
that landed language-server support (loom#680). To re-prove the
profile against a newer Loom, change that one value, push, and read the
`check lsp_gleam` job. A failure prints a `FAIL` line naming the expected
and the actual sites. To do the same by hand, build `loomd` from that
Loom and run the last step's two commands.
