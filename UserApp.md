# Haazir User App — README (v1 POC)

A build-ready specification for the **Haazir User** web app. This README captures flows, data shapes, security, and backend hooks for a fast Firebase POC that can later be reused by mobile apps.

---

## 0) Scope & Goals

* Let users discover services, request help **now**, confirm **Start/Finish via OTP**, pay (UPI QR or COD), and **rate** the partner.
* Responsive PWA first; keep the flow short and reliable.

---

## 1) Auth & Onboarding

* **Phone OTP only** (Firebase Auth). Minimal profile at signup.
* **Progressive profiling** (keep friction low):

  * Signup: **OTP + first name** only.
  * At first booking: collect **address** (GPS prefill; user adds landmark/flat no.) and allow **Save as Home/Work**.
  * After first completed job: optional **alt phone** + **email** (skippable two-field sheet).

**users/{uid}** (minimal shape)

```json
{
  "name": "Asha",
  "phone": "+91XXXXXXXXXX",
  "email": null,
  "altPhone": null,
  "addresses": [
    { "label": "Home", "line1": "", "pincode": "", "lat": 0, "lng": 0, "landmark": "", "isDefault": true }
  ],
  "createdAt": "...",
  "updatedAt": "..."
}
```

---

## 2) Core User Flows (v1)

### A) Discover & Request

1. Browse **Catalogue** (admin-managed) or use **Search**.
2. Open **Service Detail**: title, short description, images, inclusions/exclusions, SLA minutes, **Starting at ₹X**.
3. Confirm **location** (GPS or pincode).
4. (Optional) Set **price ceiling**.
5. Tap **Find Now** → creates `jobs/{jobId}` with `status: "REQUESTED"`. Backend begins matching rounds.

### B) Matching & Assignment

* "Searching nearby partners…" live status.
* When assigned, show **partner card**: name, ★ overall rating (count), admin **description** (bio), ETA.

### C) Arrival & Start Confirmation (Start OTP)

* App shows a **Start OTP** when job is **ASSIGNED**.
* User shares OTP with partner → partner enters on their app → state becomes **IN_PROGRESS**.

### D) Finish & Payment (Finish OTP → Pay)

* When work ends, app shows **Finish OTP**.
* User shares OTP → partner enters → job enters **AWAITING_PAYMENT**.
* **Payment choice** (on partner app, but user-visible instructions):

  * **UPI:** scan QR from partner’s device and pay.
  * **COD:** hand over cash.
* After payment is confirmed (manual confirm in v1), job becomes **COMPLETED**.

### E) Rate & Review

* Post-completion prompt: **1–5★ + optional comment**.
* Updates partner’s **overall ratingAvg/ratingCnt**; review visible to partner (read-only).

### F) Orders & Support

* **My Orders**: list of jobs with status, timestamps, partner details; rate if pending.
* **Cancel** allowed until **IN_PROGRESS** (configurable).

---

## 3) UI / Routes

```
/                     // Landing + location prompt
/services             // Catalogue grid
/service/:id          // Service details page
/search               // Type-ahead search
/checkout             // Service summary + address + price ceiling + Find Now
/track/:jobId         // Live status + Start/Finish OTP cards
/orders               // Order history + rating
/login                // OTP auth
```

---

## 4) Pricing Display (v1)

* Catalogue & detail: **“Starting at ₹X”** (basePrice), note: “Includes up to Y minutes. Parts extra.”
* Checkout estimate breakdown:

  * **Visit fee** (base)
  * **Service fee** (platform fee)
  * **Tax** (on service fee)
  * **Estimated total** (rounded)
* If payout escalates during matching, update the estimate; if delta > threshold, prompt user to confirm.

**jobs/{jobId}.pricing** (example)

```json
{
  "basePaise": 29900,
  "feePaise": 1900,
  "taxPaise": 342,
  "estimatePaise": 32142,
  "escalationDeltaPaise": 0
}
```

---

## 5) Payment (v1, simple)

* Payment happens **after Finish OTP**.
* Partner chooses **UPI** (QR shown on partner app) or **COD**.
* User instructions: “Scan to pay” or “Pay cash to partner.”
* Backend records payment method & timestamp; v1 uses manual confirmation.

**jobs/{jobId}.payment**

```json
{
  "method": "UPI" | "COD" | null,
  "status": "PENDING" | "PAID" | "FAILED" | "TIMEOUT",
  "receiverVPA": "haazir@icici",
  "collectedAt": "..."
}
```

---

## 6) Firestore Model (user-facing reads/writes)

**Read**

* `/services/*` (catalogue)
* Public subset of `/partners/{partnerId}`: `name`, `ratingAvg`, `ratingCnt`, `description`
* Own `/jobs/{jobId}`

**Write**

* Create job → `jobs/{jobId}`: `{ userId, serviceId, lat, lng, priceCeiling?, status: "REQUESTED" }`
* Cancel job (before IN_PROGRESS) → `status: "CANCELLED"` (callable or guarded update)
* Rate once on completed job → `userRating: { stars, comment }`

---

## 7) Security Rules (snapshot)

* Users can **read/write only their own jobs**.
* Users can set `userRating` **once** and only when `status == COMPLETED`.
* Users cannot modify server-owned fields: OTPs, payouts, partner assignment, payment, earnings.
* `/services/*` readable by all; `/partners/*` expose only non-sensitive public fields.

---

## 8) Cloud Functions touched by User journey

* **dispatch/matching**: on job create → broadcast offers → assign first accept.
* **state transitions**: via partner callables (`startJob`, `completeJob`, `confirmPayment`).
* **ratings**: on `userRating` added → update partner aggregates & copy review.

(No user callable needed in v1 unless you use a callable for cancel.)

---

## 9) Realtime & Notifications

* Job screen uses `onSnapshot` on `jobs/{jobId}` for live status.
* (Optional) Web push for “Assigned”, “Partner en route”, “Completed”.

---

## 10) Performance & UX

* Use `srcset`/lazy-load for service images; prefer WebP/AVIF.
* Keep listeners scoped to the active job; unsubscribe on unmount.
* Copy for trust: “You’ll pay after service completion (UPI/Cash).”

---

## 11) Acceptance Checklist (v1)

* [ ] OTP + first name signup works.
* [ ] Catalogue shows title/description/images; service detail repeats **Starting at ₹X** and inclusions/exclusions.
* [ ] Checkout collects address if none saved; creates job.
* [ ] Live tracking from SEARCHING → ASSIGNED → EN_ROUTE → IN_PROGRESS.
* [ ] Start OTP appears only when ASSIGNED; Finish OTP only when IN_PROGRESS.
* [ ] After Finish OTP, payment instructions show; partner collects via UPI QR or COD; job completes.
* [ ] Rating prompt shows once; posting updates partner’s overall rating and review list.
* [ ] Orders page lists recent jobs with correct status and rating state.
* [ ] Rules prevent editing server-controlled fields.

---

## 12) Future (non-blocking)

* Scheduler bookings (choose time), promo codes, richer payments (aggregator with webhook), saved addresses, multi-language, chat/calls masking, invoice PDFs, disputes.

---

**Status:** Spec complete. Safe to implement User web app for v1 POC.

---

## 🔄 SYNC UPDATE — Payment & State Flow (Aligned with User App v1)

### Updated Job State Machine

```
SEARCHING → ASSIGNED → EN_ROUTE → (Start OTP) → IN_PROGRESS → (Finish OTP) → AWAITING_PAYMENT → COMPLETED
                               └→ CANCELLED (by user before IN_PROGRESS)
SEARCHING → (no accept after rounds) → FAILED
```

* **Start OTP**: required to move EN_ROUTE → IN_PROGRESS.
* **Finish OTP**: required to move IN_PROGRESS → AWAITING_PAYMENT.
* **Payment**: UPI (QR on partner app, manual confirm in v1) **or** COD → then **COMPLETED**.

### Partner Payment Screen (v1)

* After **Finish OTP** succeeds, navigate to **Payment** screen.
* Show **amount due** and two CTAs:

  * **UPI (recommended)** → render UPI **QR** (built from deeplink `upi://pay?...`) + “Open UPI app”. After user pays, partner taps **Mark Paid** → server `confirmPayment` callable.
  * **Cash (COD)** → partner taps **Cash collected** → server `confirmPayment` callable.
* On success: toast + navigate to **Job Completed**.

### Config (admin-managed)

`/config/payments`

```json
{
  "receiverMode": "MERCHANT",        // or "PARTNER"
  "merchant": { "vpa": "haazir@icici", "name": "Haazir Services" },
  "upiTimeoutSec": 180,
  "allowCOD": true
}
```

### Cloud Functions (additions)

* `completeJob({ jobId, otp })` → verifies Finish OTP → sets `status: "AWAITING_PAYMENT"`, primes `payment` block.
* `confirmPayment({ jobId, method })` → **only when** `status == AWAITING_PAYMENT` and caller is assigned partner:

  * Sets `payment.method = "UPI" | "COD"`, `payment.status = "PAID"`, `payment.collectedAt = now`, and `status = "COMPLETED"`.

### Rules (essentials)

* Partner **cannot** set `payment.status = PAID` via client writes; must use `confirmPayment` callable.
* Lock `amountDue`, `receiverVPA`, OTP hashes from client edits.

### Minimal Client Calls (new)

```ts
// Finish → AWAITING_PAYMENT
await httpsCallable(functions, "completeJob")({ jobId, otp });

// Confirm payment (UPI manual or COD)
await httpsCallable(functions, "confirmPayment")({ jobId, method: "UPI" });
// or
await httpsCallable(functions, "confirmPayment")({ jobId, method: "COD" });
```

### QA Checklist — Added

* Finish OTP moves job to **AWAITING_PAYMENT**.
* **UPI QR** renders; “Mark Paid” completes job with `payment.method = "UPI"`.
* **COD** → “Cash collected” completes job with `payment.method = "COD"`.
* Rules reject payment confirmation if caller isn’t the assigned partner or state ≠ AWAITING_PAYMENT.

---

## 🔄 SYNC UPDATE — Payment & State Flow (Aligned with User App v1)

### Updated Job State Machine

```
SEARCHING → ASSIGNED → EN_ROUTE → (Start OTP) → IN_PROGRESS → (Finish OTP) → AWAITING_PAYMENT → COMPLETED
                               └→ CANCELLED (by user before IN_PROGRESS)
SEARCHING → (no accept after rounds) → FAILED
```

* **Start OTP**: required to move EN_ROUTE → IN_PROGRESS.
* **Finish OTP**: required to move IN_PROGRESS → AWAITING_PAYMENT.
* **Payment**: UPI (QR on partner app, manual confirm in v1) **or** COD → then **COMPLETED**.

### Partner Payment Screen (v1)

* After **Finish OTP** succeeds, navigate to **Payment** screen.
* Show **amount due** and two CTAs:

  * **UPI (recommended)** → render UPI **QR** (built from deeplink `upi://pay?...`) + “Open UPI app”. After user pays, partner taps **Mark Paid** → server `confirmPayment` callable.
  * **Cash (COD)** → partner taps **Cash collected** → server `confirmPayment` callable.
* On success: toast + navigate to **Job Completed**.

### Config (admin-managed)

`/config/payments`

```json
{
  "receiverMode": "MERCHANT",        // or "PARTNER"
  "merchant": { "vpa": "haazir@icici", "name": "Haazir Services" },
  "upiTimeoutSec": 180,
  "allowCOD": true
}
```

### Cloud Functions (additions/confirm)

* `completeJob({ jobId, otp })` → verifies Finish OTP → sets `status: "AWAITING_PAYMENT"`, primes `payment` block.
* `confirmPayment({ jobId, method })` → **only when** `status == AWAITING_PAYMENT` and caller is assigned partner:

  * Sets `payment.method = "UPI" | "COD"`, `payment.status = "PAID"`, `payment.collectedAt = now`, and `status = "COMPLETED"`.

### Rules (essentials)

* Partner **cannot** set `payment.status = PAID` via client writes; must use `confirmPayment` callable.
* Lock `amountDue`, `receiverVPA`, OTP hashes from client edits.

### Minimal Client Calls (new)

```ts
// Finish → AWAITING_PAYMENT
await httpsCallable(functions, "completeJob")({ jobId, otp });

// Confirm payment (UPI manual or COD)
await httpsCallable(functions, "confirmPayment")({ jobId, method: "UPI" });
// or
await httpsCallable(functions, "confirmPayment")({ jobId, method: "COD" });
```

### QA Checklist — Added

* Finish OTP moves job to **AWAITING_PAYMENT**.
* **UPI QR** renders; “Mark Paid” completes job with `payment.method = "UPI"`.
* **COD** → “Cash collected” completes job with `payment.method = "COD"`.
* Rules reject payment confirmation if caller isn’t the assigned partner or state ≠ AWAITING_PAYMENT.
