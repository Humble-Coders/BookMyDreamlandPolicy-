# Booking Cancellation + Refund — Reception Desktop App Integration

This document is the **step-by-step guide for the Reception Desktop App** to cancel a guest's booking and trigger the Razorpay refund.

You do **not** call the guest mobile app's `cancelBooking` Cloud Function — that one requires a guest's Firebase Auth token and refuses cross-user calls. Instead you call a **new server-to-server function** built specifically for reception: **`cancelBookingByReception`**.

Everything below is what the reception app needs to do. The Cloud Function handles all the hard parts (eligibility, idempotency, Razorpay call, Firestore writes, hotel contact snapshot). The reception app just collects input, calls the function, and shows the result.

---

## 1. The flow at a glance

```
┌───────────────────────┐   1. Staff clicks "Cancel + Refund" on a booking
│  Reception Desktop UI │
└──────────┬────────────┘
           │ 2. Preview eligibility + refund quote (read-only Firestore + local math)
           ▼
┌───────────────────────┐
│  Confirm dialog        │   3. Staff picks refund mode (POLICY / FULL / FIXED),
│  + typed confirmation  │      enters reason, retypes booking ID to confirm.
└──────────┬────────────┘
           │ 4. Reception app calls the Firebase Callable (signed-in staff,
           │    reception:true custom claim) — OR (alt) HTTPS sibling with
           │    a Google-signed service-account ID token.
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  cancelBookingByReception  (onCall, asia-south1)                 │
│  Body: { userId, groupBookingId, reason, refundMode, ... }       │
└──────────┬───────────────────────────────────────────────────────┘
           │  Inside the function (you don't do any of this):
           │   • Loads bookings + payment doc
           │   • Locks via payments.refundIntentKey  (double-refund-proof)
           │   • Calls Razorpay refund API
           │   • Flips booking statuses to CANCELLED, stamps RECEPTION audit
           ▼
┌───────────────────────┐
│  Response with         │   5. Show "Refund initiated — ₹X,XXX" toast.
│  refundId + amount     │      Refund settles per Razorpay's normal timeline
└───────────────────────┘      (instant for UPI, T+5–7 days for cards).
```

---

## 2. Prerequisites — one-time setup

The function supports **two authentication models**. Pick one. Approach A (Firebase Auth + custom claim) is **what the function as deployed today expects** — start there. Approach B (service-account ID token) is documented for future use if the reception backend is rebuilt as a non-Firebase server.

### Approach A — Firebase Auth user + `reception: true` custom claim ✅ active

This is the model the current `cancelBookingByReception.ts` validates. It uses the same `onCall` pattern as the rest of the codebase (`createBooking`, `cancelBooking`, etc.).

#### A1. Create a dedicated Firebase Auth account per staff member

One Firebase Auth account per reception staff member (e.g. `reception+priya@dreamland.internal`, `reception+anil@dreamland.internal`). Email/password sign-in is fine — reception desktop machines are trusted hardware.

> Do **not** share one account between staff. Per-staff accounts give you the `cancelledByReceptionUserId` audit trail for free and let you revoke one staff member's access without resetting everyone.

#### A2. Provision the `reception: true` custom claim

Run once per staff account from a trusted admin script (NOT from the reception desktop app — Admin SDK belongs on a server you control):

```ts
import * as admin from "firebase-admin";
admin.initializeApp({ credential: admin.credential.applicationDefault() });

const user = await admin.auth().getUserByEmail("reception+priya@dreamland.internal");
await admin.auth().setCustomUserClaims(user.uid, { reception: true });
console.log("Claim set on", user.uid);
```

The claim shows up in the next ID token the user mints (after re-sign-in or `getIdToken(true)`). The Cloud Function rejects any caller without `claims.reception === true` with `permission-denied: "Reception role required."`.

#### A3. Install Firebase SDK on the reception desktop app

```bash
npm install firebase
```

Sign in once at app boot, persist the session (Firebase SDK does this by default), and use the Callable wrapper to call the function:

```ts
import { initializeApp } from "firebase/app";
import { getAuth, signInWithEmailAndPassword } from "firebase/auth";
import { getFunctions, httpsCallable } from "firebase/functions";

const app = initializeApp({ /* same firebaseConfig as the mobile apps */ });
const auth = getAuth(app);
const functions = getFunctions(app, "asia-south1");

// At login screen:
await signInWithEmailAndPassword(auth, staffEmail, staffPassword);

// To call the function:
const callable = httpsCallable<CancelBookingByReceptionRequest, CancelBookingByReceptionResponse>(
  functions,
  "cancelBookingByReception",
);
const result = await callable(body);
// result.data is the response object
```

That's the entire auth model. No service-account keys to manage.

#### A4. Initialize Admin SDK (only on the reception backend, for the read-only preview)

If your reception app has a backend that runs queries on behalf of the desktop UI (recommended), still initialize Admin SDK there for the read-only preview in Step 3 below:

```ts
import * as admin from "firebase-admin";
admin.initializeApp({ credential: admin.credential.applicationDefault() });
const db = admin.firestore();
```

The actual cancel writes go through the Cloud Function — do not write to bookings/payments docs directly from the reception app for cancellation. Admin SDK is for reads + the unrelated scratch-card redemption flow.

---

### Approach B — Service-account ID token (alternative; not active)

Use this if the reception backend is rebuilt as a standalone Node service that doesn't want to sign in as a Firebase Auth user. **The current function does NOT accept this auth model** — it would need a separate `onRequest` HTTPS sibling (`cancelBookingByReceptionHttp`) that verifies a Google-signed ID token. Documented here so the design is on record.

#### B1. Get a service-account key

The server team creates a GCP service account `reception-desktop@<project-id>.iam.gserviceaccount.com` with the custom IAM role `roles/dreamland.reception` (permission: `cloudfunctions.functions.invoke` on the HTTPS sibling only).

You receive a **JSON key file**. Store it:

- ✅ In the reception machine's OS keychain or a secrets vault.
- ✅ In an env var (`GOOGLE_APPLICATION_CREDENTIALS=/path/to/key.json`) read at process start.
- ❌ **NOT** in the desktop app installer, source repo, or anywhere the desktop client ships to staff machines uninspected.

#### B2. Install dependencies

```bash
npm install google-auth-library firebase-admin
```

#### B3. Mint a Google ID token and call the HTTPS sibling

```ts
import { GoogleAuth } from "google-auth-library";

const auth = new GoogleAuth();
const audience = "https://asia-south1-<project-id>.cloudfunctions.net/cancelBookingByReceptionHttp";
const client   = await auth.getIdTokenClient(audience);
const headers  = await client.getRequestHeaders();
// headers.Authorization == "Bearer eyJhbGc..."

const res = await fetch(audience, {
  method: "POST",
  headers: { ...headers, "Content-Type": "application/json" },
  body: JSON.stringify(body),
});
```

The `cancelledByReceptionUserId` field in the request body carries the per-staff identity (since the service-account identity itself is shared across staff in this model).

#### B4. Trade-offs vs Approach A

| Aspect | Approach A (active) | Approach B (alt) |
|---|---|---|
| Auth source | Firebase Auth user + custom claim | GCP service account + IAM role |
| Per-staff identity | Native — auth UID is the staff | Manual — must trust the body's `cancelledByReceptionUserId` |
| Key rotation | Password reset, no key files | JSON key rotation, vault management |
| Function type | `onCall` (current) | `onRequest` (would need new sibling) |
| Best for | Reception desktop app with a UI login | Headless / cron / non-UI backend service |

---

## 3. Step-by-step: cancel a booking

### Step 1 — Load the booking group from Firestore

The reception app already has the `groupBookingId` from its booking list. Load every booking in the group:

```ts
const bookingsSnap = await db.collection("bookings")
  .where("groupBookingId", "==", groupBookingId)
  .get();

if (bookingsSnap.empty) {
  showError("Booking group not found.");
  return;
}

const bookings = bookingsSnap.docs.map(d => ({ id: d.id, ...d.data() }));
const firstBooking = bookings[0];
const userId         = firstBooking.userId;          // pass to the function
const paymentOrderId = firstBooking.paymentOrderId;  // for preview only
const hotelId        = firstBooking.hotelId;
```

### Step 2 — Check eligibility (client-side, before showing the dialog)

Block these states **in the UI** so staff don't even see the cancel button:

| Booking field | Block when |
|---|---|
| `status` | not `"CONFIRMED"` (already `CANCELLED`, `CHECKED_IN`, `COMPLETED`, `NO_SHOW` → no cancel) |
| `cancellationLockedAt` | `> 0` and `now - cancellationLockedAt < 10 * 60 * 1000` → the guest just hit cancel from the mobile app, give their flow 10 minutes to complete |
| `paymentOrderId` | empty → no payment record, no refund possible |

```ts
const now = Date.now();
const LOCK_GRACE_MS = 10 * 60 * 1000;

for (const b of bookings) {
  if (b.status !== "CONFIRMED") {
    showError(`Booking ${b.id} is ${b.status} — cannot cancel.`);
    return;
  }
  const lockedMs = b.cancellationLockedAt?.toMillis?.() ?? 0;
  if (lockedMs > 0 && now - lockedMs < LOCK_GRACE_MS) {
    showError("Guest is currently cancelling from the app. Wait 10 minutes and retry.");
    return;
  }
}
```

### Step 3 — Compute the refund preview (so staff see what will happen)

The function will recompute server-side — this is purely so the confirm dialog can show numbers. Mirror the logic from `functions/src/cancellationRefundCalculator.ts`:

```
For each booking:
  hoursBeforeCheckIn = (checkInDate - now) / 3_600_000
  Look up rooms/{roomCategoryId} → freeBefore, refundPercent

  If freeCancellation == true AND hoursBeforeCheckIn >= freeBefore:
      refundPaise = advancePaidAmountPaise                 // 100%
      bucket = "FULL_REFUND"
  Else if hoursBeforeCheckIn >= 24:
      refundPaise = floor(advancePaidAmountPaise * refundPercent / 100)
      bucket = "PARTIAL_REFUND"
  Else:
      refundPaise = 0
      bucket = "NO_REFUND"

totalRefundPaise = sum(refundPaise across bookings)
```

> **Important — the preview is advisory only.** The Cloud Function recomputes against current room policies and caps the result by Razorpay's available refund amount. Always display the **function's** response amount on the final success toast, not your local preview.

### Step 4 — Show the confirm dialog

Required fields in the dialog:

- **Booking summary** — guest name, phone, hotel, room categories, check-in date, total advance paid.
- **Refund mode picker:**
  - `"POLICY"` (default) — apply the cancellation policy quote from Step 3.
  - `"FULL"` (manager-only) — refund the entire `advancePaidAmountPaise` regardless of policy.
  - `"FIXED"` (manager-only) — refund exactly the entered amount (rupees → paise; e.g. ₹500 = 50000 paise).
- **Reason** — required, min 10 chars. Free-text, max 500 chars.
- **Typed confirmation** — staff retypes the booking ID (`GB_20260610_001`) to enable the Cancel button. Prevents wrong-row mis-clicks.

Gate `FULL` and `FIXED` behind a manager-role check in the reception app. The Cloud Function will accept them from any reception caller — the role gate is your responsibility.

### Step 5 — Call `cancelBookingByReception`

#### Approach A (active) — Firebase Callable SDK

The signed-in staff member's ID token (with `reception: true` claim) is attached automatically by the Firebase SDK. No manual header work.

```ts
import { getFunctions, httpsCallable, FunctionsError } from "firebase/functions";

const functions = getFunctions(app, "asia-south1");
const callable = httpsCallable<
  CancelBookingByReceptionRequest,
  CancelBookingByReceptionResponse
>(functions, "cancelBookingByReception");

const body: CancelBookingByReceptionRequest = {
  userId,                                        // from Step 1
  groupBookingId,
  reason: dialogReason.trim(),
  refundMode: dialogRefundMode,                  // "POLICY" | "FULL" | "FIXED"
  fixedRefundPaise:
    dialogRefundMode === "FIXED" ? dialogFixedPaise : undefined,
  cancelledByReceptionUserId: currentStaffId,    // e.g. "reception_priya@dreamland"
};

try {
  const result = await callable(body);
  const res = result.data;
  showSuccess(
    `Refund of ₹${(res.totalRefundPaise / 100).toFixed(2)} initiated. ` +
    `Razorpay ref: ${res.refundId}.`
  );
} catch (e) {
  const err = e as FunctionsError;
  // err.code is "functions/<code>" — e.g. "functions/permission-denied"
  handleError(err.code, err.message);
}
```

#### Approach B (alternative HTTPS sibling) — Service-account ID token

Same body, but call the HTTPS sibling with a Google-signed ID token (see section 2 Approach B for the token mint):

```ts
import { GoogleAuth } from "google-auth-library";

const auth = new GoogleAuth();
const audience = "https://asia-south1-<project-id>.cloudfunctions.net/cancelBookingByReceptionHttp";
const client   = await auth.getIdTokenClient(audience);
const headers  = await client.getRequestHeaders();

const res = await fetch(audience, {
  method: "POST",
  headers: { ...headers, "Content-Type": "application/json" },
  body: JSON.stringify(body),
});

if (!res.ok) {
  const err = await res.json().catch(() => ({}));
  handleError(res.status, err.code, err.message);
  return;
}
const result = await res.json();
```

Cache the GoogleAuth client; the token auto-refreshes when near expiry.

### Step 6 — Handle the response

**Success (Callable `result.data` / HTTPS 200):**

```jsonc
{
  "groupBookingId":     "GB_20260610_001",
  "totalRefundPaise":   300000,
  "perRoom": [
    { "bookingId": "B_001", "refundPaise": 150000, "bucket": "FULL" },
    { "bookingId": "B_002", "refundPaise": 150000, "bucket": "FULL" }
  ],
  "refundId":           "rfnd_OqAbc123XyZ",
  "finalRefundStatus":  "INITIATED",
  "cancellationSource": "RECEPTION",
  "refundMode":         "POLICY"
}
```

- `finalRefundStatus: "INITIATED"` → refund queued at Razorpay. Done.
- `finalRefundStatus: ""` (empty) → zero-refund case. Bookings are still CANCELLED. Show "Booking cancelled. No refund per policy."
- `refundId` is the Razorpay refund ID — store/show for support lookup.

**Errors:** Callable errors come back as `FunctionsError` with `err.code` like `"functions/<code>"`. HTTPS-sibling errors are HTTP status + JSON body. Same code names either way:

| Code | Callable `err.code` | Show to staff | Retry safe? |
|---|---|---|---|
| `unauthenticated` | `functions/unauthenticated` | "Auth expired. Sign in again." | No |
| `permission-denied` | `functions/permission-denied` | "Your account lacks reception permission." | No |
| `invalid-argument` | `functions/invalid-argument` | "Missing field: \<field\>." | No |
| `not-found` | `functions/not-found` | "Booking or payment record not found." | No |
| `failed-precondition` | `functions/failed-precondition` | "Cannot cancel: \<reason\>." (e.g. `ALREADY_CHECKED_IN`, `ALREADY_CANCELLED`, `no_payment_link`) | No |
| `resource-exhausted` | `functions/resource-exhausted` | "Too many cancellations this minute. Wait 60s." | Yes, after 60s |
| `internal` | `functions/internal` | "Refund failed. Safe to retry." | Yes |

**The function is idempotent on `groupBookingId`** — retrying after a 500 is safe. If the refund actually went through on Razorpay but Firestore failed to commit, the retry will detect the in-flight state, re-query Razorpay, and stitch the existing refund into Firestore without creating a second one.

### Step 7 — Confirm the guest sees it

The guest's mobile app reads bookings live. Within seconds of the function returning, the booking flips to **CANCELLED** in their My Stays tab, with the refund status badge ("Refund initiated"). You do not need to notify them separately — Firestore real-time listeners handle it.

For SMS/email — the function also snapshots `cancellationHotelPhoneSnapshot` and `cancellationHotelEmailSnapshot` onto each booking so the existing notification trigger (`onBookingCancelled`) fires the standard guest-facing cancellation email/SMS automatically. No reception-side action needed.

---

## 4. Refund timeline (what to tell the guest)

| Payment method | Refund visible to guest |
|---|---|
| UPI | Usually within minutes (Razorpay "optimum" speed) |
| Net banking | 2–4 working days |
| Debit / credit card | 5–7 working days |
| Wallets (Paytm/PhonePe/Mobikwik) | 24–48 hours |

The Razorpay webhook updates `cancellationRefundStatus` from `"INITIATED"` → `"COMPLETED"` when the refund settles. The guest's app reflects it automatically.

---

## 5. What the function writes (for your awareness)

You don't write any of these — the Cloud Function does — but the reception app may need to read them later (refund history view, support lookups):

**On each `bookings/{bookingId}` in the group:**

| Field | Value |
|---|---|
| `status` | `"CANCELLED"` |
| `cancellationSource` | `"RECEPTION"` |
| `cancelledAt` | server timestamp |
| `cancellationLockedAt` | `0` (cleared) |
| `cancellationRefundAmountPaise` | per-booking refund split |
| `cancellationRefundPolicyPercent` | policy % that was applied |
| `cancellationRefundId` | Razorpay refund ID |
| `cancellationRefundStatus` | `"INITIATED"` → `"COMPLETED"` after webhook |
| `cancellationHotelPhoneSnapshot` | from `hotels/{hotelId}.contactPhone` |
| `cancellationHotelEmailSnapshot` | from `hotels/{hotelId}.contactEmail` |
| `cancellationReason` | your `reason` string |
| `cancelledByReceptionUserId` | your `cancelledByReceptionUserId` string |
| `cancellationRefundMode` | `"POLICY"` \| `"FULL"` \| `"FIXED"` |

**On `payments/{paymentOrderId}`:**

| Field | Value |
|---|---|
| `refundIntentKey` | `groupBookingId` (acts as the dedupe lock) |
| `refundId` | Razorpay refund ID |
| `refundAmountPaise` | total refunded |
| `refundStatus` | `"INITIATED"` → `"COMPLETED"` after webhook |
| `refundCreatedAt` | server timestamp |

---

## 6. Edge cases & gotchas

- **Don't cancel from Firestore directly.** Reception has Admin SDK and could in theory write `status: "CANCELLED"` directly, but this **skips the Razorpay refund** and breaks the `refundIntentKey` dedupe. Always go through the function.
- **Don't call Razorpay refund directly from the reception app either.** Same reason — the Firestore lock + the booking-status flip + the hotel-contact snapshot all happen atomically inside the function. Splitting them produces orphaned refunds.
- **Multi-room group bookings cancel together.** There is no per-room cancel. If the guest wants to cancel only one room of a 3-room booking, that's a product decision not currently supported — confirm with the guest before pressing the button.
- **Already-cancelled groups.** If a staff member double-clicks Cancel, the second call returns the same successful response (idempotent replay). Safe to show success either way; don't treat the replay as an error.
- **Guest cancelled it first.** If the guest hits cancel from the mobile app and reception also hits cancel, only one succeeds — the other gets `failed_precondition: already_cancelled`. Show "Already cancelled by guest" and refresh the booking row.
- **Goodwill `FULL` or `FIXED` after check-in date passed.** Allowed (this is the whole point of manager overrides). The function bypasses the `past_check_in` block when `refundMode` ≠ `"POLICY"`. Log the override locally for the manager audit trail — the function logs it server-side too.

---

## 7. Server team — implementation status

### Approach A (active) — Firebase Auth + custom claim

- [x] Add `"RECEPTION"` to `CancellationSource` in `functions/src/types.ts`.
- [x] Add `ReceptionRefundMode` + `CancelBookingByReceptionRequest` / `Response` interfaces in `functions/src/types.ts`.
- [x] Implement `functions/src/cancelBookingByReception.ts` as `onCall`, region `asia-south1`, secret `RAZORPAY_KEY_SECRET`. Checks `claims.reception === true`, calls the same Phase 1–5 pipeline as `cancelBooking` with `cancellationSource: "RECEPTION"`.
- [x] Export from `functions/src/index.ts`.
- [ ] **Deploy:** `firebase deploy --only functions:cancelBookingByReception`.
- [ ] Provision the `reception: true` custom claim on each reception staff's Firebase Auth account (see § 2 A2).
- [ ] Add a Cloud Logging metric on `cancellationSource == "RECEPTION" && refundMode != "POLICY"` so finance gets alerted on goodwill overrides.

### Approach B (alternative) — HTTPS sibling with service-account ID token

Not yet built. Implement only if the reception team's backend cannot or should not sign in as a Firebase Auth user.

- [ ] Create `functions/src/cancelBookingByReceptionHttp.ts` as `onRequest`, region `asia-south1`, secret `RAZORPAY_KEY_SECRET`. Verify the bearer ID token via `google-auth-library`, check IAM role / email allowlist, then delegate to the same shared logic the Callable uses.
- [ ] Extract `cancelBookingByReception.ts` Phases 1–5 into `runRefundPipeline(args)` in `functions/src/cancellationPipeline.ts` so both entrypoints share one battle-tested code path. (Defer the extraction until Approach B is actually built — premature today.)
- [ ] Create custom IAM role `roles/dreamland.reception` with `cloudfunctions.functions.invoke` on the HTTPS sibling only.
- [ ] Create the reception service account and ship its JSON key to the reception team via a secure channel (1Password / vault / signed email — never source control).
- [ ] Export the HTTPS sibling from `functions/src/index.ts`.
