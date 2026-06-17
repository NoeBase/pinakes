# Pinakes

**Open knowledge packages for agents.** Like npm — but for knowledge, not code.

*Pinakes* /ˈpɪn.ə.kiːz/ — named after the [Pinakes](https://en.wikipedia.org/wiki/Pinakes),
the catalogue of the Library of Alexandria: the first index of the world's knowledge.

Knowledge an AI agent needs — a method, a team's conventions, a domain primer — is
**vendored into your repo as plain files**, so any agent reads it with its strongest
tools (`grep`, `read`, `glob`). Versioned, provenance-signed, kept current by its source.

> Code has git, then npm. Knowledge has no git — so Pinakes is the npm without one:
> a thin open envelope around free content, vendored where agents already grep.

## Why

Context7 and RAG keep knowledge *remote* — one lookup per call, never part of the
agent's working set. Pinakes makes it **local and native**: an agent cross-references it
against your actual code in a single `grep` — offline, diffable, reviewable as a PR.

There is no universal substrate for knowledge the way git is for code — knowledge is
irreducibly multi-representational (prose, graph, table). So Pinakes standardizes **only a
thin envelope** (manifest, versioning, provenance) and leaves the **content free**.

## How (preview)

```sh
pin add github:acme/team-decisions     # vendor straight from git
pin add team-decisions                 # …or by name, via the registry
pin update                             # keep tracked packages current
pin verify                             # check provenance + integrity
```

Packages land in `./knowledge/` as plain Markdown your agent already greps.

## Status

**Early — RFC stage.** The format is being designed in the open. The draft envelope and
the honest list of open questions live in **[SPEC.md](./SPEC.md)**. Issues, ideas, and
disagreement welcome.

## Substrate-independent

Pinakes is an **open standard**: publish from a bare Markdown folder, a git repo, or a
[NoeBase](https://noebase.com) Space. NoeBase is the first *reference host* — never the gatekeeper.

---

*By [NoeBase](https://noebase.com) — where thinking grows.* · pinakes.cc
