# Firebase Auth & User Administration — Complete Setup Guide

## What This System Implements

A full user authentication lifecycle for a Vite + React + Firebase app:

- Email/password registration with email verification
- Google OAuth (redirect flow, not popup — avoids COOP/COEP blocking)
- User account documents in Firestore (`userAccounts` collection) with status tracking
- Admin console — approve, deactivate, reactivate, grant/revoke admin, send password reset
- Password change from within the app (re-authenticates first)
- Login tracking — `firstLoginAt`, `lastLoginAt`, `totalLogins`, `activityCount`
- Bootstrap admin — hardcoded email self-creates with admin + approved status on first sign-in

## Step 1 — Firebase Project Setup

1. Go to <https://console.firebase.google.com/>
2. Create a new project (or use existing)
3. Authentication → Sign-in method → Enable:
   - Email/Password
   - Google
4. Authentication → Settings → Authorized domains:
   - Add your local/dev/prod domains
5. Firestore Database:
   - Create database
   - Start in production mode (recommended)
6. Project Settings → Your apps:
   - Register Web app
   - Copy Firebase config values for your `.env`

## Step 2 — Install Dependencies

```bash
npm install firebase
```

If you use React Router and context-based auth:

```bash
npm install react-router-dom
```

## Step 3 — Environment Variables (Vite)

Create `.env`:

```bash
VITE_FIREBASE_API_KEY=...
VITE_FIREBASE_AUTH_DOMAIN=...
VITE_FIREBASE_PROJECT_ID=...
VITE_FIREBASE_STORAGE_BUCKET=...
VITE_FIREBASE_MESSAGING_SENDER_ID=...
VITE_FIREBASE_APP_ID=...
```

> Vite requires `VITE_` prefix for client-exposed vars.

## Step 4 — Firebase Initialization

Create `src/lib/firebase.ts`:

```ts
import { initializeApp } from 'firebase/app'
import { getAuth, GoogleAuthProvider } from 'firebase/auth'
import { getFirestore } from 'firebase/firestore'

const firebaseConfig = {
  apiKey: import.meta.env.VITE_FIREBASE_API_KEY,
  authDomain: import.meta.env.VITE_FIREBASE_AUTH_DOMAIN,
  projectId: import.meta.env.VITE_FIREBASE_PROJECT_ID,
  storageBucket: import.meta.env.VITE_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: import.meta.env.VITE_FIREBASE_MESSAGING_SENDER_ID,
  appId: import.meta.env.VITE_FIREBASE_APP_ID,
}

export const app = initializeApp(firebaseConfig)
export const auth = getAuth(app)
export const db = getFirestore(app)
export const googleProvider = new GoogleAuthProvider()
```

## Step 5 — Firestore Data Model

Collection: `userAccounts`
Document ID: Firebase Auth `uid`

Suggested document shape:

```json
{
  "uid": "string",
  "email": "user@example.com",
  "displayName": "Jane User",
  "isAdmin": false,
  "status": "pending",
  "emailVerified": false,
  "createdAt": "serverTimestamp",
  "approvedAt": null,
  "approvedBy": null,
  "deactivatedAt": null,
  "deactivatedBy": null,
  "reactivatedAt": null,
  "reactivatedBy": null,
  "firstLoginAt": null,
  "lastLoginAt": null,
  "totalLogins": 0,
  "activityCount": 0
}
```

Status values:

- `pending` (registered, awaiting approval)
- `approved` (can access protected areas)
- `deactivated` (explicitly disabled by admin)

## Step 6 — Auth Flow (Email/Password)

Registration flow:

1. Create account with `createUserWithEmailAndPassword`
2. Send verification email with `sendEmailVerification`
3. Create/merge `userAccounts/{uid}` with `status: "pending"`
4. Require email verification before protected app access

Login flow:

1. Sign in with `signInWithEmailAndPassword`
2. Reload current user to refresh `emailVerified`
3. Read Firestore profile (`userAccounts/{uid}`)
4. Gate access on:
   - `emailVerified === true`
   - `status === "approved"`
5. Update login metrics (`firstLoginAt`, `lastLoginAt`, `totalLogins`)

## Step 7 — Google OAuth (Redirect, Not Popup)

Use redirect APIs to avoid popup blockers and COOP/COEP issues:

- Start: `signInWithRedirect(auth, googleProvider)`
- Resolve after return: `getRedirectResult(auth)`

After successful Google auth:

1. Upsert `userAccounts/{uid}`
2. If new user, set `status: "pending"` by default
3. For bootstrap admin email, set:
   - `isAdmin: true`
   - `status: "approved"`
4. Record login metrics

## Step 8 — Bootstrap Admin Strategy

In app config, define a known admin email (example: founder/owner):

```ts
const BOOTSTRAP_ADMIN_EMAIL = 'owner@yourdomain.com'
```

On first sign-in for that email:

- If no `userAccounts/{uid}` exists, create it with:
  - `isAdmin: true`
  - `status: "approved"`
  - `approvedAt: serverTimestamp()`
  - `approvedBy: "bootstrap"`

> Restrict this to one trusted email and remove/lock down once real admin provisioning exists.

## Step 9 — Admin Console Capabilities

Admin-only screen should list users and provide actions:

- Approve user:
  - Set `status: "approved"`, `approvedAt`, `approvedBy`
- Deactivate user:
  - Set `status: "deactivated"`, `deactivatedAt`, `deactivatedBy`
- Reactivate user:
  - Set `status: "approved"`, `reactivatedAt`, `reactivatedBy`
- Grant admin:
  - Set `isAdmin: true`
- Revoke admin:
  - Set `isAdmin: false`
- Send password reset:
  - `sendPasswordResetEmail(auth, email)`

For safety:

- Prevent admin from demoting/deactivating themselves unless a second admin exists
- Confirm destructive actions with modal dialogs
- Log admin actions (optional but recommended)

## Step 10 — Change Password (In-App)

Firebase requires recent login for sensitive operations:

1. Re-authenticate user (email/password credential)
2. Call `updatePassword(currentUser, newPassword)`

Typical flow:

- Prompt for current password + new password
- Reauth with `reauthenticateWithCredential`
- On success, update password
- Handle `auth/requires-recent-login` gracefully

## Step 11 — Route Protection Rules

Typical protected-route logic:

- Not signed in → redirect to login
- Signed in but email not verified → verification screen
- Verified but status `pending` → waiting approval screen
- Status `deactivated` → account disabled screen
- Approved → allow app
- Admin route also requires `isAdmin === true`

## Step 12 — Firestore Security Rules (Baseline)

Use server-side enforcement where possible (Cloud Functions/custom claims), but as a baseline:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /userAccounts/{userId} {
      allow read: if request.auth != null && (
        request.auth.uid == userId ||
        get(/databases/$(database)/documents/userAccounts/$(request.auth.uid)).data.isAdmin == true
      );

      allow create: if request.auth != null && request.auth.uid == userId;

      allow update: if request.auth != null && (
        request.auth.uid == userId ||
        get(/databases/$(database)/documents/userAccounts/$(request.auth.uid)).data.isAdmin == true
      );

      allow delete: if false;
    }
  }
}
```

> Tighten rules for field-level restrictions (for example, users should not self-set `isAdmin` or `status`).

## Step 13 — Recommended UX Messages

- Pending approval: “Your account is awaiting admin approval.”
- Deactivated: “Your account has been deactivated. Contact support.”
- Email not verified: “Please verify your email before continuing.”
- Password changed: “Your password was updated successfully.”

## Step 14 — Testing Checklist

- Register via email/password
- Verify email and confirm gated access before verification
- Confirm new users are `pending`
- Approve user in admin console and verify access granted
- Deactivate user and verify blocked access
- Google redirect sign-in works in local + deployed environments
- Bootstrap admin is created only for configured email
- Password reset email sends successfully
- Password change requires correct current password
- Non-admin user cannot access admin routes/actions

## Step 15 — Production Hardening

- Move admin authorization to custom claims + backend enforcement
- Add audit logging for admin actions
- Add rate limits / abuse protection on auth endpoints
- Add monitoring/alerts for unusual auth events
- Add account recovery/support process

---

This setup gives you a complete, production-oriented authentication and user administration lifecycle while remaining straightforward to implement in a Vite + React + Firebase stack.
