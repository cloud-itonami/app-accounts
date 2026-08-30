# Operator quickstart

There is no service to operate here. This repository is a superseded scaffold
(see [`../README.md`](../README.md)), so the only operational task it has is
**confirming that it is still superseded** — because that claim rests on
measurements of the outside world, and those go stale.

Six checks, about a minute. Every command below was run on 2026-08-16 and the
recorded output is what it printed. Run them from the repository root.

---

## 1. Is the hostname still gone?

This is the load-bearing fact. If it comes back, everything in `CLAUDE.md`
becomes worth re-reading.

```bash
dig accounts.etzhayyim.com +noall +comments | grep -i status:
```

```
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 10331
```

`NXDOMAIN` means the label does not exist — not that it exists and points
nowhere. **If this ever prints `NOERROR`, stop and re-read `CLAUDE.md`.**

Only `status:` is stable; the `id:` is a fresh random number every query.

## 2. Is that a missing label or a dead zone?

A dead zone would mean "etzhayyim is offline", which is a different situation
with a different response. Check the apex and the sibling host:

```bash
dig +short NS etzhayyim.com | head -2
curl -sS -o /dev/null -w 'apex=%{http_code}\n'  --max-time 12 https://etzhayyim.com/
curl -sS -o /dev/null -w 'authn=%{http_code}\n' --max-time 12 https://authn.etzhayyim.com/
```

```
everton.ns.cloudflare.com.
vivienne.ns.cloudflare.com.
apex=200
authn=200
```

The zone is healthy and served by Cloudflare. Exactly one label is missing —
the one this repository was written for.

The two nameservers come back in either order; both names are what matter.

## 3. Where did the account routes go?

`CLAUDE.md` lists the paths this Worker was to take over. They still answer on
`authn`, but not from etzhayyim:

```bash
for p in /api/accounts/session /xrpc/com.etzhayyim.auth.linkEmailBegin; do
  printf '%-45s -> ' "$p"
  curl -sS -o /dev/null -w '%{http_code} %{redirect_url}\n' --max-time 12 "https://authn.etzhayyim.com$p"
done
curl -sS -o /dev/null -w 'auth.gftd.ai/api/accounts/session=%{http_code}\n' \
  --max-time 12 https://auth.gftd.ai/api/accounts/session
```

```
/api/accounts/session                         -> 301 https://auth.gftd.ai/api/accounts/session
/xrpc/com.etzhayyim.auth.linkEmailBegin       -> 301 https://auth.gftd.ai/xrpc/com.etzhayyim.auth.linkEmailBegin
auth.gftd.ai/api/accounts/session=404
```

The live hop is `auth.gftd.ai` (reconfirmed 2026-08-30). The intended auth hub
is `auth.kotoba.cloud` (not `authn.gftd.ai`); that is desired topology, not the
Location header. The path this repository was built for is `404` on the live
hop. **There is no traffic waiting to be split off.**

## 4. Confirm nothing here can deploy

Two independent guards. Both must hold:

```bash
ls worker/wrangler.jsonc 2>&1 | tail -1        # must NOT exist
nbb --classpath worker/src-cljs -e '(ns p (:require [index])) (println (.-status (index/fetch-handler nil nil)))'
```

```
ls: worker/wrangler.jsonc: No such file or directory
501
```

The config is `wrangler.jsonc.disabled`, which `wrangler` does not discover, and
the handler answers `501` rather than a misleading empty `200`. If the first
command ever finds a `wrangler.jsonc`, someone has begun the ADR-0024 migration
— check `git log` before assuming it was deliberate.

## 5. Is the extraction provenance still intact?

`migration.edn` claims this is a verbatim copy of a specific upstream tree. That
claim is checkable, and checking it distinguishes "empty because it was always a
shell" from "empty because something was lost":

```bash
git hash-object CLAUDE.md MIGRATION-TODO.md NOTICE \
  worker/wrangler.jsonc.disabled worker/src-cljs/index.cljs
gh api repos/etzhayyim/root/git/trees/691c245da48f3acb11dd757218f189ff2482b1c8:60-apps \
  --jq '.tree[] | select(.path=="etzhayyim-project-accounts") | .sha'
wc -c CLAUDE.md MIGRATION-TODO.md NOTICE \
  worker/wrangler.jsonc.disabled worker/src-cljs/index.cljs | tail -1
```

```
ebfa85418e57654a3f6cf9faadf199616a2bbfde
efdef9cf1eefddf215491808e91b7bbcbf748cca
b59e2107c25b5cfd372647fbb775c530fe89ab9d
fb70955a8471da7f35dda428f5d79644e57944a1
5a5723597935f4a1dbb5fb341999d0e0e48e547a
77bb60eb6ed4a6482d569f5c808f6741f291101c
    8319 total
```

The tree sha matches `migration.edn`'s `:git-tree`, and the byte count matches
its `:bytes 8319`. The first four blobs are directly comparable against upstream
with:

```bash
gh api "repos/etzhayyim/root/contents/60-apps/etzhayyim-project-accounts?ref=691c245da48f3acb11dd757218f189ff2482b1c8" \
  --jq '.[] | "\(.sha[0:12])\t\(.name)"'
```

```
ebfa85418e57	CLAUDE.md
efdef9cf1eef	MIGRATION-TODO.md
b59e2107c25b	NOTICE
0fa0e9b0dc13	worker
```

Nothing drifted. **The provenance being clean is what makes "retire it" a safe
option rather than a lossy one** — the upstream original is still addressable.

## 6. Does the responsibility live somewhere else?

```bash
curl -sS -o /dev/null -w 'auth.itonami.cloud=%{http_code}\n'   --max-time 12 https://auth.itonami.cloud/
curl -sS -o /dev/null -w 'itonami.cloud/signin/=%{http_code}\n' --max-time 12 https://itonami.cloud/signin/
```

```
auth.itonami.cloud=200
itonami.cloud/signin/=200
```

Both live. The attach/detach implementation is in
`cloud-itonami/app-auth`'s `src/itonami/auth/worker.cljs` (search it for
`method-unlink`), decided by ADR-2608110100.

---

## If every check holds

Nothing to do. The repository is correctly inert and the README's account of it
is current.

## If a check fails

| Check | Failure | What it means |
|---|---|---|
| 1 | `NOERROR` | The hostname is back. `CLAUDE.md`'s plan may be live again — escalate before touching anything. |
| 2 | apex down | An etzhayyim-wide outage, unrelated to this repository. Not yours. |
| 3 | `200` on `auth.gftd.ai/api/accounts/session` | The account API is being served again on the live hop. Find out by whom before proposing retirement. |
| 4 | `wrangler.jsonc` exists, or handler is not `501` | Someone started the migration. `git log` first. |
| 5 | any sha differs | The working tree drifted from the recorded extraction. Do **not** retire — reconcile against upstream first. |
| 6 | either `404` | The replacement moved. The README's "where the responsibility went" needs re-checking before it can be cited. |

**Do not archive, delete, or make either copy private on the strength of these
checks alone.** They establish that the scaffold is dead, not that the owner has
decided what to do about it — the three options are in the README, and choosing
between them is theirs.
