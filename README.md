# drsListing-web
# drslisting.ai — Booking page + Universal Links / App Links host

This folder is the deployable content for the **`drslisting.ai`** link domain
(and doubles as the static host for the **booking page**). It serves:

- **`booking.html`** — the booking form patients see after scanning the QR
  code from the doctor profile's "Book" button. It is a plain static page
  (no build step) that posts to the booking-page Supabase Edge Function as
  a JSON API.
- the platform-verification files (`.well-known/...`) for App Links /
  Universal Links, and redirects the HTTPS booking links to the static page,
  so a shared `https://drslisting.ai/book/<placeId>` link:
  - **opens the DrsListing app** when it's installed (Android App Links /
    iOS Universal Links), and
  - **falls back to the browser booking page** when it isn't (static rewrite).

> **Why a static page and not the Edge Function?** Supabase rewrites
> `text/html` GET responses to `text/plain` on the shared `*.supabase.co`
> domain, so browsers showed the raw HTML source instead of rendering the
> booking form. Serving `booking.html` from a normal static host (Netlify /
> Vercel / Cloudflare Pages) is the free fix. The QR code encodes
> `AppConstants.bookingPageUrl` which points here.

## Files

| File | Purpose |
|---|---|
| `booking.html` | Static booking form — reads `placeId` from the path + `token`/`name` from the query string, POSTs to the booking-page Edge Function |
| `.well-known/apple-app-site-association` | iOS Universal Links verification (must be served from `https://drslisting.ai/.well-known/apple-app-site-association`) |
| `.well-known/assetlinks.json` | Android App Links verification (must be served from `https://drslisting.ai/.well-known/assetlinks.json`) |
| `netlify.toml` | Netlify deploy config (rewrites + headers) |
| `vercel.json` | Vercel deploy config (rewrites + headers) |

## 1. Replace the placeholders

**iOS** — edit `.well-known/apple-app-site-association` and replace
`TEAM_ID_PLACEHOLDER` with your Apple Developer **Team ID** (10 chars, e.g.
`ABCDE12345`). The bundle id `com.drslisting.ai` must match the
Runner target (it already does).

**Android** — edit `.well-known/assetlinks.json` and replace
`SHA256_FINGERPRINT_PLACEHOLDER` with your signing certificate's SHA-256
fingerprint, colon-separated uppercase hex. Generate it from the project
root:

```bash
cd android && ./gradlew signingReport
# copy the SHA256 value under your release (or debug) variant
```

> ⚠️ The fingerprint must match the **same keystore the APK/AAB is signed
> with**. Play Store builds are usually signed by Play App Signing — use the
> fingerprint Google shows under *App integrity → App signing key
> certificate*.

## 2. Deploy to your static host

### Netlify

1. Create a new Netlify site.
2. In **Deploy settings**, set **Base directory** to `universal-links/` — Netlify only reads `netlify.toml` from the base directory, so this is required for the redirects and `.well-known` content-type headers to be applied. Leave **Build command** empty and set **Publish directory** to `.`.
3. Deploy. The `.well-known` files and redirects go live automatically.

### Vercel

1. Create a new Vercel project with root directory `universal-links/`.
2. No build step needed — `vercel.json` ships the redirects and `.well-known` files.
3. Deploy.

> **Note:** the `/manage-slots/*` fallback redirect currently points to the
> Android Play Store for everyone — iOS users opening a slot-management link
> will land on an irrelevant page. If you have a neutral landing page or an
> iOS App Store link, update `netlify.toml`/`vercel.json` accordingly.

## 2b. Point the booking QR code at this host

After deploying, update `bookingHost` in **`lib/config/constants.dart`**
(inside `AppConstants`) to the live URL of this host, e.g.:

```dart
static const String bookingHost = 'https://your-site.vercel.app';
```

That constant is what the QR code and the "Open Page" / "Copy Link"
actions encode (`AppConstants.bookingPageUrl(placeId)`). The `token` shared
secret is already embedded by the app, so no other change is needed. Rebuild
the app afterwards so the new QR URL ships.

## 3. Point the domain

Set your DNS so `drslisting.ai` (and `www` if desired) point at the static
host (Netlify/Vercel provide a CNAME target after connecting the site).
Then set the custom domain in the dashboard.

## 4. Verify

- **Android:** open `https://drslisting.ai/.well-known/assetlinks.json` in a browser — must be valid JSON with the correct fingerprint. Reinstall the app, then run:
  ```bash
  adb shell pm verify-app-links com.drslisting.ai
  ```
- **iOS:** check `https://drslisting.ai/.well-known/apple-app-site-association` returns the JSON. Apple also offers the AASA validator on the *Associated Domains* support page.

## 5. Test the full flow

Share/open `https://drslisting.ai/book/<placeId>` (the QR from the doctor
profile encodes `AppConstants.bookingPageUrl`, which points at this host —
see section 2b):

- **App installed** → app opens and routes to the booking page.
- **App not installed** → browser lands on the static booking form
  (`booking.html` served by this host), which submits to the Supabase
  booking Edge Function.

---

> **Note:** the `token` shared secret is embedded in the redirect rules here
> (same value as `AppConstants.bookingSharedSecret`). If you rotate it, update
> `netlify.toml`/`vercel.json` **and** `lib/config/constants.dart` together.
