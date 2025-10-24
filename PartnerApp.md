# Haazir Partner App — README (v1 POC)

A build-ready specification for the **Haazir Partner** application. This README is designed for future implementation; it captures flows, data, functions, and guardrails for a fast Firebase POC that can scale.

---

## 0) Summary

* **Goal:** Connect partners (electrician, plumber, maid, cook, etc.) to users with a simple, reliable workflow.
* **Stack (POC):** Firebase Auth (Phone/OTP), Firestore, Cloud Functions (TS), Cloud Messaging (FCM), Cloud Scheduler, Firebase Hosting. Geo queries via **geofire-common**.
* **Core v1:** OTP-only auth → Name → Services → Pending KYC → Verified → Online. Offer accept/decline with 60s window. **Start OTP** at arrival; **Finish OTP** at completion. Overall ratings. Earnings & Dashboard. Reviews (read-only).

---

## 1) Key Concepts

* **Partner:** Service provider who registers, passes admin KYC, toggles online to receive jobs.
* **Job:** A service request from a user; moves through states from SEARCHING to COMPLETED.
* **Offer:** A round-based broadcast to eligible nearby partners with a payout. First accept wins.

---

## 2) Partner App Flows

### A) New Partner (Registration → Ready)

1. **OTP Sign-in** (Phone only). Function seeds `partners/{uid}` with `onboardingStep: "ENTER_NAME"`, `kyc.status: "PENDING"`, `isOnline: false`, empty `services`.
2. **Enter Name** → save → `onboardingStep: "SELECT_SERVICES"`.
3. **Select Services** from admin catalogue (+ optional personal prices for admin view) → save → `onboardingStep: "PENDING_KYC"`.
4. **Admin KYC**: admin uploads selfie & Aadhaar (Storage, admin-only) → sets `kyc.status: "VERIFIED"`.
5. **Ready**: trigger flips `onboardingStep: "READY"`. Online toggle becomes enabled.

### B) KYC-Approved Partner (Job Flow)

1. **Go Online**: grant location → write `_geofire` + `isOnline: true` → start receiving offers.
2. **Offer**: FCM + `jobs/{jobId}/offers/{partnerId}` with 60s expiry, shows payout. Partner **Accept/Decline**.
3. **Accept**: callable function atomically assigns job; others’ offers expire. Status → **ASSIGNED**.
4. **EN_ROUTE**: partner begins travel.
5. **Start OTP** at arrival: user shows **Start OTP**; partner enters it → **IN_PROGRESS** (server verifies).
6. **Finish OTP** at completion: user shows **Finish OTP**; partner submits → **COMPLETED** (server verifies; earnings computed).
7. **Post-job**: user rates once; partner’s overall rating updates. Partner can read reviews.

---

## 3) Job State Machine (with OTP gates)

```
SEARCHING → ASSIGNED → EN_ROUTE → (Start OTP) → IN_PROGRESS → (Finish OTP) → COMPLETED
                               └→ CANCELLED (before IN_PROGRESS)
SEARCHING → (no accept after rounds) → FAILED
```

* **OTP requirements:**

  * Start OTP: required to move EN_ROUTE → IN_PROGRESS.
  * Finish OTP: required to move IN_PROGRESS → COMPLETED.

---

## 4) UI / Routes (Partner)

```
/auth-otp              // OTP login
/onboarding/name       // enter name
/onboarding/services   // select services + optional prices
/home                  // KYC status, Online toggle, basic cards
/offers                // incoming offers (countdown, accept/decline)
/job/:jobId            // job details + Start/Finish OTP flows & state buttons
/settings              // edit services/prices, profile, logout
/dashboard             // totals, period stats, history, reviews
```

**Dashboard**

* Summary cards: **Total Orders**, **Total Earnings (₹)**, **This Week**, **This Month**, **Avg Rating**.
* History: last N completed jobs with date, service, **earning** (₹), rating if any.
* Reviews: stars, comment, date; newest first.

---

## 5) Data Model (Firestore)

**partners/{partnerId}**

```
uid, name, phone, email, photoUrl,
onboardingStep: "ENTER_NAME" | "SELECT_SERVICES" | "PENDING_KYC" | "READY",
kyc: { status: "PENDING" | "VERIFIED" | "REJECTED", verifiedAt?, verifiedBy?, aadhaarLast4?, docRefId?, selfieUrl? },
services: [serviceId,...],
partnerPrices: { [serviceId]: number },  // admin reference only
isOnline: boolean,
_geofire: { geohash, lat, lng },
ratingAvg: number, ratingCnt: number,
createdAt, updatedAt
```

**services/{serviceId}**

```
name, category, basePrice, estMinutes, isActive
```

**jobs/{jobId}**

```
userId, partnerId?, serviceId, lat, lng,
status: REQUESTED|SEARCHING|ASSIGNED|EN_ROUTE|IN_PROGRESS|COMPLETED|FAILED|CANCELLED,
payout, roundNo, createdAt, startedAt?, completedAt?,
// OTPs (server-managed hashes only)
startOtpHash, startOtpSalt, otpAttemptsStart,
finishOtpHash, finishOtpSalt, otpAttemptsFinish,
// Earnings
platformFeePct, platformFeeAmt, taxPct?, taxAmt?, partnerEarning,
// Reviews
userRating?: { stars: 1..5, comment, ratedAt }
```

**offers (subcollection)**

```
/jobs/{jobId}/offers/{partnerId} → { roundNo, payout, status: PENDING|ACCEPTED|DECLINED|EXPIRED, expiresAt }
```

**Stats & Reviews (read-only to partner)**

```
partners/{id}/stats/summary → { totalOrders, totalEarnings, avgRating, ratingCnt, lastUpdated }
partners/{id}/stats/byMonth/{YYYYMM} → { orders, earnings, from, to }
partners/{id}/reviews/{jobId} → { stars, comment, ratedAt, serviceId }
```

---

## 6) Security Rules (Essentials)

* Partner may update **only**: `name`, `services`, `partnerPrices`, `isOnline`, `_geofire`, `onboardingStep`, `photoUrl`, `updatedAt`.
* Partner **cannot** write: `kyc.*`, `ratingAvg`, `ratingCnt`, earnings or OTP fields.
* Jobs: partner reads only jobs where `partnerId == uid`. State transitions (ACCEPT/START/COMPLETE) go via **callable Functions**.
* Storage `/kyc/{uid}/...` → admins only. `/avatars/{uid}/...` → partner write, public read (optional).

> Implement using custom claims `role: "admin" | "partner"`.

---

## 7) Cloud Functions (Must-Have)

* **ensurePartnerProfile** (Auth onCreate) → seed `partners/{uid}` at `ENTER_NAME`.
* **respondToOffer** (callable) → first-accept wins; set `ASSIGNED`, freeze payout.
* **startJob** (callable) → verify Start OTP; set `IN_PROGRESS`.
* **completeJob** (callable) → verify Finish OTP; compute fees/taxes; set `COMPLETED`; write `partnerEarning`; update stats.
* **onKycVerified** (partners onUpdate) → when `kyc.status` becomes `VERIFIED`, set `onboardingStep: "READY"`; notify partner.
* **onJobRated** (jobs onUpdate) → when `userRating` appears on a `COMPLETED` job: update `ratingAvg`, `ratingCnt`; copy review under `partners/{id}/reviews/{jobId}`.
* **escalate** (HTTP + Cloud Scheduler */1 min) → bump payout if no accepts; re-broadcast; stop after max rounds.

---

## 8) Matching & Scoring (Dispatch Overview)

* Eligible partner: `isOnline === true` AND service skill match AND within geo radius.
* Score suggestion: `score = 1.2 * ratingAvg + 0.8 * (1 / distanceKm)` (fallback `ratingAvg = 4.5` if no ratings yet).
* Broadcast to top N; offer expires in **60s**.

---

## 9) OTP Logic (Server-First)

* Generate **Start OTP** on ASSIGNED; **Finish OTP** on IN_PROGRESS.
* Store **salted hashes** only; never raw OTP.
* Attempt limits (e.g., 5) and cool-downs (2–5 min) per OTP type.
* All verification happens inside **callables**; UI only posts `otp`.

---

## 10) Earnings & Currency

* Store amounts in **paise** (integers). Display as ₹ on UI.
* On **COMPLETED**: compute `platformFeeAmt`, `taxAmt`, and **`partnerEarning`**. Update `stats/summary` and `stats/byMonth/{YYYYMM}`.

---

## 11) Config (Remote Config or `/config` doc)

```json
{
  "SEARCH_RADIUS_KM": 8,
  "TOP_N_BROADCAST": 20,
  "ACCEPT_WINDOW_SEC": 60,
  "ESCALATION_STEP_PAISE": 2000,   // ₹20
  "ESCALATION_STEP_PERCENT": 5,
  "ESCALATION_MAX_ROUNDS": 3,
  "PLATFORM_FEE_PCT": 10,
  "TAX_PCT": 0,
  "OTP_ATTEMPT_LIMIT": 5,
  "ASSIGNED_STALE_MIN": 20
}
```

---

## 12) Minimal Client → Function Contracts

```ts
respondToOffer({ jobId, action: "ACCEPT" | "DECLINE" })
startJob({ jobId, otp })       // state must be EN_ROUTE
completeJob({ jobId, otp })    // state must be IN_PROGRESS
```

---

## 13) Emulator & Dev Notes

* Use Firebase Emulators (Auth, Firestore, Functions, Hosting) for local flows.
* Seed data: services catalogue, 3–5 fake partners, a test job.
* Verify rules with the Security Rules unit test framework.

---

## 14) QA / Acceptance Checklist

* [ ] OTP → Name → Services → Pending KYC → **Ready**.
* [ ] Online disabled until **KYC Verified**.
* [ ] Offer arrives, 60s countdown, **Accept** assigns; others expire.
* [ ] **Start OTP** moves job to **IN_PROGRESS**.
* [ ] **Finish OTP** completes job and computes `partnerEarning`.
* [ ] Dashboard shows **Total Orders**, **Total Earnings**, **This Week/Month**, **Avg Rating**.
* [ ] Reviews visible to partner after user rating.
* [ ] Rules block partner from editing KYC, ratings, earnings, OTP fields.

---

## 15) Future Extensions (Non-Blocking)

* PAN/Aadhaar OCR, bank account/UPI onboarding.
* In-app navigation + distance/ETA via Maps API.
* Dispute center; review moderation; tipping.
* Surge pricing; partner quality tiers; incentives.
* Multi-city ops; service-area polygons.

---

**Status:** Spec complete. Safe to start implementation when ready.
