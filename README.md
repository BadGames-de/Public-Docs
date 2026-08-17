# BadGames — Public Docs

Public developer documentation for the integration points our servers expose: plugin message
channels you can receive from a client mod, HTTP APIs, and the data shapes that go with them.

If you are writing a mod, a companion app, or an overlay against a BadGames server, the specs
you need are here. Everything in this repository is meant for third-party use — nothing
documented here is internal-only.

---

## Contents

| Document | What it covers |
|---|---|
| [`game-share-channel.md`](game-share-channel.md) | `badgames:game_share` — server → client plugin message channel reporting map, round start/end and lobby vote results. Includes the wire format and a Fabric receiver example. |

More documents are added as features become public. There is no separate index to keep in sync
— this table is it.

## Planned

Documents that do not exist yet. Listed so you know what is coming and what to ask about, not
as a promise of a date.

- **REST API** — public HTTP endpoints, authentication, rate limits.
- **Further plugin message channels** — as they are opened up for mod use.

---

## How to read these specs

**Every document describes the live behaviour of the current server build.** If something in a
spec disagrees with what a server actually sends, the server is right and the document is a bug
— please report it.

**Specs are written to be decoded defensively.** Where a document says a field is optional, a
type may be unknown, or unrecognised keys must be ignored, that is not boilerplate: it is how
we make additive changes without breaking your build. A decoder that follows those notes keeps
working across releases; a strict one will not.

**Conventions used throughout:**

| | |
|---|---|
| Channel names | Namespaced `badgames:<name>`, lower snake case |
| Byte order | Big-endian unless a document says otherwise |
| JSON | `null`-valued fields are omitted rather than emitted as `null` |
| Numeric enums | Additive — new values get new numbers, existing numbers are never reused or renumbered |

## Compatibility

We treat these interfaces as public contracts and change them additively:

- New message types, endpoints and fields **will** appear over time.
- Existing type tags, field names and their meanings are not renumbered or repurposed.
- If a breaking change is genuinely unavoidable, it ships under a new channel name or a new API
  version rather than mutating the old one.

There is no version negotiation on the plugin message channels. Servers may run older builds
than the newest document describes, so treat a missing field as "this server is older" rather
than an error.

## Questions, corrections, additions

Open an issue on this repository for anything spec-related: a payload that does not match its
document, a field whose behaviour is unclear, or an integration point you would like documented.
Pull requests fixing wording, examples or errors are welcome.

Please include the server and the affected channel or endpoint when reporting a mismatch, plus
a hex dump or raw body if you have one — it makes the difference between a guess and a fix.
