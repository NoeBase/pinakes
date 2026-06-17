# Pinakes — Format Specification

**Version:** `0` (draft) · **Status:** RFC — designed in the open · **License:** Apache-2.0

> This document specifies the **Pinakes envelope**: the thin, open wrapper that turns a
> folder of knowledge into a versioned, verifiable, vendorable package. It is a **draft**.
> Sections marked **(OPEN)** are unresolved and actively up for discussion — see
> [§12 Open questions](#12-open-questions). Disagreement is the point; open an issue.

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are to be interpreted as in
[RFC 2119](https://www.rfc-editor.org/rfc/rfc2119), but only loosely while this is a draft.

---

## 1. Motivation

Code has a universal substrate (`git`) and a distribution layer on top of it (`npm`,
`cargo`, …). Knowledge has neither. The knowledge an agent needs — a method, a team's
conventions, a domain primer — lives in wikis, PDFs, RAG indexes, and people's heads. None
of it is *vendored where the agent already works*.

Pinakes does **not** try to be "git for knowledge." Knowledge is irreducibly
multi-representational (prose, graph, table); there is no single diffable substrate that
fits all of it. So Pinakes standardizes **only a thin envelope** — identity, versioning,
provenance, integrity — and leaves the **content free**. The content is read by the agent
with its strongest, most native tools: `grep`, `read`, `glob`.

Design goals, in priority order:

1. **Local & native.** A package is plain files in the consumer's repo. No runtime, no
   server round-trip to use it.
2. **Verifiable.** Every file is integrity-hashed; every package records where it came from.
3. **Substrate-independent.** Publishable from a bare folder, a git repo, or a hosted Space.
4. **Minimal.** The envelope adds as little structure as possible over the raw content.

---

## 2. Terminology

- **Knowledge package** (or *package*): a directory containing a manifest
  (`pinakes.json`) and one or more content files.
- **Envelope:** the metadata the manifest carries — everything Pinakes standardizes. The
  content inside is *not* part of the envelope.
- **Source:** the origin a package is resolved and fetched from (a git repo, a path, a URL,
  or a registry name).
- **Vendoring:** copying a resolved package into the consumer repo as plain files (as
  opposed to referencing it remotely).
- **Consumer:** the repository that vendors packages, conventionally into `./knowledge/`.
- **Registry:** a service mapping a human name to a source (§6.2). Anyone may host one.
- **Reference host:** an implementation of a registry + package host. NoeBase runs the
  first one; it is never required.

---

## 3. Lifecycle overview

```
 publish            resolve              vendor            verify / update
┌────────┐        ┌──────────┐        ┌────────────┐      ┌──────────────┐
│ source │ ─────▶ │  name →  │ ─────▶ │ ./knowledge │ ──▶ │ re-hash, diff │
│ + meta │        │  source  │        │  /<pkg>/    │      │ provenance    │
└────────┘        └──────────┘        └────────────┘      └──────────────┘
```

1. A **publisher** adds a `pinakes.json` to a folder of knowledge (§4).
2. A **consumer** runs `pin add <source>`; the source is **resolved** (§6) and the package
   is **vendored** into `./knowledge/<pkg>/` (§5).
3. The CLI records what it pulled in a **lockfile** (§5.2) and can later **verify** (re-hash
   + check provenance) and **update** (re-resolve to a newer version).

---

## 4. The knowledge package

### 4.1 Layout

A package is a directory. It MUST contain a `pinakes.json` manifest at its root. All other
files are content, in any layout the publisher chooses.

```
team-decisions/
├─ pinakes.json        # the envelope (required)
├─ conventions.md
├─ decisions.md
└─ graph.json
```

### 4.2 The manifest — `pinakes.json`

```jsonc
{
  "pinakes": "0",                       // spec version this manifest targets (required)
  "name": "team-decisions",             // package identity (required)
  "version": "2026.06.0",               // see §7 — versioning is OPEN (required)
  "description": "How our team makes and records architectural decisions.",
  "license": "CC-BY-4.0",               // license of the CONTENT (SHOULD be set)

  "source": {                           // where this package authoritatively lives
    "type": "git",                      // "git" | "path" | "url"  (OPEN: more types)
    "url": "https://github.com/acme/team-decisions",
    "ref": "a1b2c3d4",                  // commit/tag pinned at publish time
    "subpath": "."
  },

  "representations": ["prose", "graph"],// hint to agents/tools; informative (OPEN, §4.4)

  "provenance": {                       // see §8
    "published_by": "acme",
    "published_at": "2026-06-04T10:00:00Z",
    "method": "git-commit",             // "git-commit" | "sigstore" | "minisign" | "none"
    "signature": null                   // OPEN — signing not yet specified
  },

  "contents": [                         // every shipped file, with integrity (required)
    { "path": "conventions.md", "media": "text/markdown", "integrity": "sha256-…" },
    { "path": "decisions.md",   "media": "text/markdown", "integrity": "sha256-…" },
    { "path": "graph.json",     "media": "application/json", "integrity": "sha256-…" }
  ]
}
```

**Field notes**

- `pinakes` — the spec version. This document defines version `"0"`. Consumers MUST refuse a
  manifest whose major version they do not understand.
- `name` — MUST match `^[a-z0-9][a-z0-9._-]*$`. Uniqueness is only meaningful **within a
  registry** (§6.2); a bare git source needs no global name.
- `version` — REQUIRED, but its **semantics are OPEN** (§7). Treat it as an opaque,
  orderable label for now.
- `source` — records the package's authoritative origin so a vendored copy can be traced
  back and re-fetched. The `ref` SHOULD pin an immutable point (a commit SHA, not a branch).
- `contents` — the integrity manifest. `pinakes.json` itself is **not** listed in
  `contents`. Every other file in the package SHOULD be listed; a consumer MAY warn on
  files present on disk but absent from `contents` (possible tampering).

### 4.3 Content entries & integrity

Each entry in `contents` has:

- `path` — POSIX relative path from the package root. MUST NOT escape the root (no `..`,
  no absolute paths, no symlinks out).
- `media` — an [IANA media type](https://www.iana.org/assignments/media-types) hint.
  Informative; agents may ignore it and just `grep`.
- `integrity` — a [Subresource-Integrity](https://www.w3.org/TR/SRI/)-style string:
  `"<algo>-<base64(digest)>"`. v0 implementations MUST support `sha256`. The digest is over
  the **raw file bytes**.

### 4.4 Representations *(OPEN)*

The thesis is that knowledge is multi-representational. `representations` is a free-form
hint array (e.g. `"prose"`, `"graph"`, `"table"`) so tools can route. Whether this should be
a controlled vocabulary, per-file rather than per-package, or dropped entirely, is **open**.

---

## 5. Vendoring into a consumer repo

### 5.1 `./knowledge/` layout

By convention, vendored packages land under `./knowledge/` in the consumer repo, one
directory per package:

```
your-repo/
├─ src/…
├─ knowledge/
│  ├─ team-decisions/        # ← a vendored package, verbatim
│  │  ├─ pinakes.json
│  │  ├─ conventions.md
│  │  └─ decisions.md
│  └─ domain-primer/
│     └─ glossary.md
└─ package.json
```

The directory name MAY differ from the package `name` (a consumer can vendor the same
package under an alias). The lockfile (§5.2) is the source of truth for what is installed.

### 5.2 The lockfile — `knowledge/.pinakes-lock.json` *(OPEN: name/location)*

Written by the CLI, committed to the consumer repo. It records, for each installed package:
the requested source, the resolved version + ref, the directory, and a top-level integrity
digest covering the package's `contents`.

```jsonc
{
  "pinakes": "0",
  "packages": {
    "team-decisions": {
      "requested": "github:acme/team-decisions",
      "version": "2026.06.0",
      "resolved": { "type": "git", "url": "https://github.com/acme/team-decisions", "ref": "a1b2c3d4" },
      "dir": "knowledge/team-decisions",
      "integrity": "sha256-…"        // digest over the sorted contents manifest
    }
  }
}
```

`pin verify` re-hashes the on-disk files and compares against the lockfile; `pin update`
re-resolves each `requested` source to a newer version and rewrites the files + lock.

---

## 6. Sources & resolution

### 6.1 Direct sources

A source string MAY be given directly, needing no registry:

- `github:acme/team-decisions` — shorthand for the repo's default branch.
- `github:acme/team-decisions/sub/dir#v2` — subpath `sub/dir` at ref `v2`.
- `git+https://example.com/x.git#<ref>`
- `path:../local/folder` — vendor from the local filesystem.
- `https://…/pkg.tar.gz` — a packaged tarball *(OPEN: tarball format/signing)*.

### 6.2 Registry resolution *(OPEN)*

A bare `name` (e.g. `pin add team-decisions`) is resolved via a **registry**: a mapping from
`name` → source. A registry is just an HTTP endpoint returning that mapping; anyone can host
one, and a consumer MAY configure multiple. The wire protocol, name-collision rules across
registries, and trust model are **open**. NoeBase will run the first reference registry.

---

## 7. Versioning *(OPEN — the hard one)*

Semver encodes *behavioral* compatibility, which does not map cleanly onto prose. What does
a "breaking change" to a domain primer mean? Candidate models on the table:

- **CalVer** (`YYYY.MM.MICRO`) — fits "kept current by its source"; used in examples here as
  a placeholder default, **not** a decision.
- **Semver-of-meaning** — major = the claims changed, minor = additions, patch = wording.
- **Content-addressed only** — no human version; the integrity digest *is* the version.

No model is chosen. This is the single most important open question.

---

## 8. Provenance & integrity

Two distinct guarantees:

- **Integrity** (specified, v0): every file carries a `sha256` digest (§4.3); the lockfile
  pins them. This detects corruption and post-vendor tampering. It does **not** prove *who*
  authored the content.
- **Provenance / authenticity** *(OPEN)*: proving a package genuinely comes from its claimed
  publisher. v0 offers only weak provenance — a pinned git commit (`method: "git-commit"`),
  which is tamper-evident but not signed. Stronger options under discussion:
  [Sigstore](https://www.sigstore.dev/) keyless signing, `minisign`, or SSH-sig. The
  `provenance.signature` field is reserved but its format is undefined.

---

## 9. The `pin` CLI *(informative — not part of the format)*

The format does not require any specific tool. The reference CLI surface:

```sh
pin add github:acme/team-decisions   # resolve + vendor into ./knowledge/
pin add team-decisions               # …or by name, via a registry (§6.2)
pin update [name]                    # re-resolve tracked packages to newer versions
pin verify                           # re-hash on-disk files; check provenance
pin list                             # show installed packages from the lockfile
```

A package and a consumer that follow §4–§8 are conformant regardless of which tool produced
them.

---

## 10. Security considerations

- **Vendored content is executed by your agent's reasoning, not your CPU** — but it can
  still carry prompt-injection. Treat a freshly added package as untrusted input; review the
  diff like any dependency PR. Integrity hashes prove *what* you got, not that it is *safe*.
- Path entries MUST be sandboxed to the package root (§4.3) to prevent writes outside
  `./knowledge/<pkg>/` during vendoring.
- Registry resolution (§6.2) is a trust boundary; a malicious registry can point a name at a
  hostile source. Name → source mappings SHOULD be reviewable.

---

## 11. Non-goals

- **Not** a runtime, server, or query engine. Pinakes produces files; your agent does the
  rest with `grep`/`read`.
- **Not** a content standard. The envelope never dictates how knowledge is written.
- **Not** a single canonical representation ("git for knowledge"). See §1.
- **Not** a gatekept registry. The format works with zero hosted infrastructure.

---

## 12. Open questions

1. **Versioning semantics** (§7) — CalVer vs semver-of-meaning vs content-addressed.
2. **Unit taxonomy** (§2, §4) — is the package always a whole folder? Are sub-units (a single
   file, a single claim) independently addressable?
3. **Provenance & signing** (§8) — which signing scheme, and is it mandatory for registries?
4. **Registry protocol** (§6.2) — wire format, multi-registry precedence, name collisions.
5. **Representations** (§4.4) — controlled vocabulary, per-file vs per-package, or drop it?
6. **Lockfile shape & location** (§5.2) — one lock, per-package, or none (rely on manifests)?
7. **Tarball / non-git sources** (§6.1) — packaging + integrity for sourceless distribution.
8. **Dependencies** — may a package depend on another package? If so, how is that resolved?
9. **Update semantics** — what does `pin update` do when content (not just version) drifts;
   how are local edits to vendored files surfaced/preserved?

Have an opinion on any of these? → <https://github.com/NoeBase/pinakes/issues>

---

## Appendix A — a complete minimal package

`team-decisions/pinakes.json`:

```json
{
  "pinakes": "0",
  "name": "team-decisions",
  "version": "2026.06.0",
  "description": "How our team makes and records architectural decisions.",
  "license": "CC-BY-4.0",
  "source": {
    "type": "git",
    "url": "https://github.com/acme/team-decisions",
    "ref": "a1b2c3d4",
    "subpath": "."
  },
  "provenance": {
    "published_by": "acme",
    "published_at": "2026-06-04T10:00:00Z",
    "method": "git-commit",
    "signature": null
  },
  "contents": [
    { "path": "conventions.md", "media": "text/markdown", "integrity": "sha256-9f86d0…" },
    { "path": "decisions.md",   "media": "text/markdown", "integrity": "sha256-2c2640…" }
  ]
}
```

`team-decisions/conventions.md`:

```markdown
# Conventions

- Never deploy on Friday. See incident #44.
- Every architectural decision gets an ADR in `decisions.md`.
```
