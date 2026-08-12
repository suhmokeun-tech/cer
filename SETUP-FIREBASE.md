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
    },
    "duty-board-dev": {
      ".read": true,
      ".write": true,
      "$record": {
        ".validate": "newData.isString() && newData.val().length <= 500000"
      }
    },
    "secrets": {
      "$env": {
        "officer": {
          ".write": true,
          "hash": { ".read": false, ".validate": "newData.isString() && newData.val().matches(/^[0-9a-f]{64}$/)" },
          "salt": { ".read": true, ".validate": "newData.isString() && newData.val().length <= 64" }
        },
        "members": {
          "$memberId": {
            ".write": true,
            "hash": { ".read": false, ".validate": "newData.isString() && newData.val().matches(/^[0-9a-f]{64}$/)" },
            "salt": { ".read": true, ".validate": "newData.isString() && newData.val().length <= 64" }
          }
        }
      }
    },
    "unlock-attempts": {
      "$env": {
        "officer": {
          ".read": false,
          ".write": "newData.val() === root.child('secrets/' + $env + '/officer/hash').val()"
        },
        "members": {
          "$memberId": {
            ".read": false,
            ".write": "newData.val() === root.child('secrets/' + $env + '/members/' + $memberId + '/hash').val()"
          }
        }
      }
    }
  }
}
```

Test mode rules expire after 30 days and the site would silently stop saving, so don't skip this.

The `duty-board-dev` block and the `secrets`/`unlock-attempts` trees are what make PIN checking
happen on Firebase's servers instead of in the browser (see the caveat section below) — they're
required even if you only ever use `duty-board` (production). `duty-board-dev` is only used by the
`cerupdate-dev.html` sandbox copy for testing changes without touching live data.

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

## How PIN checking works now

The rules above let anyone who knows the site URL read and write most of the data (the
`duty-board`/`duty-board-dev` trees) — that's still what makes the app work without accounts,
so members can submit availability with no login.

PINs are the exception. They're hashed in the browser (SHA-256, with a random salt per PIN)
before anything is sent, and the actual match check happens inside the rules above, evaluated on
Firebase's servers — the stored hash has `.read: false` and is never sent to any browser, ever.
Checking a PIN attempt works by trying to write the computed hash to `unlock-attempts/...`; the
rules only let that write through if it matches the real (unreadable) hash, so a successful write
means "correct PIN" and a permission error means "wrong PIN." No PIN, and no PIN hash, ever
appears in the app's network traffic.

One caveat worth knowing: *setting or changing* a PIN is still governed by the same open-write
trust model as the rest of the app (anyone who can reach the site can already rewrite the roster
today), so this closes the "PINs are readable off the wire" gap without adding real accounts.
Locking down who's allowed to administer the board would be a separate, bigger project — worth
doing if this ever holds something more sensitive than a duty roster.
