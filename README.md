# Borang Pencalonan MPP 2026

A lightweight, serverless nomination form system for the Student Representative Council (Majlis Perwakilan Pelajar) election at KKTM / TVETMARA Petaling Jaya.

Candidates submit their nomination through a clean public form, and the committee reviews everything through an auth-protected admin panel with one-click CSV export. No backend server to maintain, no hosting bill, just static pages on GitHub Pages talking to Firebase.

> **Live form:** `https://amcaqt.github.io/<your-repo>/`
> **Admin panel:** `https://amcaqt.github.io/<your-repo>/admin.html`

---

## Features

**Public nomination form**
- 14 contestable positions (President down to Exco Technical)
- Matric number validation with fixed `DFI` prefix, digits only
- Digit-only phone input
- Experience and motivation fields
- Required declaration checkbox
- Honeypot field for basic bot/spam filtering (no captcha needed)
- Responsive, works on phone and desktop

**Admin panel**
- Login-protected using Firebase Authentication
- Persistent session (stay logged in across refreshes)
- Live table of all nominations, newest first
- Quick stats (total candidates, positions filled)
- Filter by position
- One-click CSV export (Excel-friendly, UTF-8 with BOM)

---

## Tech Stack

| Layer | Tech |
|-------|------|
| Frontend | Plain HTML, CSS, vanilla JavaScript (no framework, no build step) |
| Database | Cloud Firestore |
| Auth | Firebase Authentication (Email/Password) |
| Hosting | GitHub Pages (static) |
| SDK | Firebase Web SDK v10 via ESM CDN |

No bundler, no npm install, no server. Everything runs in the browser.

---

## Project Structure

```
.
├── index.html      # Public nomination form
├── admin.html      # Auth-protected admin dashboard + CSV export
├── logo.png        # TVETMARA PJ logo
└── README.md
```

---

## Security Design

This is the part worth understanding, because a static public site collecting personal data (name, matric, phone) needs to be handled properly.

**Firebase config keys are public by design.** They ship inside the HTML and that is expected. Security does not come from hiding keys, it comes from Firestore Security Rules.

The rules enforce a strict split:

- **Anyone can `create`** a nomination (that is the whole point of a public form), but only if the payload is well-formed. Empty or oversized junk documents get rejected.
- **Only authenticated users can `read`.** The public cannot pull the list of candidates and their phone numbers. That data is only visible after logging into the admin panel.
- **Nobody can `update` or `delete`** from the browser. Records are immutable client-side.

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /pencalonan/{docId} {
      allow create: if isValid();
      allow read:   if request.auth != null;
      allow update, delete: if false;
    }
    function isValid() {
      let d = request.resource.data;
      return d.keys().hasAll(['nama','matrik','kelas','semester','telefon','jawatan','createdAt'])
        && d.nama is string && d.nama.size() > 0 && d.nama.size() < 150
        && d.matrik is string && d.matrik.size() < 30
        && d.jawatan is string;
    }
  }
}
```

**On top of the rules, the client also handles:**
- HTML escaping on all rendered fields to prevent stored XSS from form input
- Proper CSV cell escaping to prevent broken columns and CSV injection
- A honeypot field to silently drop simple spam bots

---

## Setup / Deployment

Want to run your own copy? Here is the full flow.

### 1. Firebase project
1. Create a project at [console.firebase.google.com](https://console.firebase.google.com)
2. Register a **Web app** (`</>`) and copy the `firebaseConfig` block
3. Paste your config into the `firebaseConfig` object in both `index.html` and `admin.html`

### 2. Firestore
1. **Firestore Database** > Create database
2. Start in **Production mode**
3. Location: `asia-southeast1` (Singapore, closest for Malaysia)
4. Go to the **Rules** tab, paste the rules from the Security section above, and **Publish**

### 3. Authentication (for the admin panel)
1. **Authentication** > Get started
2. **Sign-in method** > enable **Email/Password** only
3. **Users** > Add user, create your admin account (email + password)

### 4. Deploy to GitHub Pages
1. Push `index.html`, `admin.html`, and `logo.png` to a public repo
2. **Settings** > **Pages** > deploy from branch `main`, folder `/ (root)`
3. Your form goes live at `https://<username>.github.io/<repo>/`

---

## Usage

**Candidates** open the form link, fill in their details, pick a position, and submit.

**Committee** opens `admin.html`, logs in with the admin account, reviews the table, filters by position if needed, and hits **Export CSV** to hand data to the college.

---

## Positions

President, Deputy President, Secretary 1, Secretary 2, Treasurer, and the Exco lineup: Kerohanian, Akademik, Sukan, Disiplin, Kebudayaan, Kebajikan, Usahawan, Media, Teknikal.

---

## Notes

- Built for KKTM / TVETMARA Petaling Jaya MPP election, session 2026.
- Free tier friendly. Firestore's free quota comfortably covers a single college election.
- Storage was intentionally left out to avoid the Blaze billing requirement. Candidate photos, if any, are collected separately.
