# 📬 EcoInvite — Deployment Guide

A retro Windows 98-themed invite & RSVP web app. Real-time, multi-user, deployable as a single HTML file.

---

## 🔥 Step 1 — Create a Firebase Project (free)

1. Go to [https://console.firebase.google.com](https://console.firebase.google.com)
2. Click **"Add project"** → give it a name (e.g. `ecoinvite`) → Continue
3. Disable Google Analytics (optional) → **Create project**

---

## 🔐 Step 2 — Enable Anonymous Authentication

1. In the Firebase console, go to **Build → Authentication**
2. Click **"Get started"**
3. Under **Sign-in method**, click **Anonymous** → toggle **Enable** → Save

This gives every user a persistent ID automatically — no sign-up needed.

---

## 🗄️ Step 3 — Create Firestore Database

1. Go to **Build → Firestore Database**
2. Click **"Create database"**
3. Choose **"Start in test mode"** (allows all reads/writes for 30 days — good for development)
4. Pick a region close to you → **Enable**

> ⚠️ **Before going live**, replace test mode rules with the secure rules below.

### Firestore Security Rules (paste into the Rules tab)

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Users can only read/write their own profile
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }

    // Anyone authenticated can read ecosystems (to join by code)
    // Only members can write to their own member entry
    match /ecosystems/{ecoCode} {
      allow read: if request.auth != null;
      allow create: if request.auth != null;

      match /members/{memberId} {
        allow read: if request.auth != null;
        allow write: if request.auth != null && request.auth.uid == memberId;
      }
    }

    // Invites: sender can create, recipient can update (rsvp only)
    match /invites/{inviteId} {
      allow create: if request.auth != null && request.auth.uid == request.resource.data.fromUid;
      allow read:   if request.auth != null &&
                       (request.auth.uid == resource.data.fromUid ||
                        request.auth.uid == resource.data.toUid);
      allow update: if request.auth != null &&
                       request.auth.uid == resource.data.toUid &&
                       request.resource.data.keys().hasOnly(['rsvp']);
    }
  }
}
```

---

## ⚙️ Step 4 — Get Your Firebase Config

1. In Firebase console, go to **Project Settings** (gear icon ⚙️)
2. Scroll to **"Your apps"** → click **"Add app"** → choose **Web (</>)**
3. Give it a nickname (e.g. `ecoinvite-web`) → **Register app**
4. Copy the `firebaseConfig` object — it looks like:

```js
const firebaseConfig = {
  apiKey: "AIzaSy...",
  authDomain: "ecoinvite-xxxxx.firebaseapp.com",
  projectId: "ecoinvite-xxxxx",
  storageBucket: "ecoinvite-xxxxx.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abcdef"
};
```

---

## 📝 Step 5 — Paste Config Into the HTML File

Open `index.html` and find this block near the top:

```js
const FIREBASE_CONFIG = {
  apiKey:            "PASTE_YOUR_API_KEY_HERE",
  authDomain:        "PASTE_YOUR_AUTH_DOMAIN_HERE",
  projectId:         "PASTE_YOUR_PROJECT_ID_HERE",
  storageBucket:     "PASTE_YOUR_STORAGE_BUCKET_HERE",
  messagingSenderId: "PASTE_YOUR_MESSAGING_SENDER_ID_HERE",
  appId:             "PASTE_YOUR_APP_ID_HERE"
};
```

Replace each `"PASTE_YOUR_..."` value with the values from your Firebase config.

> ✅ It's safe to expose these values in your frontend code — Firebase uses Security Rules (above) to control actual data access.

---

## 🚀 Step 6 — Deploy to GitHub + Netlify

### GitHub
```bash
git init
git add index.html README.md
git commit -m "Initial EcoInvite deploy"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/ecoinvite.git
git push -u origin main
```

### Netlify
1. Go to [https://netlify.com](https://netlify.com) → **Add new site → Import from Git**
2. Connect your GitHub account → select the `ecoinvite` repo
3. Build settings: leave **blank** (no build command, no publish directory needed — Netlify will serve `index.html` directly)
4. Click **Deploy site**

Your app will be live at a URL like `https://ecoinvite-abc123.netlify.app`

---

## 🔒 Step 7 — Authorize Your Netlify Domain in Firebase

1. In Firebase console → **Authentication → Settings → Authorized domains**
2. Click **"Add domain"**
3. Paste your Netlify URL (e.g. `ecoinvite-abc123.netlify.app`)
4. Save

This allows Firebase Auth to work from your Netlify domain.

---

## 📁 File Structure

```
ecoinvite/
├── index.html    ← The entire app (rename ecoinvite-firebase.html to index.html)
└── README.md     ← This file
```

---

## 🗃️ Firestore Data Structure

```
users/
  {uid}/
    name:     "Alex"
    ecoCode:  "XK7P2M"
    ecoName:  "The Crew"
    uid:      "firebase-uid"

ecosystems/
  {ecoCode}/
    name:      "The Crew"
    createdBy: "firebase-uid"
    createdAt: timestamp
    members/
      {uid}/
        name:     "Alex"
        uid:      "firebase-uid"
        joinedAt: timestamp

invites/
  {inviteId}/
    fromUid:   "firebase-uid"
    fromName:  "Alex"
    toUid:     "firebase-uid"
    toName:    "Jordan"
    ecoCode:   "XK7P2M"
    title:     "Birthday Bash 🎉"
    date:      "2025-03-15"
    time:      "19:00"
    location:  "123 Main St"
    details:   "Bring snacks!"
    rsvp:      "pending" | "yes" | "no"
    createdAt: timestamp
```

---

## 🆓 Firebase Free Tier Limits (Spark plan)

| Resource | Free limit |
|---|---|
| Firestore reads | 50,000 / day |
| Firestore writes | 20,000 / day |
| Firestore storage | 1 GB |
| Auth users | Unlimited |
| Hosting bandwidth | 10 GB / month |

More than enough for personal use or small groups.

---

## 🛠️ Troubleshooting

**"Firebase config not found"** → Make sure you replaced all `PASTE_YOUR_...` values in the HTML.

**"Permission denied" errors** → Check that you've published the Security Rules in Firestore.

**Auth not working on Netlify** → Make sure you added your Netlify domain to Firebase Authorized Domains (Step 7).

**Users can't see each other's invites** → Both users must be in the same ecosystem (same `ecoCode`). Invites are filtered by `ecoCode` AND `toUid`.
