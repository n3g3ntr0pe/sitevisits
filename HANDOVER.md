# Handover: Photo Map setup (for a local Claude Code session)

Written 2026-10-14 by a Claude Code cloud session. It's addressed to a fresh Claude Code session on the owner's own computer. Read all of it before you act. The work is mostly done. What's left is the Google Cloud setup, two values in `index.html`, and a live test.

---

## 0. Update 2026-10-14 (local session): read this first

**The design changed. Where this section disagrees with §3, §6 or §8 below, this section wins. The rest of the file is kept as history.**

### Reversed decisions

- **Scanning a whole Drive is forbidden.** The owner's rule: the app MUST NOT scan a user's entire Drive or Photos. The `drive.metadata.readonly` auto-scan is gone.
- **Photos come from user picks only:**
  - **Local files:** Choose photos, Choose folder, or drag and drop. No sign-in.
  - **Google Drive Picker** with the `drive.file` scope, which is non-sensitive and reaches only the picked files. Picked files are downloaded (`alt=media`) into the browser and go through the same pipeline as local files.
- **GPS comes from EXIF in the browser** via `exifr@7.1.3` (`full` build, HEIC-capable), not from Drive's `imageMediaMetadata`. This avoids the `thumbnailLink` CORS problem, and thumbnails come from the file itself.
- **Dependencies pinned:** `exifr@7.1.3`, `@googlemaps/markerclusterer@2.6.2`. Keep exact versions.

### Why: verified against Google's docs on 2026-10-14

- The end user shoots on an **iPhone**. The likely backup target is iCloud (no API) or Google Photos. Drive is the least likely.
- **Google Photos is a dead end for GPS.** The Picker API has no location field, `=d` downloads strip "the location metadata", and since 2025-04 the Library API only reads app-created items. The workaround is a download or Takeout, then local files.
- Picking a folder under `drive.file` probably doesn't grant access to its children (inferred, not tested). So the Picker allows browsing into folders but not selecting a folder.

### Current state: done and tested

| Item | State |
|---|---|
| Cloud project | `sitevisits-511607`, number `673202376182`, billing linked |
| APIs | Maps JavaScript, Drive, Picker enabled |
| API key "Maps Platform API Key" | Referrers: Pages URL, `localhost:8000`, `docs.google.com`. APIs: Maps JS and Picker. In `CONFIG.MAPS_API_KEY`. |
| Auth platform | External, **Testing**. Test users: owner and end user. Scope `drive.file` only. |
| OAuth client `sitevisits` | Web. Origins `https://n3g3ntr0pe.github.io`, `http://localhost:8000`. In `CONFIG.OAUTH_CLIENT_ID`. The auto-created client secret is unused and must never be committed. |
| Local test | Owner confirmed local files and the Drive Picker work on `localhost:8000` |

**Open items:**
- Live test on Pages.
- Hand-off to the end user (README "Instructions for users", including the iPhone **Options → Location** tip).
- Check that the end user's iPhone has Location Services → Camera turned on.

---

## 1. What this is

**Photo Map** is a rapid prototype: a single web page that plots the GPS locations of photos stored in someone's **Google Drive** on a Google Map.

- **Owner and builder:** GitHub user `n3g3ntr0pe`, the person you're working with.
- **End user:** another person, who opens the page and signs in with **their own Google account**. Their photo data goes straight from Drive to their own browser. Nothing is stored anywhere else.
- **Assumption, not yet verified:** the photos are in Google **Drive**, not Google **Photos**. See §8 if that turns out to be wrong.

| | |
|---|---|
| Repo | https://github.com/n3g3ntr0pe/sitevisits (public, default branch `main`) |
| Live site | https://n3g3ntr0pe.github.io/sitevisits/ (GitHub Pages, deploy from branch `main`, `/ (root)`) |
| Live state now | The page loads and shows **"Setup needed: add the OAuth client ID and Maps API key…"**. That's expected until step 5 below. |

## 2. Why the work moved to your machine

The cloud session couldn't finish the setup:

- **gcloud sign-in was blocked twice** by the cloud session's safety checks, once as a secret-store write and once as an account change. The owner's Google credential can't enter that container.
- **The cloud session has no browser tools.** The consent screen, test users and OAuth client have **no public API**, so they have to be done in the Cloud Console UI.
- Your machine already has an **authenticated gcloud** (credential in the keyring) and **Claude in Chrome**, which is the owner's preferred browser. Between them you can do the whole setup, with the owner approving prompts.

## 3. Settled design decisions (don't revisit)

- **Static single page, no backend.** The browser uses the **Google Identity Services token client** (`google.accounts.oauth2.initTokenClient`) with scope `https://www.googleapis.com/auth/drive.metadata.readonly`.
  - That scope is **restricted**. That's fine while the app stays in **Testing** mode with listed **test users** (up to 100). Publishing would need Google verification, so don't publish.
  - Tokens last about 1 hour, with no refresh tokens by design. On a 401 the page resets to "Sign in with Google". Sign-out revokes the token.
- **No Picker.** The page auto-scans the whole Drive using Drive API v3 `files.list`:
  - `q = mimeType contains 'image/' and trashed = false`, `spaces=drive`, `pageSize=1000`, paged with `nextPageToken`
  - `fields = nextPageToken,files(id,name,thumbnailLink,webViewLink,imageMediaMetadata(location,time))`
  - Drive parses EXIF GPS into `imageMediaMetadata.location`, so there's no image download or EXIF parsing. Locations at exactly 0,0 are skipped.
- **Map:** Google Maps JS API, loaded dynamically with `v=weekly`, `libraries=marker`, `loading=async` and a callback. It uses `AdvancedMarkerElement` with `mapId: 'DEMO_MAP_ID'`, which is fine for a prototype. Clustering comes from `@googlemaps/markerclusterer@2` on unpkg (global `markerClusterer.MarkerClusterer`; it resolved to 2.6.2 at test time).
- **Pin click:** `addEventListener('gmp-click', …)` with `gmpClickable: true`. The old `addListener('click')` logged a deprecation warning, so it was replaced and retested (commit `1d2be8e`).
- **Info window:** shows the thumbnail (`thumbnailLink` with `=sNNN` rewritten to `=s400`, `referrerpolicy="no-referrer"`, and the `<img>` removed on error), the file name, the EXIF date reformatted from `YYYY:MM:DD HH:MM:SS` to `YYYY-MM-DD HH:MM`, and an **Open in Google Drive** link.
- **Status line:** shows "X of Y photos have a location." during and after the scan.
- **Setup guard:** if either `CONFIG` value still starts with `YOUR_`, the page shows "Setup needed" and stops.
- **Hosting:** GitHub Pages.
- **Rejected alternatives:**
  - Leaflet/OpenStreetMap, which would avoid Maps billing. The owner chose Google Maps.
  - A Colab + Folium notebook.
  - A My Maps KML import.
  - "Option 2": the builder runs the scan and hands over a static map.

## 4. Repo contents and history

| File | What it is |
|---|---|
| `index.html` | The whole app, about 250 lines of vanilla JS and CSS, with a dark-mode-aware header. **`CONFIG` is at lines 49–53**: `OAUTH_CLIENT_ID` and `MAPS_API_KEY` are placeholders, `MAP_ID` is `'DEMO_MAP_ID'`. |
| `README.md` | Setup guide with the real URLs: §1 Pages, §2 Google Cloud, §3 add credentials, §4 hand-off instructions for the end user, plus Troubleshooting and Notes. |
| `HANDOVER.md` | This file. |

Commits, oldest first:

- `b569634`: GitHub's placeholder README.
- `dc7aea8`: adds the app and the guide.
- `1d2be8e`: the `gmp-click` fix and clearer Pages wording in README §1.
- The commit that added this file, merged into `main` from branch `claude/laughing-pascal-l9zsyl` by PR.

**Pull `main`.** Everything above is on it, so no branch juggling is needed.

Git identity used so far: `n3g3ntr0pe`. Your local git config should already have it.

## 5. What's been verified and what hasn't

**Verified** in a headless Chromium smoke test in the cloud. It used the real Maps JS library (loaded without a key, so it showed the "for development purposes only" watermark), a stubbed GIS token client, mocked Drive responses (6 files over 2 pages) and markerclusterer from npm. The following all worked:

- Paging, request params and the `Bearer` header
- Filtering out photos with no location or a 0,0 location ("3 of 6 photos have a location")
- Markers and clustering
- Pin click opening the info window with the date and Drive link
- The thumbnail fallback when the image returned 403
- Sign-out revoking the token and clearing the pins

The JS also passes `node --check`.

**Not verified yet:**

- Real OAuth sign-in
- Real Drive data
- Whether real `thumbnailLink` images display in an `<img>` (the fallback is in place)
- GPS coverage for HEIC photos
- Real Maps key and billing

One harmless console warning is expected: `<gmp-advanced-marker>: Please use addEventListener with 'gmp-' events instead of addListener.` It comes from **inside markerclusterer** (its cluster icons), not from our code. Ignore it.

## 6. Your task, step by step

### Step 0: ask the owner first

Collect these before creating anything:

1. **The end user's Google account email.** It's needed as a test user, and this handover doesn't have it.
2. Whether the owner's own Google account should also be a test user. The answer is normally yes, so they can test first. Ask which address.
3. Run `gcloud auth list` and `gcloud billing accounts list --filter=open=true`, and confirm with the owner:
   - the active account is the one that should own the project;
   - which billing account to use, if there's more than one.
   - If there's **no open billing account**, the owner has to create one at https://console.cloud.google.com/billing and enter card details themselves. **Never enter payment details yourself.** The Maps JavaScript API won't work without billing. A one-user prototype normally stays inside the free monthly usage.
4. If gcloud says the Cloud **Terms of Service** haven't been accepted, which happens with first-time Cloud users, the owner has to accept them once in the console.

### Step 1: project, billing, APIs and Maps key (gcloud)

Run these in a terminal, getting the owner's approval as usual:

```bash
PROJECT="photo-map-$(date +%s | tail -c 7)"     # project IDs must be globally unique
gcloud projects create "$PROJECT" --name="Photo Map"
gcloud billing projects link "$PROJECT" --billing-account="<BILLING_ACCOUNT_ID>"   # from step 0
gcloud services enable drive.googleapis.com maps-backend.googleapis.com apikeys.googleapis.com --project="$PROJECT"
gcloud services api-keys create --project="$PROJECT" --key-id=photo-map-maps \
  --display-name="Photo Map (Maps JS)" \
  --allowed-referrers="https://n3g3ntr0pe.github.io/sitevisits/*" \
  --api-target=service=maps-backend.googleapis.com
gcloud services api-keys get-key-string photo-map-maps --project="$PROJECT" --format='value(keyString)'
```

Notes:

- `maps-backend.googleapis.com` is the **Maps JavaScript API**. `apikeys.googleapis.com` is needed for the `api-keys` commands.
- The key is restricted to **websites** `https://n3g3ntr0pe.github.io/sitevisits/*` and to the **Maps JavaScript API only**. For local testing, also add `http://localhost:8000/*` with `gcloud services api-keys update photo-map-maps --allowed-referrers=…` (the list replaces the old one, so include both entries).
- Key restrictions can take a few minutes to take effect.
- The flags were checked against Cloud SDK 570, but the commands have never been run for real.

### Step 2: consent screen and OAuth client (Cloud Console, via Claude in Chrome)

No API exists for this part. gcloud's `iap oauth-brands` and `iap oauth-clients` only make Workspace-internal clients owned by IAP, and `iam oauth-clients` is for workforce federation. **Neither works here.** Use the console, in the **Google Auth Platform** section. The URL paths below are the current ones; if one doesn't resolve, use the left-hand navigation names instead.

1. https://console.cloud.google.com/auth/overview?project=PROJECT, then **Get started**:
   - App name `Photo Map`, user support email = the owner's.
   - Audience **External**.
   - Contact email = the owner's.
   - Agree to the policy, then **Create**.
2. **Audience** (`/auth/audience?project=PROJECT`):
   - Publishing status stays **Testing**. Don't click "Publish app".
   - **Test users → Add users**: the end user's email, plus the owner's.
3. **Data access** (`/auth/scopes?project=PROJECT`):
   - **Add or remove scopes**, filter for `drive.metadata.readonly`.
   - Tick `.../auth/drive.metadata.readonly`, click **Update**, then **Save**.
4. **Clients** (`/auth/clients?project=PROJECT`), then **Create client**:
   - Application type: **Web application**, name `Photo Map web`.
   - **Authorized JavaScript origins:** `https://n3g3ntr0pe.github.io`, exactly, with no path and no trailing slash. Optionally add `http://localhost:8000` for local testing.
   - No redirect URIs are needed, because the token client uses a popup.
   - **Create**, then copy the **Client ID** (`…apps.googleusercontent.com`).
   - **No client secret is needed. Don't store or commit one.**

### Step 3: put the values into the app and deploy

Edit `index.html` lines 50–51:

```js
  OAUTH_CLIENT_ID: '<client id>.apps.googleusercontent.com',
  MAPS_API_KEY:    '<maps key>',
```

Leave `MAP_ID` as `'DEMO_MAP_ID'`. Then commit to `main` and push. Both values are meant to be public:

- an OAuth client ID isn't a secret;
- the Maps key is locked to the Pages URL and the Maps JS API.

**Never** commit gcloud tokens or credentials.

Pages redeploys in about 1–2 minutes. Check that https://n3g3ntr0pe.github.io/sitevisits/ now shows "Sign in with the Google account that holds your photos." instead of "Setup needed".

### Step 4: owner test

The owner signs in on the live page with their test-user account. They should see:

1. **"Google hasn't verified this app"**, then **Continue**. That's expected in Testing mode.
2. A consent prompt for read-only Drive metadata. Allow it.
3. The status line counting up, then "X of Y photos have a location.", with pins on the map.
4. Pin clicks open a pop-up with a thumbnail (or no image, if thumbnails are blocked), the date, and the Drive link.

If something fails, use the README **Troubleshooting** table and the browser console. Common causes:

- `origin_mismatch`: the JS origin isn't exactly `https://n3g3ntr0pe.github.io`.
- "Access blocked: app has not completed verification": that account isn't a test user.
- "Google Maps rejected the API key", or a grey or watermarked map: billing, or the key's referrer restriction.
- A Drive API 403 saying the API is disabled: the Drive API isn't enabled on **this** project, or hasn't propagated yet.

It's fine to have the owner do the actual Google sign-in themselves rather than driving their account with Claude in Chrome.

### Step 5: hand off to the end user

Send them the link, with the instructions in README §4:

- sign in with the account that holds the photos;
- click Continue on the unverified-app screen;
- allow read-only access;
- sign in again after about an hour.

## 7. Small fixes that are OK to make

- Fix doc wording that turns out to be wrong once you've done the real console flow. README §2 describes the same steps as §6 here.
- **GitHub Pages runs Jekyll** on this repo, which processes Liquid template tags even inside Markdown code spans. Never put a double opening curly brace, or an opening curly brace followed by a percent sign, in any Markdown file, or the Pages build can fail.
- Keep the app a single static `index.html`. No build step, no backend, no new dependencies.

## 8. Caveats and fallbacks

- **Coverage depends on GPS EXIF.** These usually have no location: screenshots, WhatsApp and other messaging-app downloads, and edited exports. Few pins doesn't mean a bug, because the status line shows the ratio.
- **HEIC:** Drive may or may not extract GPS from HEIC. Untested.
- **Shared drives aren't scanned** (`spaces=drive`, no `corpora=allDrives`). Files shared *with* the user that appear in their Drive are included.
- **If the photos are really in Google Photos**, this approach doesn't work. The Photos Library API only exposes media the app itself created. The fallback is a **Google Takeout** export of Photos, followed by local EXIF parsing (for example a small script that writes a static map). That's a new design, so discuss it with the owner first.
- **Testing mode:** up to 100 test users. Each new person has to be added as a test user before they can sign in.

## 9. Environment notes from the cloud sessions (background only)

- The cloud GitHub proxy couldn't create repos or call the Pages API, so the owner enabled Pages by hand. On your machine, normal `git` and `gh` against your own credentials should just work.
- Pages is already configured: Source = Deploy from a branch, Branch = `main`, `/ (root)`. Ignore the Visibility/Enterprise upsell box on the Pages settings page.
- A local smoke test without real credentials is possible:
  - `python3 -m http.server 8000` in the repo, after temporarily putting non-`YOUR_` values in `CONFIG`;
  - stub `google.accounts.oauth2` and mock `https://www.googleapis.com/drive/v3/files` (for example with Playwright `page.route`).
  - Don't commit the fake values.
