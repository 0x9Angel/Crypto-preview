# Contributing to Crypto and Gotham

Thank you for your interest in contributing.

Crypto is a self-hosted encrypted messenger. Gotham is the
post-quantum mixnet protocol it rides on. Both are written in Rust
and published under AGPLv3 with a commercial dual-licensing option.

This document covers everything you need to land a change.

---

## Before you start

1. **Read the Code of Conduct** — [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md).
   It applies to every interaction in this project.
2. **Read the architecture docs** — [GOTHAM.md](GOTHAM.md) for the
   protocol, [README.md](README.md) for the application.
3. **Search existing issues and PRs** before opening a new one.
4. **For security issues** — do NOT open a public issue. See
   [SECURITY.md](SECURITY.md) for the private reporting channel.

---

## Types of contributions

We welcome:

- **Bug reports** — open a GitHub issue with a clear repro.
- **Documentation fixes** — typos, clarifications, missing sections.
- **Tests** — unit, integration, property-based, fuzz harnesses.
- **Refactoring** — provided it does not change observable behaviour.
- **New features** — but please discuss in an issue first; we may
  decline features that expand the threat surface or contradict the
  documented protocol semantics.

We do NOT accept (without prior maintainer agreement):

- Changes that weaken security properties documented in
  [GOTHAM.md](GOTHAM.md) or in [crypto-gotham/src/lib.rs](crypto-gotham/src/lib.rs).
- Changes to the wire format without a version bump and migration plan.
- Telemetry, analytics, "phone-home" code paths of any kind.
- Dependencies that are unmaintained or that introduce a transitive
  unmaintained / GPL-incompatible dependency.

---

## Development setup

### Prerequisites

- **Rust** stable (currently 1.84+) — install via `rustup`.
- **Tauri** prerequisites — see https://tauri.app/start/prerequisites/
  for your OS.
- **SQLCipher** development headers (Linux: `libsqlcipher-dev`;
  macOS: `brew install sqlcipher`).
- **Node.js** 20+ for the frontend dev server.
- **`cargo-audit`** for the security check workflow:
  `cargo install cargo-audit`.

### First build

```bash
git clone https://github.com/0x9Angel/Crypto.git
cd Crypto
cargo build --workspace
cargo test  --workspace
cd crypto-tauri && npm install && cargo tauri dev
```

The first build takes 5-10 minutes (heavy dependency tree on the
Tauri side). Subsequent incremental builds are fast.

### Workspace layout

```
crypto-proto              wire types shared across crates
crypto-agent              X3DH + Double Ratchet session manager
crypto-store              SQLCipher-backed local storage
crypto-server             federation server (optional)
crypto-cli                command-line client
crypto-tauri              desktop / mobile app (Rust + React)
crypto-enterprise         OIDC / SCIM / audit log
crypto-gotham             Gotham mixnet protocol primitives
crypto-gotham-relay       Gotham relay binary
crypto-tests              cross-crate integration tests
```

See the comments at the top of each crate's `src/lib.rs` for module
responsibilities.

---

## Branching and commit style

### Branches

- `main` — production line. Always green. Merges are squash-only.
- `gotham-v0.2` — Gotham protocol v0.2 feature work, gated on
  external audit before merging back to `main`.
- `feat/<short-name>`, `fix/<short-name>`, `docs/<short-name>` for
  individual changes.

### Commit messages

We use [Conventional Commits](https://www.conventionalcommits.org/)
with project-specific scopes:

```
<type>(<scope>): <imperative summary, ≤ 72 chars>

<optional body, wrap at 72 chars, explain WHY>

<optional footer: Refs #123 / Fixes #456 / Breaking-Change: ...>
```

Types we use: `feat`, `fix`, `perf`, `refactor`, `docs`, `test`,
`chore`, `build`, `ci`.

Scopes are crate names without the `crypto-` prefix, plus a few
cross-cutting scopes: `gotham`, `relay`, `tauri`, `store`, `server`,
`agent`, `proto`, `cli`, `enterprise`, `checklist`, `security`, `lint`.

Example commits from the project history:

```
feat(tauri): A4.3 + A4.4 — send_message + set_own_profile route via Gotham
feat(gotham): A1.8 — folded-KEM (shift-and-pad β) header construction
chore(lint): clippy clean sweep — workspace warnings 76 → 0
chore(checklist): A1.3 — constant-time HKDF sub-key expansion done
docs(security): add coordinated-disclosure policy
```

**Do NOT** add `Co-Authored-By` lines unless explicitly agreed with
the project lead. The repository policy is human-authored attribution
only.

---

## Pull request process

1. **Open an issue first** for non-trivial changes. A 1-line PR
   description is fine for typos, but feature work needs a design
   discussion.
2. **Branch from `main`** (or `gotham-v0.2` for protocol v0.2 work).
3. **Make focused commits.** One logical change per commit. A 50-line
   refactor and a 200-line feature should not share a commit.
4. **Run the full test suite** before pushing:
   ```bash
   cargo fmt --all
   cargo clippy --workspace --all-targets -- -D warnings
   cargo test --workspace
   cargo audit
   ```
5. **Update relevant docs.** If you touch a public API, the doc
   comment changes. If you add a feature, [GOTHAM-CHECKLIST.md](GOTHAM-CHECKLIST.md)
   or [CHECKLIST.md](CHECKLIST.md) gets a tick.
6. **Open the PR** with the same conventional-commit summary as your
   commits. Describe the WHY in the PR body, link the issue, and
   highlight breaking changes.
7. **Respond to review** promptly; rebase rather than merge `main`
   into your branch to keep history linear.

### CI gates

A PR cannot merge until:

- `cargo fmt --check` passes (no formatting drift).
- `cargo clippy -- -D warnings` passes (zero warnings).
- `cargo test --workspace` passes (all tests green).
- `cargo audit` passes or its findings are documented in
  [SECURITY.md](SECURITY.md).
- `cargo doc --no-deps` produces zero warnings.
- At least one maintainer approval.

---

## Coding standards

### Rust

- **`#![deny(clippy::unwrap_used)]` in production paths.** Tests may
  unwrap freely (this is enforced via `cfg_attr(not(test), deny(...))`).
- **`#![warn(missing_docs)]`** on every public crate. Every `pub` item
  needs a doc comment.
- **`zeroize` every secret-bearing struct.** Use `ZeroizeOnDrop` when
  the secret outlives a single function.
- **Constant-time crypto.** Any code path that touches secret material
  must be free of secret-dependent branches and memory accesses.
- **No `panic!` in production paths.** A panic in a relay's
  `process()` brings the whole relay down and is observable from the
  network. Return `Result` and let the caller decide.
- **One crate, one responsibility.** If a change crosses two crates,
  it likely needs to be two PRs.

### Comments

- Default to writing no comments — well-named identifiers explain
  themselves.
- Add a comment when the *why* is non-obvious: a subtle invariant, a
  workaround, a known footgun.
- Do not write comments that describe what the code does line by
  line. Do not write comments that reference the current task or
  ticket number — that goes in the commit message or PR description.

### Frontend (React + TypeScript)

- Functional components only, hooks-based.
- TypeScript strict mode on.
- Tailwind utility classes; no global CSS overrides unless documented.
- One component per file when feasible.

---

## Tests

Every PR that touches code MUST keep the test count at parity or
greater. Concretely:

- New public function → at least one unit test (or argument why a
  unit test is impossible).
- New cross-crate seam → an integration test in
  `crypto-gotham-relay/tests/` or `crypto-tests/tests/`.
- New error path → a test that exercises it.
- Fuzz-friendly parser → a fuzz target under `crypto-gotham/fuzz/`.

Run the relevant subset locally before pushing:

```bash
cargo test -p crypto-gotham
cargo test -p crypto-gotham-relay
cargo test -p crypto-tauri
cargo test -p crypto-tests
```

---

## Documentation contributions

- Markdown files at the repo root: keep the line length under 80
  characters where reasonable.
- Crate-level rustdoc on `src/lib.rs` for every public crate.
- Examples in rustdoc comments should compile (or be marked
  `# ignore` with a reason).

---

## Licensing

By contributing, you agree that your contributions will be licensed
under the project's dual licence:

- **AGPLv3** for community use, OR
- The project's commercial licence (LICENSE-COMMERCIAL) for
  commercial use.

If your contribution includes any third-party code, you must clearly
identify it and confirm that its licence is compatible (Apache-2.0,
MIT, BSD-2-Clause, BSD-3-Clause, MPL-2.0, ISC, or AGPL-compatible).

---

## Recognition

We maintain a contributors list. Substantive technical contributions
will be acknowledged in:

- The `CHANGELOG.md` entry for the release containing the change.
- The repository contributors page on GitHub.
- For security researchers, the SECURITY.md advisory crediting them
  by name (or anonymously, at their preference).

---

## Questions?

- For technical questions: open a GitHub Discussion.
- For private or sensitive matters: use the contact channels in
  [SECURITY.md](SECURITY.md).
- For Code of Conduct concerns: see
  [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md).
