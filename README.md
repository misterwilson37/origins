# Origins Game Fair Planner

A single-file web app for Jake, Dan, and Tim to browse the Origins Game Fair schedule, flag events, build per-person favorites, detect schedule conflicts, and export for Claude-assisted scheduling.

Lives at: **origins.misterwilson.org**

---

## Prerequisites

- A GitHub account (you already have one — it's where misterwilson.org lives)
- A Firebase project (you're already comfortable with these)
- About 20 minutes

---

## Part 1: Firebase Setup

### 1.1 Create a new Firebase project

1. Go to [console.firebase.google.com](https://console.firebase.google.com)
2. Click **Add project**
3. Name it `origins-planner` (or similar — this becomes part of your project ID)
4. Disable Google Analytics (not needed)
5. Click **Create project**

### 1.2 Create the Firestore database

1. In your new project, click **Firestore Database** in the left sidebar
2. Click **Create database**
3. Choose **Production mode** (we'll set the rules properly in a moment)
4. Choose your region — `us-east1` is closest to Columbus/Nashville
5. Click **Enable**

### 1.3 Deploy the security rules

The file `firestore.rules` in this repo is ready to deploy.

**Option A — Firebase CLI (recommended):**
```bash
npm install -g firebase-tools
firebase login
firebase init firestore   # point it at your new project
# it will ask which project; select origins-planner
# it will ask about the rules file — use firestore.rules
firebase deploy --only firestore:rules
```

**Option B — Paste manually:**
1. In Firebase Console, go to **Firestore Database → Rules**
2. Replace everything with the contents of `firestore.rules`
3. Click **Publish**

### 1.4 Get your Firebase config

1. In Firebase Console, click the gear icon → **Project settings**
2. Scroll to **Your apps** → click the `</>` (Web) icon
3. Register the app with nickname `origins-planner-web`
4. Copy the `firebaseConfig` object — it looks like:
```javascript
const firebaseConfig = {
  apiKey: "AIzaSy...",
  authDomain: "origins-planner-12345.firebaseapp.com",
  projectId: "origins-planner-12345",
  storageBucket: "origins-planner-12345.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abc123..."
};
```

### 1.5 Add your config to index.html

Open `index.html` and find this block near the bottom (around line 700):
```javascript
const FIREBASE_CONFIG = {
  apiKey:            "REPLACE_WITH_YOUR_API_KEY",
  authDomain:        "REPLACE_WITH_YOUR_PROJECT_ID.firebaseapp.com",
  ...
```
Replace each `REPLACE_WITH_...` value with the values you copied in step 1.4.

### 1.6 Restrict your API key (optional but recommended)

Since the API key is visible in the HTML source, restrict it to your domain so nobody else can use it:

1. Go to [console.cloud.google.com/apis/credentials](https://console.cloud.google.com/apis/credentials)
2. Find the API key Firebase created (named something like `Browser key (auto created by Firebase)`)
3. Click on it → **Application restrictions** → **HTTP referrers**
4. Add: `https://origins.misterwilson.org/*`
5. Save

This means the key only works when requests come from your domain.

---

## Part 2: GitHub Pages Setup

### 2.1 Create the repo

1. On GitHub, create a new repo named `origins-planner` (or whatever you want)
2. Add `index.html` and `firestore.rules` to it
3. Add this `README.md` too

### 2.2 Enable GitHub Pages

1. Go to the repo → **Settings → Pages**
2. Source: **Deploy from a branch**
3. Branch: `main`, folder: `/ (root)`
4. Click **Save**

GitHub will give you a URL like `https://yourusername.github.io/origins-planner`

### 2.3 Point origins.misterwilson.org at it

Since your domain is on Cloudflare (you've done this for other subdomains):

1. Log in to Cloudflare → your domain → **DNS**
2. Add a new CNAME record:
   - **Type:** CNAME
   - **Name:** origins
   - **Target:** `yourusername.github.io`
   - **Proxy status:** DNS only (gray cloud) — GitHub Pages needs this
3. Save

Then in GitHub, go to your repo → **Settings → Pages → Custom domain**, type `origins.misterwilson.org`, and click Save. GitHub will provision an SSL cert automatically (takes a few minutes).

You should also add a `CNAME` file to the root of your repo containing just:
```
origins.misterwilson.org
```
This prevents GitHub from forgetting your custom domain when you push.

---

## Part 3: First-Time Use

### 3.1 Upload the Origins schedule CSV

1. Go to `origins.misterwilson.org`
2. Click the **Upload** tab
3. Make sure the year selector says **Origins 2026**
4. Either drag-and-drop your CSV from tabletop.events, or click the upload zone
5. Watch the upload progress — it stores events in Firestore in chunks of 200

After the first upload, all three of you can browse the schedule immediately.

### 3.2 Import your Google Sheet preferences

This lets you bring in the TRUE flags you've already set in your spreadsheet without redoing them.

1. In Google Sheets, go to **File → Download → Comma Separated Values (.csv)**
2. In the app, select your name (Jake/Dan/Tim) from the user buttons
3. Click **Upload → Import My Preferences**
4. Select the CSV you just downloaded
5. The importer looks for a column with mostly TRUE values (your preference column), then cross-references event numbers to flag those events for your account

The preferences show up as "★ Pref" badges and can be filtered with the **My Prefs** button.

### 3.3 Start favoriting

Click the ★ on any event card to favorite it. Favorites are saved to Firestore immediately and sync across all browsers. Tim's favorites on his laptop will show up on your phone.

---

## Part 4: CSV Version Management

Every time the Origins schedule gets updated on tabletop.events, you can re-upload the CSV:

1. Download the fresh CSV
2. Go to Upload tab
3. Drop it in

The app will:
- **Compare** it against the current version
- **Alert you** if any events you favorited were removed (with the name and who cared)
- **List** what's new
- **Store** the version in Firestore history so you can see what changed and when

The version history is visible at the bottom of the Upload tab. Old event data is replaced, but favorites are preserved — they're stored by event number, so if an event keeps its number across CSV updates, the favorite survives.

---

## Part 5: Building a Schedule with Claude

When you're ready to plan your schedule:

1. Select your name
2. Go to the **Schedule** tab
3. Click **Export for Claude →**
4. A `.txt` file downloads with all your favorites sorted by day and time, including group overlap info
5. Upload that file here and ask Claude to help you build a conflict-free schedule

The Schedule tab also shows a ⚠ conflict warning if two of your favorites overlap on the same day.

---

## Part 6: At the Convention

The **Today** tab is designed for on-site use:
- Automatically highlights the current day of the week
- Shows your personal schedule for the day at the top
- Shows what everyone else in the group has flagged below that
- All session links open directly to the tabletop.events registration page
- Works on mobile

The day buttons let you switch between days if you're planning ahead or checking someone else's schedule.

---

## Data Model Reference

```
Firestore:
│
├── years/
│   └── {year}/                    ← metadata: eventCount, latestVersion, etc.
│       ├── chunks/
│       │   └── {0..N}/            ← 200 events per chunk, full event objects
│       └── versions/
│           └── {timestamp}/       ← upload history: filename, count, diff summary
│
├── userFavorites/
│   └── {jake|dan|tim}/            ← one doc per user
│       └── { "2026": { "4264": Timestamp, ... }, "2027": {...} }
│
└── userPrefs/
    └── {jake|dan|tim}/            ← one doc per user
        └── { "2026": ["4264","510",...], "2027": [...] }
```

**Event object structure** (stored in chunks):
```javascript
{
  num:        "4264",          // Event Number from CSV — primary key
  name:       "Highlander - The Board Game",
  type:       "Board Game",
  desc:       "...",
  publisher:  "Crazy Pawn Games",
  complexity: "Medium",        // Low / Medium / High
  taught:     true,            // Rules Taught = Yes
  tournament: false,
  duration:   150,             // minutes
  room:       "Hall C - 2530",
  day:        "Wednesday",
  time:       "12:00 PM",
  startUTC:   "2026-06-17T16:00:00",  // for conflict detection
  url:        "https://tabletop.events/...",
  bgg:        "https://boardgamegeek.com/...",
  flags: {    // auto-computed on upload
    norse: false, greek: false, myths: false, zombies: false,
    cthulhu: false, apocalypse: false, archery: false,
    deckbuilder: false, workerplacement: false, legacy: false,
    coop: false, platonic: false, premiere: false
  }
}
```

---

## Updating for Future Years

When Origins 2027 rolls around:

1. In `index.html`, find the `CONV_DATES` object and add the 2027 entry with the correct start date
2. Upload the 2027 CSV via the Upload tab (select "Origins 2027" in the year dropdown)
3. The 2026 data stays intact — years are completely independent in Firestore

No code changes needed beyond the date update.

---

## Troubleshooting

**"No events yet" after upload:**
Check the browser console for Firestore permission errors. The most common cause is a Firestore rules typo or the project ID being wrong in FIREBASE_CONFIG.

**Favorites not persisting:**
Open browser devtools → Network tab — look for failed Firestore writes. Usually means the API key is restricted too aggressively or the rules blocked the write.

**CSV import finds 0 preferences:**
Your sheet's TRUE column might have a different header. Try renaming the column header to `TRUE` before exporting, or check that the values are actually the text `TRUE` (not a checkbox or `1`).

**Custom domain not working:**
Make sure GitHub Pages has the custom domain set AND the CNAME file is in your repo root. Cloudflare proxy (orange cloud) must be OFF for GitHub Pages — use DNS only.

**"REPLACE_WITH_YOUR_API_KEY" error in console:**
You didn't replace the placeholder values in FIREBASE_CONFIG. Do that and push again.
