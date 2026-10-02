# Conventions

Rules shared by the `wg*` projects. Each repo's `AGENTS.md` states the ones that apply
there, because that is the file people and agents actually read; this is the canonical
text and the reasoning behind it. The rules name no project: an example stands for any
project the rule fits.

## Repos and names

Each level of a name has its own rule. `<name>` is the project's name and `<abbr>` its
abbreviation, a letter or a few: a project `wgfoo` might take `f`, or `foo`.

| level | rule | example |
|---|---|---|
| library repo | `lib<name>` | `libwgfoo` |
| product / app repo | `wg-<name>` | `wg-foo` |
| project, in prose | `<name>`, or `lib<name>` where the app could be meant | wgfoo |
| artifact | `lib<name>.a` (`<name>.lib` from MSVC) | `libwgfoo.a` |
| public symbols | `wg<abbr>_` / `WG<ABBR>_` | `wgf_model_create`, `WGF_COLOR_RED` |
| internal symbols | `wg<abbr>_<section>_priv_` | `wgf_gfx_priv_node_of` |
| tooling env vars | the repo's name, upper-cased | `LIBWGFOO_CACHE_DIR` |

- **A library repo is named for its library: `lib<name>`.** The same name is the
  archive, the build system's project, the per-user cache folder, and the prefix of its
  tooling variables, so nothing translates between them. Bindings live in the library's
  repo (below), so a repo needs no `-<lang>` suffix to tell its languages apart. A
  binding that ever has to live apart is `lib<name>-<lang>`.
- **Repos that predate this keep their names until they are next renamed or retired**;
  GitHub redirects a renamed repo, so a rename costs little when the time comes.
- **Prefixes are per library, never a shared `wg_`.** More than one of these libraries
  owns a logger, an event bus and a filesystem; one prefix would collide at link time.
  Take `wg` plus the library's abbreviation, short and unique among the libraries.
- **Internal symbols keep the prefix and add `_priv_` after the section**
  (`wg<abbr>_<section>_priv_`), declared in a header whose name ends in `_priv.h`,
  beside the sources, never under `include/`. The stem stays whole, so searching for a
  project's prefix finds all of its code, and the section names whose internals a call
  reaches, which is what lets a checker hold a layer to its own internals and the
  others' public API. A call site reads as what it is without looking anything up, and the
  project's checker enforces the split. Promoting a symbol is a rename: the contract
  change made visible, not an inconvenience. (A marker grafted onto the prefix, such as
  `wg<abbr>i_`, changes the stem; projects that predate this keep it until they're
  renamed or retired.)
- **Build flags are the exception:** a flag the build system passes (`-DWG<ABBR>_HEADLESS`)
  must be spelled the same in the `#ifdef`, so it keeps the public prefix wherever it
  appears.
- **A variable naming another project takes that project's name** (`SOKOL_DIR`, for a
  sokol checkout), because that is what the project is called.

## What belongs in which repo

- **Split by dependency weight, not by topic.** A module with no system dependencies
  (path, uri, logger, json, lru_cache, event, fileio) should be vendorable as drop-in
  `.c`/`.h` pairs. A module that drags in libcurl, TLS or platform SDKs stays a built
  library, because those link requirements have to live somewhere honest. That, not
  "networking is conceptually separate", is why `fetch_url` doesn't belong in a
  utilities library: linking utils shouldn't pull in libcurl.
- **Core libraries stay plain C.** Language bindings live in the library's repo, under
  `bindings/<lang>/`, built only on the public API, so a public API change updates them
  in the same commit, and the library's verification runs their suites. Scripting
  hosts, and networking beyond asset downloads, are separate repos built on the public
  API.
- **A library doesn't depend on a sibling to get a system dependency.** It exposes a
  way in (a polled request, a setting) and lets the application wire one up.

## Ownership

Library and app repos live in the **whirlinggizmo** org. Under **robknopf**: forks of
third-party code we carry fixes on, mirrors of third-party tools we pin (below), and
predecessors kept as a baseline on maintenance only.

A repo that is retired is **archived**, not deleted: read-only, still cloneable and
linkable. Before archiving, its README gets a notice at the top saying it is archived,
what it was pinned to, and where to look instead, pointing only at something public. An
archived repo can't be edited, so a link to something not yet published would stay
broken. A Pages site outlives the archiving: unpublish it first if a newer copy
replaces it.

## Top-level files

At a repo's root, where people look: `README.md`, `LICENSE`, `THIRD_PARTY_NOTICES.md`
(below), `BUILDING.md` (what to install, required and optional, each build, and every
tool), and `AGENTS.md` (with `CLAUDE.md` pointing to it). Everything else, for the
people working on the code, goes in `docs/`: its architecture, its own conventions,
`BINDINGS.md` for binding authors.

## Builds

- **A build is a CMake preset named `<platform>-<variant>`.** The platform is what a
  program links against, with its architecture, and its toolchain where it has more
  than one: `linux-x64`, `macos-arm64`, `windows-x64-msvc`, `windows-x64-mingw`,
  `wasm32`. The variant is `release` or `debug`, always named, then what the build
  *adds*, in order: a backend other than the platform's default (`webgpu`), options
  (`headless`, `threads`), a sanitizer (`tsan`, `asan`, `ubsan`). **A name only ever
  adds** (`-threads`, never `-nothreads`), so a name never says what a build lacks:
  `linux-x64-debug-headless`, `wasm32-release-webgpu-threads`.
- **A native platform's presets show only on that host** (a `condition` on
  `${hostSystemName}`); a cross build shows wherever it can run.
- **What a build makes is in `out/<platform>/<variant>/`; its work in `build/<preset>/`**
  (the build system's cache, objects, generated sources, a tool's byproducts for that
  build). In `out/`: the library in `lib/`, its public headers in `include/` where it is
  staged for others, programs in `bin/`, a web build's whole site in `site/`, and in
  `share/<name>/` what ships with it (`LICENSE`, `THIRD_PARTY_NOTICES.md`, tools a
  program outside the repo runs). Deleting `out/` is a clean; deleting `build/` loses
  nothing a consumer uses.
- **A program outside the repo builds against `out/`, never `build/`**, from the same
  commit as the library, and finds a variant by its name.
- **The path names the build, not the tool:** anything that makes a given variant (the
  preset, or a script that builds the web library with emcc alone) writes the same
  `out/` directory from the same flags.
- **MSVC builds use the static C runtime** (`/MT`, `/MTd` for debug): one choice per
  variant, so a consumer never meets a runtime mismatch at link time.
- **Nothing is fetched at build time.** What a machine sets up once and every build
  shares (downloaded or built tools, a Wine prefix) goes in a per-user cache,
  `~/.cache/<name>/` (`~/Library/Caches` on macOS, `%LOCALAPPDATA%` on Windows),
  overridable by a tooling variable (`LIB<NAME>_CACHE_DIR`). It is safe to delete.

## Vendored dependencies

- **Vendor whole directories, pinned to a commit**, updated together by a
  `tools/update_<dep>.py` script (or by hand, as the repo's deps notes say) that records
  the upstream commit, and the fork's if we carry fixes, in a `VERSION` file beside the
  files. Prefer a release tag; pin upstream's main branch instead when fixes we need
  (security fixes above all) landed after the last tag, and say so in `VERSION`.
- **Fixes we need go on a fork**, each on its own branch merged into the fork's `main`,
  never into upstream's history. An upstream that refuses some kind of contribution
  (LLM-written code, say) is why a fork exists; its own `AGENTS.md` keeps upstream's
  rules for anything offered upstream.
- **An altered file says so at its top**: that it is altered, from what, by whom, and
  what changed, as zlib-style licenses require ("altered source versions must be plainly
  marked"). Make the note in the fork, so every repo that vendors it gets it. A file
  changed in place, without a fork, marks each change and names the project doing it,
  and keeps that name correct when the project is renamed.
- **`THIRD_PARTY_NOTICES.md`** at the root has every vendored license's full text,
  grouped by license, with its copyright holders (code embedded inside a dependency
  included: a JSON parser inside a glTF loader, a decoder inside a font library), which
  files are altered, and what a binary must ship. A build stages it into
  `share/<name>/`. Keep it current with the vendored code, in the same commit.
- **A prebuilt tool we download** (a shader compiler, a toolchain) is pinned by commit
  and checked against a SHA-256 per platform, written whole (a `.part` file renamed
  into place), and the cached copy is checked again before it is used. Its source is
  upstream first, then a mirror under robknopf that tags the pinned commit, so a tool
  that moves or vanishes upstream doesn't break a checkout.

## Public API shape

Where a library is meant to be bound from other languages, keep the public surface
language-agnostic, and enforce it with a tool that reads the headers through clang,
not by convention:

- **Every parameter and return** is a handle, an integer, float, bool or enum,
  `const char *` (UTF-8 text), a fixed-layout math value by value (vectors, a
  quaternion, a matrix: a type whose layout can never change), or a byte span
  (`const unsigned char *data, int size`: copied in before the call returns; out, owned
  by the handle or task that produced it). No other pointers, no records (a struct
  that could grow a field), no `void *`, no `...`.
- **No callbacks but the frame loop's**: the platform owns the loop, so it has to call
  the program. Anything else that waits (a load, a download, a task) is a handle whose
  status the program reads, and a status changes only at the start of a frame.
- **Every value a setter stores has a getter**, and a setter's comment says whether it
  clamps or refuses: it clamps when every value in range is the same request at another
  fidelity, and refuses (returns false) when the value would change what was asked or
  means nothing, with a "false for ..." sentence naming every refusal.
- **A resource is created from a path and an object from a resource handle**, a
  resource is released (reference counted) and an object destroyed.

## Bindings

A binding names things in its own language's style; what every binding keeps is the
correspondence:

1. **Each C function has exactly one public name.** Overloads of that name count as one.
2. **A public member that calls C calls one C function.** Anything that combines calls
   calls the members that wrap them, never C directly.
3. **Sugar is welcome on top**: constructors, operators, methods on a handle type. It
   reaches C only through the one name.
4. **Private plumbing is exempt**: the frame loop's trampoline, a helper polling tasks.

Then a C call's documentation, above all its refusal sentence, has one home in each
binding, a binding can be audited for coverage call by call, and a second path to C
can't skip a check the first makes. Each binding checks this on every build, against
the headers.

## Tools

- **Python 3, standard library only, the same on every OS.** No make, no shell scripts.
- **A script is `<verb>_<noun>`**: what it does, and to what (`check_`, `gen_` for
  committed generated files, `run_`, `build_`, `measure_`, `setup_`, `update_`,
  `verify_`, ...). **A module, imported and never run, is one word**, and no script
  imports another: what scripts share goes in a module.
- **Every tool answers `--help`** with its usage and nothing else done, and **refuses an
  argument it doesn't take**. A check runs every tool so, and fails a name, an import or
  a `--help` that breaks these.
- **No tool reads source code as text.** What a tool needs to know about code comes from
  something that parses it: the headers and sources through clang's AST, a shader
  through its compiler's reflection, Python through `ast`, another language through its
  own compiler. Matching a program's output, or reading data and config, is fine.
  Where nothing parses it, raise it rather than scan.
- **One command verifies everything a change touches** (every local preset, the
  bindings, the web in a browser, a real Windows machine when asked), stopping at the
  first failure, and its pass names every step it skipped for want of a tool, so a pass
  can't be read as more than it was.
