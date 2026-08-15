# app-accounts

**A superseded scaffold.** This repository holds a five-file Cloudflare Worker
skeleton that was going to serve `accounts.etzhayyim.com` — account lifecycle
management (linked auth methods, provider link/unlink, `/manage` UI) split off
an auth Worker under ADR-0024 Step 3. It never shipped. The handler returns
`501`, `wrangler.jsonc` is renamed `.disabled` so no deploy can reach it, and
**the hostname it was written for no longer exists in DNS.**

Nothing here is wired to anything. Read this file before you read `CLAUDE.md`
or `MIGRATION-TODO.md`, both of which describe a topology that has since been
dismantled around them.

> **Looking for itonami account management?** It is not here. Despite living in
> the `cloud-itonami` org, every line of this repository is about
> `etzhayyim.com`. The live itonami surface is `auth.itonami.cloud`
> ([`cloud-itonami/app-auth`](https://github.com/cloud-itonami/app-auth)), with
> enrolment at `itonami.cloud/signin/`.

## What changed underneath it

Measured 2026-08-16. Every row is reproducible from
[`docs/operator-quickstart.md`](docs/operator-quickstart.md).

| What this repo asserts | What is true now |
|---|---|
| `CLAUDE.md`: "auth Worker の現行 route が `accounts.etzhayyim.com/*` を serve し続ける" | `accounts.etzhayyim.com` is **NXDOMAIN**. Not a stale route — no DNS record at all. The zone itself is healthy (Cloudflare NS, apex answers `200`), so this is one missing label, not an outage. |
| `wrangler.jsonc.disabled` blocks deploy until the route is peeled off the auth Worker | There is no route to peel off. `authn.etzhayyim.com` still answers `200`, but `/api/accounts/session` and `/xrpc/com.etzhayyim.auth.linkEmailBegin` both `301` to `auth.gftd.ai`, where `/api/accounts/session` is `404`. |
| `MIGRATION-TODO.md`: seed awaiting a Stripe/fiat → USDC codemod | The codemod scan recorded in that same file found none of the patterns it was written to remove. The blocker was never the codemod. |
| ADR-0024: split `accounts.etzhayyim.com` off `60-apps/etzhayyim-project-auth` | `etzhayyim/root@main:60-apps` now contains exactly one entry, `etzhayyim-project-organism`. The Worker this was to split from is gone from the default branch. |

## Where the responsibility went

The sibling extraction — `etzhayyim-project-auth` → `cloud-itonami/app-auth` —
faced the same dead domains and was **replaced rather than ported**
(ADR-2608110100, accepted; it measured the same NXDOMAIN on 2026-08-11). The
replacement is a ClojureScript Worker on `auth.itonami.cloud`. Its
`src/itonami/auth/worker.cljs` routes a `method-unlink` POST into
`federated/unlink!`, and `src/itonami/auth/federated.cljs` carries the OAuth
provider table — `:google`, `:github`, `:microsoft`, `:apple`. Its own README
describes the surface as attach / list / detach of a verified route on a
passkey DID.

That is the responsibility set this scaffold was created to receive. **It
already exists somewhere else, under a different domain, in a different
language.**

ADR-2608110100 never names `app-accounts` (zero occurrences). The sibling was
decided; this repository was left where it stood.

## What is actually in here

Five tracked files, 8,319 bytes, extracted verbatim from
`etzhayyim/root@691c245d:60-apps/etzhayyim-project-accounts` — a commit whose
message is `refactor: drain metadata-only app shells (#3243)`.

The provenance in `migration.edn` is **intact and verified byte-for-byte**
(2026-08-16): the declared `:git-tree` `77bb60eb…` is the real upstream tree
sha for that path at that revision, and all five blobs still hash to what
upstream holds. Nothing drifted. The extraction was done correctly — what it
extracted was already a shell.

| File | Role |
|---|---|
| `worker/src-cljs/index.cljs` | The whole implementation: a `fetch` handler returning `501`. |
| `worker/wrangler.jsonc.disabled` | Deploy config, deliberately un-findable by `wrangler`. `database_id` is the literal string `TBD-create-via-wrangler-d1-create`. |
| `CLAUDE.md` | The ADR-0024 split plan. Describes the pre-2026-07 topology. |
| `MIGRATION-TODO.md` | Charter-compliance checklist, all boxes unchecked. |
| `NOTICE` | Apache-2.0 + etzhayyim Charter Rider v3.1. |

`README.edn` and `migration.edn` are machine-readable records added by the
extraction tooling, not part of the five.

**There is a byte-identical second copy** at
`etzhayyim/com-etzhayyim-app-accounts` — same source revision, same tree sha,
all five blobs equal, differing only in `migration.edn`'s `:destination`. Two
repositories, one dead scaffold.

The two are not registered the same way. This repository is in
`manifest/west.yml` as `app-accounts`; the etzhayyim copy is **not in the
manifest at all**, though the GitHub repository is public and un-archived and a
checkout of it sits at `orgs/etzhayyim/com-etzhayyim-app-accounts`. Anyone
acting on the duplicate should treat it as an unregistered orphan checkout and
route it through the `git-cleanup-conflict` runbook rather than assuming west
will account for it.

## What to do with it

This is an owner decision, not something a maintenance pass should take
unilaterally. The options, and what each costs:

1. **Retire both copies.** The responsibility is served elsewhere and the
   domain is gone. Retiring clears one west entry plus one unregistered orphan,
   and stops this shell from surfacing in "least mature repository" rankings,
   where it scores low for the honest reason that there is nothing in it.
2. **Repoint at `itonami.cloud`.** Only if account management is wanted as a
   Worker separate from `app-auth` — but `app-auth` already holds attach /
   detach, so this would give one custody two implementations, which is
   precisely what ADR-2608110100 declined to do.
3. **Leave it.** Costs nothing but keeps a shell that reads, to anyone who
   opens `CLAUDE.md` first, as active planned work.

Until one is chosen, this file is the correction: **do not treat `CLAUDE.md` or
`MIGRATION-TODO.md` here as a live plan.**
