# Haazir Partner App — README (v1 POC)

Build-ready specification for the **Haazir Partner** web app (responsive PWA). Aligned with User/Admin apps and Firebase backend.

---

## 0) Scope & Goals
- Onboard partners fast (OTP → Name → Services).
- Admin-driven KYC (selfie + Aadhaar + admin description).
- Toggle Online, receive offers, do Start/Finish via OTP, collect payment (UPI QR or COD), complete jobs.
- Show earnings, totals, and reviews.

---

## 1) Auth & Onboarding
- **Phone OTP only** (Firebase Auth).
- First login seeds `partners/{uid}` with `onboardingStep: "ENTER_NAME"`, `kyc.status: "PENDING"`, `isOnline: false`.
- Flow: **OTP → Name → Select Services → Pending KYC → (Admin verifies) → Ready**.
- Online toggle enabled only when `kyc.status === "VERIFIED"`.

**partners/{uid} (key fields)**
```json
{
  "name": "Ravi",
  "services": ["electrician", "plumber"],
  "isOnline": false,
  "_geofire": { "geohash": "", "lat": 0, "lng": 0 },
  "kyc": { "status": "PENDING" },
  "description": "5 yrs electrician; AC/fan specialist",
  "ratingAvg": 0,
  "ratingCnt": 0
}
```

---

## 2) Core Flows (Partner)
1) **Go Online** → share location → eligible for offers within radius.
2) **Offer** arrives (60s timer). **Accept** or **Decline**; first accept wins.
3) **EN_ROUTE** to customer; on arrival ask for **Start OTP** → moves to **IN_PROGRESS**.
4) Finish work; ask for **Finish OTP** → moves to **AWAITING_PAYMENT**.
5) **Payment screen**: choose **UPI** (show QR + “Open UPI app”, then **Mark Paid**) or **COD** (**Cash collected**). Job becomes **COMPLETED**.
6) View **Earnings per job**, **Total Earnings**, **Total Orders**, **Reviews**.

---

## 3) Updated Job State Machine (v1)
```
SEARCHING → ASSIGNED → EN_ROUTE → (Start OTP) → IN_PROGRESS → (Finish OTP) → AWAITING_PAYMENT → COMPLETED
                               └→ CANCELLED (before IN_PROGRESS)
SEARCHING → (no accept) → FAILED
```

- **Start OTP**: EN_ROUTE → IN_PROGRESS.
- **Finish OTP**: IN_PROGRESS → AWAITING_PAYMENT.
- Payment (UPI/COD) → COMPLETED.

---

## 4) Screens / Routes
```
/auth-otp
/onboarding/name
/onboarding/services
/home                // KYC status, Online toggle
/offers              // incoming offers with timer
/job/:jobId          // details + Start/Finish OTP + Payment
/dashboard           // totals, recent jobs, reviews
/settings            // edit services, logout
```

---

## 5) Data Model (partner-relevant)
**jobs/{jobId} (highlights)**
```json
{
  "partnerId": "uid",
  "serviceId": "electrician",
  "status": "AWAITING_PAYMENT",
  "payout": 29900,
  "startOtpHash": "...",
  "finishOtpHash": "...",
  "pricing": {
    "basePaise": 29900, "feePaise": 1900, "taxPaise": 342, "estimatePaise": 32142
  },
  "payment": {
    "method": "UPI" | "COD" | null,
    "status": "PENDING" | "PAID" | "FAILED" | "TIMEOUT",
    "receiverVPA": "haazir@icici",
    "collectedAt": null
  },
  "partnerEarning": 29900
}
```

**stats & reviews (read-only to partner)**
```
partners/{uid}/stats/summary   // totalOrders, totalEarnings, avgRating, ratingCnt
partners/{uid}/stats/byMonth/{YYYYMM}  // orders, earnings
partners/{uid}/reviews/{jobId} // stars, comment, ratedAt, serviceId
```

---

## 6) Cloud Functions (partner-related)
- `respondToOffer({ jobId, action })` – atomic accept.
- `startJob({ jobId, otp })` – verify **Start OTP** → `IN_PROGRESS`.
- `completeJob({ jobId, otp })` – verify **Finish OTP** → `AWAITING_PAYMENT`.
- `confirmPayment({ jobId, method })` – **UPI** (manual confirm) or **COD** → set `PAID` → `COMPLETED`.
- `onJobRated` – updates `ratingAvg/ratingCnt`, copies review.
- `onKycVerified` – flips `onboardingStep:"READY"` and notifies.
- `escalate` (cron */1 min) – bump payout if no accept (cap rounds).

---

## 7) Payments (v1 simple)
- `/config/payments` (admin-managed):
```json
{
  "receiverMode": "MERCHANT",
  "merchant": { "vpa": "haazir@icici", "name": "Haazir Services" },
  "upiTimeoutSec": 180,
  "allowCOD": true
}
```
- **UPI:** build `upi://pay` deeplink → render QR → “Mark Paid” (manual).  
- **COD:** “Cash collected”.

Helper:
```ts
export function upiDeeplink({ vpa, name, paise, jobId }:{vpa:string;name:string;paise:number;jobId:string}){
  const am = (paise/100).toFixed(2);
  const p = new URLSearchParams({ pa:vpa, pn:name, am:am, cu:"INR", tn:`Haazir Job ${jobId}` });
  return `upi://pay?${p.toString()}`;
}
```

---

## 8) Security Rules (essentials)
- Partner can update only safe fields in own partner doc.
- Jobs: state transitions (ACCEPT/START/COMPLETE/PAY) via **callables**.
- Partner **cannot** write KYC, ratings, OTP hashes, payout/amountDue, or set `payment.status="PAID"` directly.

Rule sketch:
```rules
match /jobs/{j} {
  allow read: if request.auth != null && resource.data.partnerId == request.auth.uid;
  allow write: if false; // use Functions
}
```

---

## 9) QA Checklist
- OTP → Name → Services → Pending KYC → Ready.
- Online toggle blocked until Verified.
- Offer accept assigns within 60s; losers expire.
- Start OTP → IN_PROGRESS; Finish OTP → AWAITING_PAYMENT.
- UPI QR shows; “Mark Paid” OR “Cash collected” → COMPLETED.
- Earnings/Orders/Reviews visible and correct.
- Rules block unsafe writes.
