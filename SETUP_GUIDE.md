# Masters 2026 Golf Comp - Setup Guide

## Quick Start

1. **Set up Firebase** (free tier is more than enough)
2. **Configure the app** with your Firebase credentials
3. **Add golfers** to the tournament field
4. **Set up live scoring** (optional - can use manual entry)
5. **Share the link** with your competition participants

---

## Step 1: Firebase Setup

### Create a Firebase Project
1. Go to [Firebase Console](https://console.firebase.google.com)
2. Click **"Add project"**
3. Name it something like `masters-golf-comp`
4. Disable Google Analytics (not needed) and click **Create**

### Enable Realtime Database
1. In your project, go to **Build → Realtime Database**
2. Click **Create Database**
3. Choose a location closest to you (e.g., `europe-west1` for Ireland/UK)
4. Start in **Test Mode** (you can lock it down later)

### Get Web App Config
1. Go to **Project Settings** (gear icon in top left)
2. Under "Your apps", click the **Web** icon (`</>`)
3. Register your app (name doesn't matter)
4. Copy the config values and paste them into the `firebaseConfig` object in `index.html`

```javascript
const firebaseConfig = {
    apiKey: "AIzaSy...",
    authDomain: "your-project.firebaseapp.com",
    databaseURL: "https://your-project-default-rtdb.europe-west1.firebasedatabase.app",
    projectId: "your-project"
};
```

### (Optional) Secure the Database
After the competition, or before going live, update the database rules:
```json
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```
For production, you could restrict writes, but for a small friend group, test mode is fine.

---

## Step 2: Admin Login

- Default admin credentials: **admin / admin123**
- Change the password immediately after first login
- The admin can manage everything from the Admin panel

---

## Step 3: Add the Tournament Field

1. Login as admin
2. Go to **Admin → Manage Golfers**
3. Paste the list of golfers (one per line)
4. Click **Save Golfer List**

### Masters 2026 Field (example - update with actual field)
You can paste the official Masters field here. The field is usually announced a few weeks before the tournament.

---

## Step 4: Configure Competition Settings

Go to **Admin → Settings** and configure:
- **Competition Name**: e.g. "Masters 2026"
- **Picks per team**: 10 (default)
- **Best to count**: 5 (default)
- **Default score for missing golfers**: +10 (default)
- **WD/DQ penalty**: +20 (default)
- **Picks Deadline**: Set this to before the first tee time on Thursday

---

## Step 5: Live Scoring Setup

You have three options for scoring:

### Option A: Manual Score Entry
- Go to **Admin → Manual Scores**
- Enter scores for each golfer as the tournament progresses
- Simple but requires manual updates

### Option B: ESPN API Direct (Recommended)

This fetches scores directly from ESPN's public JSON API. No Google Sheets or Apps Script needed — it's the simplest and fastest method.

#### How It Works
- The app calls ESPN's scoreboard API every 60 seconds (configurable)
- Scores are parsed from the JSON response and saved to Firebase
- Cut/WD/DQ statuses are auto-detected and penalties applied

#### Setup Steps

1. **Find the ESPN Tournament ID**
   - Go to the ESPN leaderboard page for the tournament (e.g., `espn.com/golf/leaderboard`)
   - The event ID is in the URL, e.g. `espn.com/golf/leaderboard/_/tournament/401811941`
   - **Masters 2026 = `401811941`**

2. **Configure in the App**
   - Login as admin
   - Go to **Admin → Live Scoring**
   - Set mode to **ESPN API (Direct)**
   - Enter the Tournament ID (e.g. `401811941`)
   - Set refresh interval (60 seconds is recommended)
   - Set the cut penalty (extra strokes added on top of the cut line, default 5)

3. **Test It**
   - Click **Test ESPN Fetch** to verify it connects
   - You'll see the tournament name, player count, and a sample leaderboard
   - It also shows name matching — how ESPN names map to your golfer list
   - If names don't match, adjust them in **Manage Golfers**

4. **Save and Go**
   - Click **Save Scoring Config**
   - Scores will auto-update every 60 seconds while the page is open

#### Notes
- This uses ESPN's public (but undocumented) scoreboard API — it's been stable for years
- The cut line is auto-detected from ESPN data; cut players get the cut line score + penalty
- Fuzzy name matching handles case differences, accents, and "Last, First" formats
- **Manual scores always override** auto-fetched scores
- The API only works while someone has the app open in a browser (it's client-side)

### Option C: Auto-Fetch from ESPN via Google Sheets (Fallback)

Use this as a backup if the ESPN API stops working. This uses a Google Sheet with `IMPORTHTML` to scrape the ESPN leaderboard, with a checkbox-toggle mechanism to force refreshes, and serves the data as JSON via an Apps Script web app.

#### How It Works
1. The Google Sheet has two `IMPORTHTML` formulas — one at row 2 and one at row 200
2. A checkbox in A1 toggles which formula has the **valid** ESPN URL and which has a **broken** URL
3. An Apps Script timer toggles the checkbox every few minutes, which forces Google Sheets to re-fetch the ESPN data
4. The Apps Script also serves the valid data as JSON via a `doGet()` web endpoint
5. The Masters Comp app fetches that JSON endpoint and updates scores in Firebase

#### Step 5a: Create the Google Sheet

1. Create a new Google Sheet
2. In cell **A1**, insert a **checkbox** (Insert → Checkbox)
3. In cell **A2**, paste this formula:
   ```
   =IMPORTHTML("https://www.espn.com/golf/leaderboard?t=1234567890", "table")
   ```
4. In cell **A200**, paste the same formula but with a **broken** URL:
   ```
   =IMPORTHTML("https://www.espn.com/golf/leaderboar?t=1234567890", "table")
   ```
   (Note: "leaderboar" is intentionally misspelled — this is the broken version)

The idea: when the checkbox is toggled, the script swaps which cell has the working URL and which has the broken one, with a new timestamp. This forces Google Sheets to re-fetch the data.

#### Step 5b: Add the Apps Script

1. Go to **Extensions → Apps Script**
2. Delete any existing code and paste this:

```javascript
// Toggle the checkbox and swap the IMPORTHTML formulas to force a refresh
function uncheckbox() {
  const sheetName = "Sheet1";
  const ss = SpreadsheetApp.getActive();
  const sheet = ss.getSheetByName(sheetName);

  if (!sheet) {
    Logger.log("Sheet not found: " + sheetName);
    return;
  }

  const box = sheet.getRange("A1");
  const boxValue = box.getValue();

  // Toggle checkbox value
  const newValue = !boxValue;
  box.setValue(newValue);

  const timestamp = new Date().getTime(); // Unique value to bust cache
  const baseUrl1 = "https://www.espn.com/golf/leaderboar";  // Broken URL
  const baseUrl2 = "https://www.espn.com/golf/leaderboard"; // Working URL

  let formula1 = '=IMPORTHTML("' + baseUrl1 + '?t=' + timestamp + '", "table")';
  let formula2 = '=IMPORTHTML("' + baseUrl2 + '?t=' + timestamp + '", "table")';

  if (newValue === true) {
    // A1 is TRUE: broken URL at A2, working URL at A200
    sheet.getRange("A2").setFormula(formula1);
    sheet.getRange("A200").setFormula(formula2);
  } else {
    // A1 is FALSE: working URL at A2, broken URL at A200
    sheet.getRange("A2").setFormula(formula2);
    sheet.getRange("A200").setFormula(formula1);
  }

  SpreadsheetApp.flush();
  Logger.log("Formulas updated with timestamp: " + timestamp);
}

// Check if a row has valid (non-empty, non-error) data
function isValidData(row) {
  return row.some(cell => cell !== "" && cell !== "#N/A");
}

// Get the valid table data (whichever IMPORTHTML succeeded)
function getValidTableData() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Sheet1");
  
  // Data ranges for the two IMPORTHTML tables
  // Adjust these ranges based on the size of the ESPN leaderboard table
  const dataA2 = sheet.getRange("B3:L158").getValues();     // First table data
  const dataA200 = sheet.getRange("B201:L356").getValues();  // Second table data
  const headersA2 = sheet.getRange("B2:L2").getValues();     // Headers for first table
  const headersA200 = sheet.getRange("B200:L200").getValues(); // Headers for second table

  let validData = [];
  let headers = [];

  const boxValue = sheet.getRange("A1").getValue();

  // Check which table has valid data based on checkbox state
  const hasValidDataA2 = dataA2.some(row => isValidData(row));
  if (hasValidDataA2 && boxValue === false) {
    validData = dataA2;
    headers = headersA2;
  }

  const hasValidDataA200 = dataA200.some(row => isValidData(row));
  if (hasValidDataA200 && boxValue === true) {
    validData = dataA200;
    headers = headersA200;
  }

  if (validData.length === 0) {
    Logger.log("No valid data found");
    return null;
  }

  // Return headers + data rows
  const result = [headers[0], ...validData];
  return result;
}

// Web app entry point — called when a GET request hits the deployed URL
function doGet(e) {
  const validData = getValidTableData();

  if (!validData) {
    return ContentService.createTextOutput("No valid data found.")
      .setMimeType(ContentService.MimeType.TEXT);
  }

  return ContentService.createTextOutput(JSON.stringify(validData))
    .setMimeType(ContentService.MimeType.JSON);
}
```

#### Step 5c: Deploy the Apps Script

1. Click **Deploy → New Deployment**
2. Type: **Web app**
3. Execute as: **Me**
4. Who has access: **Anyone**
5. Click **Deploy** and **copy the URL** — you'll need this for the app

#### Step 5d: Set Up the Auto-Refresh Trigger

1. In Apps Script, go to **Triggers** (clock icon on left sidebar)
2. Click **+ Add Trigger**
3. Configure:
   - Function: `uncheckbox`
   - Event source: Time-driven
   - Type: Minutes timer
   - Interval: **Every 5 minutes** (or every minute during tournament)
4. Click **Save**

#### Step 5e: Configure in the Masters Comp App

1. Login as admin
2. Go to **Admin → Live Scoring**
3. Set mode to **Auto-Fetch (ESPN via Google Sheets)**
4. Paste the Apps Script web app URL
5. Set refresh interval (60 seconds recommended)
6. Click **Test Fetch** to verify it works — it will show:
   - Whether columns (PLAYER, SCORE, THRU) were detected
   - Name matching preview (how ESPN names map to your golfer list)
7. Click **Save Scoring Config**

#### Important Notes
- The `uncheckbox` trigger runs every 5 minutes on Google's servers, refreshing the ESPN data
- The app fetches from the Google Sheet every 60 seconds (configurable)
- During the tournament, scores will be at most ~5 minutes behind ESPN
- The **Test Fetch** button shows name matching — verify ESPN names match your golfer list
- The app does fuzzy name matching (case-insensitive, handles accents, "Last, First" format)
- If a name doesn't match, you can manually adjust the golfer name in "Manage Golfers" to match ESPN
- **Manual scores always override** auto-fetched scores (use Admin → Manual Scores for corrections)
- Update the **Cut Line** and **Cut Penalty** settings after the cut is made on Friday

---

## Step 6: Hosting

### Option A: Simple File Share
- Just share the `index.html` file (with Firebase config filled in)
- Anyone can open it in a browser
- Works offline with cached data

### Option B: GitHub Pages (Free)
1. Create a GitHub repository
2. Upload `index.html`
3. Enable GitHub Pages in Settings
4. Share the URL (e.g., `yourusername.github.io/masters-comp`)

### Option C: Firebase Hosting (Free)
1. Install Firebase CLI: `npm install -g firebase-tools`
2. Run `firebase init hosting`
3. Deploy: `firebase deploy`
4. Share the URL

### Option D: Google Cloud Storage (like the old golf comp)
1. Create a GCS bucket
2. Upload `index.html`
3. Make it public
4. Share the URL

---

## How It Works

### For Players
1. Register/login
2. Pick 10 golfers from the tournament field
3. Submit picks before the deadline
4. Watch the leaderboard update live during the tournament

### Scoring
- Each golfer's score is their tournament score relative to par
- **Best 5 out of 10** golfers count for your team
- Lower is better (under par = negative = good)
- Cut/WD/DQ golfers get penalty scores
- Unranked golfers default to +10

### Example
If your 5 counting golfers finish at: -8, -4, -2, +1, +3
Your team score = -8 + -4 + -2 + 1 + 3 = **-10**

---

## Differences from the Old Golf Comp

| Feature | Old (GolfComp.html) | New (MastersComp) |
|---------|---------------------|-------------------|
| Data storage | Hardcoded in HTML | Firebase (live sync) |
| User accounts | None | Login/Register |
| Pick management | Fixed in code | Users pick their own teams |
| Score source | Google Sheets only | Auto-fetch + manual override |
| Admin panel | None | Full admin controls |
| Mobile support | Basic | Fully responsive |
| Real-time sync | Page refresh | Firebase real-time |
| Export/Import | None | Full backup/restore |
