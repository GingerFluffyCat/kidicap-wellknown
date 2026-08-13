# kidicap passkey association files

Serves the app↔domain association files for the passkey relying-party ID
`auth.kidicap.app`, used by the KIDICAP Flutter app (`passkeys` package).

Both platforms refuse to create or use a passkey unless they can fetch these
files from the **rpId domain** over HTTPS. Nothing in the app can work around a
missing or wrong file.

| File | Consumed by | Purpose |
|---|---|---|
| `.well-known/assetlinks.json` | Google Play services | authorises the Android app for this rpId |
| `.well-known/apple-app-site-association` | Apple CDN | authorises the iOS app for this rpId |

## Before this works

**1. Fill in the Android fingerprints.** The three
`REPLACE_WITH_PLAY_APP_SIGNING_SHA256_*` placeholders must become the SHA-256 of
the certificate that **actually signs the installed APK**.

If the app ships via Play App Signing (it does, if it's on Play), that is the
*Play-managed app signing certificate*, **not** the local `kidicap.keystore`
upload cert:

> Play Console → your app → Release → Setup → App signing → *App signing key
> certificate* → SHA-256 certificate fingerprint

Using the upload cert here is the single most common cause of "passkeys silently
don't work on Android". Each flavor (`de.gip.kidicap`, `.qa`, `.sta`) is a
separate Play entry with its own signing key, so all three need their own value.

The local **debug** fingerprint is already present on the `.qa` and `.sta`
entries so debug builds work. It is deliberately **not** on the production
entry: anything signed with that debug key could otherwise request passkeys for
this domain under the production app id. Remove it from qa/sta too once you no
longer need local debug builds against this domain.

To read the release fingerprint locally instead (only valid if you do *not* use
Play App Signing):

```sh
keytool -list -v -keystore kidicap.keystore -alias "$keyAlias" \
  -storepass "$storePassword" | grep -A1 SHA256
```

**2. Point DNS at GitHub Pages.** `auth.kidicap.app` currently has **no DNS
record at all** (neither does the `kidicap.app` apex), so the domain must be
registered and delegated first. Then:

```
auth.kidicap.app.  CNAME  <org-or-user>.github.io.
```

In the repo: Settings → Pages → Source `main` / root, Custom domain
`auth.kidicap.app`, then wait for the Let's Encrypt cert and tick **Enforce
HTTPS**. Neither platform will follow a redirect or accept a bad cert.

**3. Keep `.nojekyll`.** GitHub Pages runs Jekyll by default, and Jekyll
**excludes dotfolders** — without this file `.well-known/` is never published
and both URLs 404.

## Known risk: Content-Type on the Apple file

Apple documents that `apple-app-site-association` must be served as
`application/json`. It has no file extension, so GitHub Pages will serve it as
`application/octet-stream` and **GitHub Pages cannot set custom headers**.

Verify it before trusting it:

```sh
curl -sSI https://auth.kidicap.app/.well-known/apple-app-site-association \
  | grep -i content-type
```

If Apple's CDN rejects it, GitHub Pages cannot be fixed — move this to a host
with header control (Cloudflare Pages or Netlify via a `_headers` file, or any
reverse proxy). Android has no such issue: `assetlinks.json` gets
`application/json` from its extension.

## Verifying

```sh
# raw fetches — both must be 200, HTTPS, no redirect
curl -sSI https://auth.kidicap.app/.well-known/assetlinks.json
curl -sS   https://auth.kidicap.app/.well-known/assetlinks.json | jq .

# Google's validator (authoritative for Android)
curl -sS "https://digitalassetlinks.googleapis.com/v1/statements:list\
?source.web.site=https://auth.kidicap.app\
&relation=delegate_permission/common.get_login_creds" | jq .

# Apple's CDN copy (authoritative for iOS; propagation is not instant)
curl -sS "https://app-site-association.cdn-apple.com/a/v1/auth.kidicap.app" | jq .
```

Google caches statements, and Apple's CDN copy updates on its own schedule, so
allow time after a change rather than assuming a fresh edit is live.

## Also required, and not in this repo

Each customer Keycloak realm must be configured to match, or the app's
assertions are rejected even with these files correct:

- WebAuthn Passwordless Policy → **Relying Party ID** = `auth.kidicap.app`
- **Extra Origins** must include the Android app origin
  `android:apk-key-hash:<base64url-sha256-of-signing-cert>` — Android's
  Credential Manager asserts that instead of an `https://` origin, so without it
  iOS logins succeed and every Android login fails origin validation.

Note that Keycloak's WebAuthn policy is **per-realm**: pointing rpId at
`auth.kidicap.app` breaks passkey login in desktop browsers on that realm,
because a browser requires the rpId to be a registrable suffix of the page
origin. If any tenant uses web passkeys, the mobile clients need their own realm.
