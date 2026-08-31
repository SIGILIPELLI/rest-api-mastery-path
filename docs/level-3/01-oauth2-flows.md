# 01 · OAuth2 Flows

Level 1's authentication module covered API keys and bearer tokens as
opaque credentials. OAuth2 is the standard framework for how a *token gets
issued in the first place* — especially when a third-party app needs
limited access to a user's data on another service, without ever seeing
that user's password.

## The core actors

- **Resource owner** — the user who owns the data (e.g. a GitHub user).
- **Client** — the app requesting access (e.g. a CI tool wanting to read
  the user's repos).
- **Authorization server** — issues tokens after the user consents
  (`github.com/login/oauth/authorize`).
- **Resource server** — the API that accepts the token (`api.github.com`).

The point of OAuth2: the client never sees the user's GitHub password. It
gets a scoped, revocable **access token** instead.

## Authorization Code flow (the standard for web/mobile apps)

```
1. Client redirects the user's browser to the authorization server:

   GET https://auth.example.com/authorize?
       response_type=code
       &client_id=abc123
       &redirect_uri=https://myapp.com/callback
       &scope=read:profile
       &state=xyz789

2. User logs in and approves the requested scopes.

3. Authorization server redirects back with a one-time code:

   GET https://myapp.com/callback?code=SPLXLOBEZQQY&state=xyz789

4. Client exchanges the code for tokens (server-to-server, not visible to the browser):
```

```bash
curl -X POST https://auth.example.com/token \
  -d "grant_type=authorization_code" \
  -d "code=SPLXLOBEZQQY" \
  -d "redirect_uri=https://myapp.com/callback" \
  -d "client_id=abc123" \
  -d "client_secret=$CLIENT_SECRET"
```

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "8xLOxBtZp8"
}
```

Two exchanges instead of one is deliberate: step 3's code is exposed to the
browser (in the redirect URL, potentially logged) but is single-use and
short-lived; step 4's exchange happens server-to-server with a secret the
browser never sees, so a leaked authorization code alone is useless without
the client secret.

The `state` parameter is a CSRF defense: the client generates a random
value before step 1, and verifies the same value comes back in step 3 —
without it, an attacker could trick a victim's browser into completing an
OAuth flow initiated by the attacker.

## PKCE — required for public clients (mobile, SPA)

A mobile app or single-page app can't safely hold a `client_secret` (it
ships inside the app binary or JS bundle, so any user can extract it). PKCE
(**P**roof **K**ey for **C**ode **E**xchange) replaces the secret with a
per-request, dynamically generated proof:

```
1. Client generates a random "code_verifier", derives a "code_challenge" = SHA256(code_verifier):

   GET https://auth.example.com/authorize?
       response_type=code&client_id=abc123&redirect_uri=...
       &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
       &code_challenge_method=S256

2. Exchange the code, presenting the original verifier instead of a secret:
```

```bash
curl -X POST https://auth.example.com/token \
  -d "grant_type=authorization_code" \
  -d "code=SPLXLOBEZQQY" \
  -d "code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk" \
  -d "client_id=abc123"
```

The authorization server recomputes `SHA256(code_verifier)` and checks it
matches the `code_challenge` from step 1 — proving that whoever is
redeeming the code is the same party that started the flow, without any
shared secret. PKCE is now recommended for **all** clients, not just public
ones (RFC 8252 / the current OAuth2 Security BCP).

## Client Credentials flow — service-to-service, no user involved

When there's no human user in the loop (a backend job calling another
service's API on its own behalf):

```bash
curl -X POST https://auth.example.com/token \
  -d "grant_type=client_credentials" \
  -d "client_id=service-a" \
  -d "client_secret=$CLIENT_SECRET" \
  -d "scope=orders:read"
```

```json
{ "access_token": "eyJhbGciOiJSUzI1NiIs...", "token_type": "Bearer", "expires_in": 3600 }
```

One request, no redirect, no user consent screen — appropriate for
machine-to-machine access where the "client" *is* the resource owner of its
own scoped access.

## Refresh tokens

Access tokens are deliberately short-lived (often an hour) to limit the
blast radius of a leaked one. A long-lived **refresh token** lets the
client get a new access token without re-prompting the user for
credentials:

```bash
curl -X POST https://auth.example.com/token \
  -d "grant_type=refresh_token" \
  -d "refresh_token=8xLOxBtZp8" \
  -d "client_id=abc123" \
  -d "client_secret=$CLIENT_SECRET"
```

```json
{ "access_token": "eyJhbGciOiJSUzI1NiIs...(new)", "expires_in": 3600, "refresh_token": "new-8xLOxBtZp9" }
```

Best practice is **refresh token rotation**: each use invalidates the old
refresh token and issues a new one, so a stolen refresh token that gets
used by an attacker is immediately detectable (the legitimate client's next
refresh attempt fails, signaling compromise).

## Flows to avoid

- **Resource Owner Password Credentials** — the client collects the user's
  actual username/password and trades them directly for a token. Deprecated
  in the current OAuth2 spec; it defeats the entire point (the client sees
  the password) and should only ever appear in legacy, fully first-party
  migrations.
- **Implicit flow** — returned the access token directly in the redirect
  URL fragment, skipping the code-exchange step. Deprecated in favor of
  Authorization Code + PKCE, since tokens in URL fragments are more exposed
  (browser history, referrer leaks, logs).

## Worked example: a CLI tool authenticating to a REST API

A CLI tool (public client, can't hold a secret) needs delegated,
revocable access to a user's account:

1. CLI opens the user's browser to the authorize URL with PKCE parameters.
2. User approves; browser redirects to a local `http://localhost:8912/callback`
   the CLI is temporarily listening on.
3. CLI exchanges the code + verifier for tokens, stores the refresh token
   in the OS keychain (never plaintext on disk).
4. CLI uses the access token as `Authorization: Bearer ...` on every API
   call, transparently refreshing when it expires.

## Exercise

1. Explain why the Authorization Code flow uses a two-step exchange
   (code, then token) instead of returning the access token directly at
   step 3.
2. Why is PKCE mandatory for a mobile app but optional (though still
   recommended) for a traditional server-side web app that can hold a
   `client_secret`?
3. A backend service needs to call a partner's API with no user involved.
   Which flow fits, and why would Authorization Code be the wrong choice
   here?
4. What does refresh token rotation protect against that a static,
   reusable refresh token does not?
