# Photo Map

A single web page that plots the GPS locations of photos in someone's Google Drive on a Google Map. Each person signs in with their own Google account. Their photo data goes straight from Google Drive to their own browser. Nothing is stored or sent anywhere else.

Live site (once Pages is on): **https://n3g3ntr0pe.github.io/sitevisits/**

---

## 1. Publish on GitHub Pages

1. The repo is `n3g3ntr0pe/sitevisits`, with `index.html` at the root of `main`. It must stay public for Pages on a free plan.
2. Go to **Settings → Pages** (`https://github.com/n3g3ntr0pe/sitevisits/settings/pages`). Under **Build and deployment**, set the **Source** dropdown to **Deploy from a branch**. Under **Branch**, pick `main` and `/ (root)`, then click **Save**. Ignore the Visibility / GitHub Enterprise box on that page. It's an upsell and isn't needed.
3. After a minute the site is live at `https://n3g3ntr0pe.github.io/sitevisits/`.

At this point the page loads but shows "Setup needed". That's expected.

## 2. Set up Google Cloud

Go to <https://console.cloud.google.com> and sign in with **your** account (the builder's, not the other user's).

1. **Create a project**, e.g. "Photo Map".
2. **Enable two APIs** (APIs & Services → Library):
   - **Google Drive API**
   - **Maps JavaScript API**. Google Maps requires a billing account on the project. A prototype with one user normally stays inside the free monthly usage.
3. **Configure the OAuth consent screen** (Google Auth Platform / OAuth consent screen):
   - Audience / user type: **External**
   - Publishing status: leave it in **Testing**
   - App name and support email: anything sensible, such as "Photo Map"
   - Data access / scopes: add `.../auth/drive.metadata.readonly`
   - **Test users: add the other user's Google account email**, and your own if you want to test.
4. **Create the OAuth client ID** (Credentials → Create credentials → OAuth client ID):
   - Application type: **Web application**
   - Authorized JavaScript origins: `https://n3g3ntr0pe.github.io` (no path, no trailing slash)
   - Optionally also add `http://localhost:8000` for local testing.
   - Copy the **Client ID**.
5. **Create the Maps API key** (Credentials → Create credentials → API key), then click **Edit**:
   - Application restrictions: **Websites**, and add `https://n3g3ntr0pe.github.io/sitevisits/*` (plus `http://localhost:8000/*` if you test locally)
   - API restrictions: **Maps JavaScript API** only
   - Copy the **key**.

## 3. Add the credentials

In `index.html`, edit the `CONFIG` block near the top:

```js
const CONFIG = {
  OAUTH_CLIENT_ID: '1234567890-abc.apps.googleusercontent.com',
  MAPS_API_KEY:    'AIza...',
  MAP_ID:          'DEMO_MAP_ID',
};
```

Commit the change. GitHub Pages redeploys within a minute or two.

It's normal for both values to be visible in a public repo. An OAuth client ID isn't a secret, and the Maps key only works on your GitHub Pages address because of the website restriction.

## 4. Hand it to the other user

Send them the link `https://n3g3ntr0pe.github.io/sitevisits/` and tell them:

- Click **Sign in with Google** and choose the account that holds the photos.
- Google will show **"Google hasn't verified this app"**. Click **Continue**. This warning appears because the app is a private prototype in testing mode.
- Allow read-only access to Drive file information. The app can see file names and photo metadata. It can't change or delete anything.
- The map fills in after the scan. The status line shows how many photos had a location. Click a pin to see the photo and a link to open it in Drive.
- Sign-in lasts about an hour. After that, click **Sign in with Google** again.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| "Setup needed" | `CONFIG` values not filled in or not committed yet |
| Sign-in popup says `redirect_uri_mismatch` or `origin_mismatch` | The OAuth client's Authorized JavaScript origin doesn't exactly match `https://n3g3ntr0pe.github.io` |
| "Access blocked: app has not completed verification" | That Google account isn't in the consent screen's **Test users** list |
| Map shows "For development purposes only" or a grey map | Billing isn't enabled, or the Maps key's website restriction doesn't match the page URL |
| Few or no pins | Those photos have no GPS data (screenshots, messaging-app downloads and edited exports usually lose it), or the photos live in Google Photos rather than Drive |
| Pin pop-up shows no image | The Drive thumbnail couldn't load. The "Open in Google Drive" link still works |

## Notes

- The page scans every image in the user's own Drive, including files shared with them that appear in their Drive. Shared drives aren't included.
- Testing mode allows up to 100 test users, which is plenty for a prototype. Making it public would require Google's verification process for this scope.
