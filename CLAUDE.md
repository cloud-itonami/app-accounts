# accounts.etzhayyim.com — Account Lifecycle Management (Scaffold)

**Status**: SCAFFOLD ONLY (2026-04-14). 本 Worker は ADR-0024 Step 3 の物理分離先として
ディレクトリと skeleton のみ用意した。**まだ deploy しない**。route は wrangler に
登録していない (auth Worker の現行 route が `accounts.etzhayyim.com/*` を serve し続ける)。

## 目的

ADR-0024 "Auth / Accounts Worker Topology" に基づき、`accounts.etzhayyim.com` を
`60-apps/etzhayyim-project-auth` から物理分離する。

| Worker | ドメイン | 責務 |
| --- | --- | --- |
| `etzhayyim-auth` (既存) | `authn.etzhayyim.com` | Passkey / OAuth PKCE / Session 発行 / DID / Service Auth |
| `etzhayyim-accounts` (本 Worker) | `accounts.etzhayyim.com` | Linked auth methods / actor.score / provider link-unlink / `/manage` UI |

## Migration 前提

1. ADR-0022 Step 7 本番 deploy 完了 (Cookie / AUTH_SERVICE delegation session 昇格
   path の live 削除、`[auth][deprecated]` zero-traffic 確認)
2. ADR-0024 Step 2 完了 (auth Worker の `authn.etzhayyim.com` route pattern 絞り込み、
   外部 `com.atproto.*` zero-traffic 確認)

上記 2 点が未了の間、本 Worker は **scaffold のみ**。実装の移設は別 PR で行う。

## 移設対象 (auth Worker から剥がす route)

`60-apps/etzhayyim-project-auth/worker/src-ts/index.ts` のうち `accounts.etzhayyim.com`
でのみ serve されるべき path:

- `GET  /manage` — account management UI (Svelte SPA)
- `GET  /api/accounts/session`
- `POST /xrpc/com.etzhayyim.auth.linkEmailBegin`
- `POST /xrpc/com.etzhayyim.auth.linkEmailVerify`
- `POST /xrpc/com.etzhayyim.auth.linkOAuthStart`
- `POST /xrpc/com.etzhayyim.auth.unlinkMethod`
- `GET  /oauth/link/google/callback`
- `GET  /oauth/link/microsoft/callback`

NSID は移設時に `com.etzhayyim.auth.*` → `com.etzhayyim.accounts.*` に rename する (ADR-0024
責務マトリクス準拠)。旧 NSID は 90 日 alias で受け付ける。

## D1 分離

| DB binding | 現所属 Worker | 移設後 |
| --- | --- | --- |
| `AUTH_DB` (passkey_credentials, email_link_codes) | auth | **auth 継続** |
| `ACCOUNTS_DB` (linked_auth_methods, score_history) | 未分離 | **accounts に新設** |
| `KEYS_DB` (revoked_sessions, DID signing keys) | auth | **auth 継続** |

accounts Worker は passkey credential lookup が必要な場面で `env.AUTH_SERVICE`
service binding 経由で auth Worker に問い合わせる (直接 `AUTH_DB` には触らない)。

## References

- `90-docs/adr/0024-auth-accounts-worker-topology.md`
- `90-docs/adr/0022-auth-topology-consolidation.md`
- `60-apps/etzhayyim-project-auth/CLAUDE.md`
