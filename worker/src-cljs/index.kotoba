;; accounts.etzhayyim.com Worker entry — ClojureScript (squint) port of the former
;; src-ts/index.ts (ADR-2606251200: 60-apps TS→cljs+edn; accounts is the first
;; non-pilot app fully ported, lint:no-new-ts enforced-roots).
;;
;; SCAFFOLD ONLY (ADR-0024 Step 3) — not yet deployed (wrangler.jsonc.disabled).
;; auth Worker から以下の handler を移設する PR で wrangler を有効化する:
;;   - GET  /manage                              (Svelte SPA delegate)
;;   - GET  /api/accounts/session                → handle-accounts-session
;;   - POST /xrpc/com.etzhayyim.accounts.linkEmailBegin   → handle-link-email-begin
;;   - POST /xrpc/com.etzhayyim.accounts.linkEmailVerify  → handle-link-email-verify
;;   - POST /xrpc/com.etzhayyim.accounts.linkOAuthStart   → handle-link-oauth-start
;;   - POST /xrpc/com.etzhayyim.accounts.unlinkMethod     → handle-unlink-method
;;   - GET  /oauth/link/{google,microsoft}/callback       → handle-oauth-link-callback
;; 旧 com.etzhayyim.auth.* NSID は 90 日 alias 期間で受け付ける（ADR-0024 責務マトリクス）。
;;
;; Env binding (Cloudflare): ACCOUNTS_DB (D1) / AUTH_SERVICE (Fetcher) / ASSETS? /
;;   {GOOGLE,MICROSOFT}_OAUTH_CLIENT_{ID,SECRET}? — typed via the Worker runtime, not TS.
(ns index)

(defn fetch-handler
  "Cloudflare Worker fetch entry. Scaffold: 501 until the auth handlers are moved here."
  [_request _env]
  (js/Response. "accounts.etzhayyim.com scaffold — not yet deployed (ADR-0024 Step 3)"
                #js {:status 501
                     :headers #js {:content-type "text/plain"}}))

(def ^:export default #js {:fetch fetch-handler})
