# SSO Integration Guide for Child PWAs

This document describes how embedded PWA apps (iframes) can receive and use Firebase SSO credentials from the parent Zwartbol Industries portfolio.

---

## How It Works

The parent portfolio (`index.html`) authenticates users via Firebase Auth (Google SSO). When a user is signed in, the parent broadcasts auth credentials to every embedded iframe via `postMessage`. Child PWAs listen for this message to receive the user's identity and a Firebase ID token.

---

## Message Format

The parent sends a message of type `admin-auth` to all embedded iframes:

```javascript
{
  type: 'admin-auth',
  isAdmin: boolean,           // Whether the user has admin privileges
  key: string | null,         // Admin password (legacy, for backward compat)
  sso: {                      // null when no SSO user is signed in
    uid: string,              // Firebase user ID
    email: string,            // User email address
    displayName: string,      // User display name
    photoURL: string,         // User avatar URL
    idToken: string,          // Firebase ID token (JWT) — use for Firestore/API auth
  } | null,
}
```

---

## Receiving Auth in a Child PWA

Add this listener in your child PWA's JavaScript (before any auth-dependent logic):

```javascript
// --- SSO Auth Receiver ---
let currentUser = null;
let isAdmin = false;

window.addEventListener('message', (event) => {
  const data = event.data;
  if (!data || data.type !== 'admin-auth') return;

  isAdmin = !!data.isAdmin;

  if (data.sso) {
    currentUser = {
      uid: data.sso.uid,
      email: data.sso.email,
      displayName: data.sso.displayName,
      photoURL: data.sso.photoURL,
      idToken: data.sso.idToken,
    };
    onUserSignedIn(currentUser);
  } else {
    currentUser = null;
    onUserSignedOut();
  }
});

function onUserSignedIn(user) {
  // Update UI: show user avatar, enable user-specific features
  console.log('User signed in:', user.displayName, user.email);
}

function onUserSignedOut() {
  // Update UI: hide user-specific features
  console.log('User signed out');
}
```

---

## Using the ID Token with Firebase (Firestore)

If your child PWA has its own Firebase Firestore instance (sharing the same Firebase project), you can use `signInWithCustomToken` or, more simply, pass the ID token to your backend. For direct Firestore access from the child PWA using the **same Firebase project**:

```html
<!-- Load Firebase SDKs (same version as parent) -->
<script src="https://www.gstatic.com/firebasejs/10.12.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.12.0/firebase-auth-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore-compat.js"></script>

<script>
  // Initialize with the SAME Firebase config as the parent
  const firebaseApp = firebase.initializeApp({
    apiKey: "AIzaSyAqRWVgti-CuXzr5AkhDKmPlu7iA2MQpUg",
    authDomain: "arnout-s-homelab.firebaseapp.com",
    projectId: "arnout-s-homelab",
    storageBucket: "arnout-s-homelab.firebasestorage.app",
    messagingSenderId: "258011834485",
    appId: "1:258011834485:web:5afb1efaa303b4dd93f03c",
  });
  const db = firebaseApp.firestore();
  const auth = firebaseApp.auth();

  // Listen for SSO credentials from parent
  window.addEventListener('message', async (event) => {
    const data = event.data;
    if (!data || data.type !== 'admin-auth') return;

    if (data.sso && data.sso.idToken) {
      // The child PWA shares the same Firebase project, so the user's
      // auth state will automatically be available if the parent and child
      // share the same auth domain. However, since iframes are on different
      // origins (github.io subpaths), we use signInWithCredential:
      try {
        const credential = firebase.auth.GoogleAuthProvider.credential(data.sso.idToken);
        await auth.signInWithCredential(credential);
        console.log('Signed in via parent SSO');
      } catch (err) {
        console.warn('SSO sign-in failed, using token for display only:', err.message);
      }
    } else {
      // User signed out
      try { await auth.signOut(); } catch {}
    }
  });
</script>
```

---

## Storing Per-User Data in Firestore

The Firestore rules support per-user data under `/users/{userId}/app_data/`:

```javascript
// Save user preferences (only the authenticated user can read/write their own data)
async function saveUserData(key, value) {
  if (!currentUser) return;
  await db.collection('users').doc(currentUser.uid)
    .collection('app_data').doc(key)
    .set(value, { merge: true });
}

// Load user data
async function loadUserData(key) {
  if (!currentUser) return null;
  const doc = await db.collection('users').doc(currentUser.uid)
    .collection('app_data').doc(key).get();
  return doc.exists ? doc.data() : null;
}
```

---

## Showing User Info in the UI

```javascript
function onUserSignedIn(user) {
  // Example: show avatar + name in a header bar
  const header = document.getElementById('user-info');
  if (header) {
    header.innerHTML = `
      <img src="${user.photoURL}" alt=""
           style="width:28px;height:28px;border-radius:50%;border:1px solid #f59e0b80;">
      <span style="font-family:'JetBrains Mono',monospace;font-size:12px;color:#a1a1aa;">
        ${user.displayName || user.email}
      </span>
    `;
  }
}
```

---

## Security Notes

1. **Never trust `isAdmin` blindly** on the client side. Use it for UI toggling only. Sensitive operations should verify the ID token server-side.
2. **ID tokens expire** after ~1 hour. The parent refreshes and re-sends tokens automatically when the iframe loads or auth state changes.
3. The `key` field (admin password) is provided for **backward compatibility** with apps that used the old password-based auth. New apps should use the `sso` object instead.
4. Always validate the `event.origin` in production if you want to restrict which parent can send auth messages.

---

## Quick Checklist for Adding SSO to a Child PWA

- [ ] Add `window.addEventListener('message', ...)` listener for `admin-auth` messages
- [ ] Extract `sso` object for user identity (uid, email, displayName, photoURL)
- [ ] Use `idToken` if you need authenticated Firestore access
- [ ] Update UI to show/hide user-specific features based on auth state
- [ ] Store per-user data under `/users/{uid}/app_data/` in Firestore
- [ ] Test sign-in and sign-out flows by toggling auth in the parent portfolio
