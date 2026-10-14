# Photo Map

A single web page that plots the GPS locations of photos on a Google Map. The user chooses which photos to map, from their own device or from their own Google Drive. The app never scans a whole Drive or photo library. Photos are read in the user's browser, and nothing is stored or sent anywhere else.

Live site: **https://n3g3ntr0pe.github.io/sitevisits/**

---

## How it works

| Source | What happens |
|---|---|
| **Choose photos / Choose folder / drag and drop** | The browser reads the photo files directly. No sign-in is needed. |
| **From Google Drive** | The user signs in with Google and picks photos in the Google Picker. The app gets the `drive.file` scope, which only reaches the files picked. Each picked file is downloaded into the browser. |

In both cases [exifr](https://github.com/MikeKovarik/exifr) reads the GPS position and capture date from the photo's EXIF data, including HEIC files from iPhones. Photos without GPS data are counted but not plotted.

Google Photos isn't supported as a source. Google's Photos APIs remove location data, so photos from there have to be downloaded first, for example with Google Takeout.

## Setup (already done for this repo)

Everything lives in Google Cloud project `sitevisits-511607` (project number `673202376182`).

1. **GitHub Pages:** Settings → Pages → Deploy from a branch → `main`, `/ (root)`.
2. **APIs enabled:** Maps JavaScript API, Google Drive API, Google Picker API. Maps needs a billing account on the project. A small prototype normally stays inside the free monthly usage.
3. **API key** (one key for Maps and the Picker):
   - Application restriction: **Websites**: `https://n3g3ntr0pe.github.io/sitevisits/*`, `http://localhost:8000/*`, `https://docs.google.com/*`. The Picker needs the last one.
   - API restriction: **Maps JavaScript API** and **Google Picker API**.
4. **Google Auth Platform:**
   - Audience: **External**, publishing status **Testing**. Don't publish.
   - Test users: everyone who will use the Drive button (up to 100).
   - Data access: `.../auth/drive.file` only. It's a non-sensitive scope.
   - Client: Web application, Authorized JavaScript origins `https://n3g3ntr0pe.github.io` and `http://localhost:8000`, no redirect URIs. The client secret isn't used.
5. **`CONFIG` in `index.html`:** `MAPS_API_KEY`, `MAP_ID`, `OAUTH_CLIENT_ID` and `APP_ID` (the project number). If `OAUTH_CLIENT_ID` still starts with `YOUR_`, the Drive button stays hidden and local files still work.

These values are meant to be public. An OAuth client ID and a project number aren't secrets, and the key only works from the listed websites and for the listed APIs.

## Instructions for users

Send them the link `https://n3g3ntr0pe.github.io/sitevisits/` and tell them:

- **Photos on your computer or phone:** click **Choose photos** or **Choose folder**, or drag photos onto the map. No sign-in is needed.
  - On an **iPhone**, Safari removes the location from photos picked from the photo library unless you turn it on. In the picker, tap **Options** and switch on **Location**. Alternatively, use the page on a computer with the original photos.
- **Photos in Google Drive:** click **From Google Drive** and sign in. Your account must be on the test-user list.
  - Google shows **"Google hasn't verified this app"**. Click **Continue**. The app is a private prototype in testing mode.
  - Allow access to "the specific Google Drive files you use with this app". The app sees only the photos you pick.
  - Pick photos, holding Ctrl or Cmd to pick several, then click **Select**. You can open folders to browse inside them, but you can't select a whole folder.
  - Sign-in lasts about an hour.
- Click a pin to see the photo, its date and links to Google Maps (and Drive). **Clear** empties the map.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| "Setup needed" | `MAPS_API_KEY` not filled in |
| No "From Google Drive" button | `OAUTH_CLIENT_ID` not filled in, or a Google script failed to load (see console) |
| Sign-in says `origin_mismatch` | The client's JavaScript origins don't exactly match the page's origin. Changes can take minutes to hours to apply |
| "Access blocked: app has not completed verification" | That Google account isn't in **Test users** |
| Picker says "API developer key is invalid" | The key is missing `https://docs.google.com/*` or the Picker API restriction |
| Grey map or "Google Maps rejected the API key" | Billing, or the key's website restriction doesn't match the page URL (`localhost`, not `127.0.0.1`) |
| Few or no pins | Those photos have no GPS data. Screenshots, messaging-app copies, edited exports and iPhone uploads without **Location** switched on usually have none |
| Pop-up shows no image | The browser can't display that format (HEIC outside Safari) and the photo has no embedded thumbnail |

## Local testing

```
python3 -m http.server 8000 --bind 127.0.0.1
```

Open `http://localhost:8000`, not `127.0.0.1:8000`, so that it matches the key and the OAuth origins.
