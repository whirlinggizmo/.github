# Conventions

Rules shared by the `wg*` projects. Each repo's `AGENTS.md` states the ones that apply
there, because that is the file people and agents actually read; this is the canonical
text and the reasoning behind it.

## Repos and names

Each level of a name has its own rule, and `lib` belongs to exactly one of them.

| level | rule | example |
|---|---|---|
| library repo | `wg<name>-<lang>` | `wgutils-c`, `wgrender-c` |
| bindings repo | `wg<name>-<lang>` | `wgutils-hx` |
| product / app repo | `wg-<name>` | `wg-renderer`, `wg-vf` |
| project, in prose | `<name>` | wgrender |
| artifact | `lib<name>.a` | `libwgrender.a` |
| public symbols | `wg<initial>_` / `WG<INITIAL>_` | `wgr_model_create`, `WGR_COLOR_RED` |
| internal symbols | `wg<initial>i_` / `WG<INITIAL>I_` | `wgri_model_flush` |
| tooling env vars | the project's name | `WGRENDER_WEB_PROFILE` |

- **`lib` is a linker convention, not part of a name.** `-lwgrender` resolves to
  `libwgrender.a`, which is the whole of the prefix's job. A repo doesn't need it (the
  `-<lang>` suffix already says "the library"), and a Haxe binding isn't a `lib` in any
  linker's sense. On Windows the native spelling is `wgrender.dll` + `wgrender.lib`;
  MinGW keeps the Unix form because it uses `ld`.
- **Prefixes are per library, never a shared `wg_`.** More than one of these libraries
  owns a logger, an event bus and a filesystem; one prefix would collide at link time.
  Take `wg` plus the name's first letter: `wgu_` (utils), `wgr_` (render), `wgn_` (net).
- **Public and internal differ by one letter**, so a call site reads as what it is
  without looking anything up, and a script can enforce the split. Promoting a symbol is
  a rename — that's the contract change made visible, not an inconvenience.
- **Build flags are the exception:** a flag the build system passes (`-DWGR_HEADLESS`)
  must be spelled the same in the `#ifdef`, so it keeps the public prefix wherever it
  appears.
- **A variable naming another project takes that project's name** — `LIBRL_DIR`,
  because librl is what librl is called.
- **Distinguish in prose only when it's ambiguous:** "libwgrender" where `wg-renderer`
  could be meant, plain "wgrender" otherwise. This is the `curl` / `libcurl` case.

## What belongs in which repo

- **Split by dependency weight, not by topic.** A module with no system dependencies
  (path, uri, logger, json, lru_cache, event, fileio) should be vendorable as drop-in
  `.c`/`.h` pairs. A module that drags in libcurl, TLS or platform SDKs stays a built
  library, because those link requirements have to live somewhere honest. That — not
  "networking is conceptually separate" — is why `fetch_url` doesn't belong in a
  utilities library: linking utils shouldn't pull in libcurl.
- **Core libraries stay plain C.** Scripting hosts, language bindings and networking
  beyond asset downloads are separate repos built on the public API.
- **A library doesn't depend on a sibling to get a system dependency.** It exposes a
  hook and lets the application wire one in.

## Ownership

Library repos live in the **whirlinggizmo** org. `librl` and `librl-hx` stay under
**robknopf**: librl is the raylib-backed predecessor, kept as a parity baseline and on
maintenance only.

## Vendored dependencies

- Vendor whole directories, updated together by a `tools/update_<dep>.sh` script that
  records the upstream commit (and the fork's, if we carry fixes) in a `VERSION` file
  beside the files.
- Fixes we need go on a fork, each on its own branch merged into the fork's `main`,
  never into upstream's history.
- A modified copy keeps its licence's attribution accurate: if the licence asks that a
  changed version say so (zlib does), mark each change and name the project doing the
  modifying — and keep that name correct when the project is renamed.

## Public API shape

Where a library is meant to be bound from other languages, keep the public surface
language-agnostic: handles, integers, floats, enums and `const char *`, and no other
pointers. Enforce it with a script rather than trusting convention.
