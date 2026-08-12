# Making the Duty Board shared across all browsers

Until now every piece of data — roster, availability, assignments, published schedules,
PINs, settings — lived in `localStorage`, which is private to **one browser on one device**.
Publishing a schedule on your laptop was invisible to everyone else, because there was no
server in the picture at all.

The app now reads and writes a **Firebase Realtime Database** instead. Every visitor talks to
the same database, and changes push out live — a phone with the page open updates within about
a second of an officer publishing, without a refresh.

You need to do the five steps below **once** before uploading the file.

---

## 1. Create the Firebase project

1. Go to <https://console.firebase.google.com> and sign in with a Google account.
2. **Add project** → name it (e.g. `cer-duty-board`) → you can turn Google Analytics **off**.

## 2. Create the Realtime Database

1. In the left sidebar: **Build → Realtime Database** → **Create Database**.
2. Pick the location closest to you.
3. When it asks for rules, choose **Start in test mode** for now — step 4 replaces them.

> Make sure you pick **Realtime Database**, not **Firestore**. They are different products and
> this app uses Realtime Database.

## 3. Register a web app and copy the config

1. Sidebar gear icon → **Project settings** → scroll to **Your apps** → click the **`</>`** (web) icon.
2. Give it any nickname. **Do not** tick "Also set up Firebase Hosting".
3. Firebase shows a `firebaseConfig = { ... }` block. Copy the values.
4. Open `cerupdate.html`, find `const FIREBASE_CONFIG` near the top of the `<script>` block
   (around line 1256), and paste your values in:

```js
const FIREBASE_CONFIG = {
  apiKey: "AIzaSy...",
  authDomain: "cer-duty-board.firebaseapp.com",
  databaseURL: "https://cer-duty-board-default-rtdb.firebaseio.com",
  projectId: "cer-duty-board",
  storageBucket: "cer-duty-board.appspot.com",
  messagingSenderId: "123456789012",
  appId: "1:123456789012:web:abc123"
};
```

`databaseURL` is the important one — it only appears in the config after step 2, so if you
don't see it, go back and create the database first.

## 4. Set the database rules

Realtime Database → **Rules** tab → replace everything with this → **Publish**:

```json
{
  "rules": {
    "duty-board": {
      ".read": true,
      ".write": true,
      "$record": {
        ".validate": "newData.isString() && newData.val().length <= 500000"
      }
    }
  }
}
```

Test mode rules expire after 30 days and the site would silently stop saving, so don't skip this.

## 5. Upload

Upload the edited `cerupdate.html` to your web host (`index.html` is a byte-identical copy —
upload whichever name your host expects). Both files in this repo are kept in sync.

---

## Moving your existing data over

The first time you open the site **on the browser that already has your roster and past
schedules**, it will notice the database is empty and ask:

> This browser has N saved Duty Board records, but the shared database is empty.
> Upload them now so everyone else can see them?

Click **OK** and everything is copied up. Do this from the browser with the most complete
data, and do it **before** anyone else visits — once someone else writes a record, the
database is no longer empty and the prompt won't appear again.

If you'd rather start clean, click Cancel.

---

## What to expect

- The **● Live** indicator in the header now reflects a real connection. It shows **⚠ Offline**
  if the browser can't reach the database.
- Changes from other people appear automatically. No refresh, no "hard reload" instructions
  for your members.
- If the network drops, the page keeps showing the last schedule it saw (it's still mirrored to
  `localStorage`) and writes resume when the connection returns.
- If `FIREBASE_CONFIG` is left blank, the app behaves exactly as it did before — this-browser-only.
  That's the fallback, not the goal.

## Cost

Firebase's free Spark plan allows 100 simultaneous connections and 1 GB stored. This app stores
a few hundred KB at most and a squad will never approach 100 people with the page open at once.
No card required.

## One security caveat, so it isn't a surprise

The rules above let anyone who knows the site URL read and write the data. That is what makes
the app work without accounts — members submit availability with no login.

The consequence: **the officer PIN and member PINs are readable** by anyone technical enough to
open the database URL, and are stored as plain text. They keep honest people out of the officer
tabs; they are not real security. Don't reuse a password that matters anywhere else.

Closing that gap properly means moving the PIN check to a server (a Cloudflare Worker or Firebase
Auth + Cloud Function), which is a larger change. Worth doing if this ever holds anything
sensitive — say the word and I'll build it.
