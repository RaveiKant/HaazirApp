# Haazir User App — README (v1 POC)

Build-ready specification for the **Haazir User** web app (responsive PWA). Aligned with Partner/Admin apps and Firebase backend.

---

## 0) Scope & Goals
- Discover services, request help **now**, confirm **Start/Finish via OTP**, **pay after finish** (UPI QR or COD), and **rate** the partner.
- Minimal friction; PWA-first; later reusable for mobile.

---

## 1) Auth & Progressive Profiling
- **Phone OTP** signup + **first name** only.
- At first booking: collect **address** (GPS prefill; landmark/flat no.).
- After first completed job: optional **alt phone** + **email** (skippable).

**users/{uid} (minimal)**
```json
{
  "name": "Asha",
  "phone": "+91XXXXXXXXXX",
  "email": null,
  "altPhone": null,
  "addresses": [
    { "label": "Home", "line1": "", "pincode": "", "lat": 0, "lng": 0, "landmark": "", "isDefault": true }
  ]
}
```

---

## 2) Core Flows (User)
1) Browse **Catalogue** or **Search**; open **Service Detail** (images, inclusions/exclusions, SLA, **Starting at ₹X**).
2) Confirm **location** and (optional) **price ceiling**; tap **Find Now**.
3) App **searches nearby partners**; when assigned, show **partner card** (name, ★ rating, bio, ETA).
4) **Start OTP** at arrival → partner enters it to **start**.
5) **Finish OTP** when done → partner enters it; job moves to **AWAITING_PAYMENT**.
6) **Pay after finish**:
   - **UPI**: scan QR shown on partner’s app (or tap UPI deeplink).
   - **COD**: hand over cash.
7) Job becomes **COMPLETED**; prompt to **rate** (stars + comment).
8) **Orders** page lists history and ratings.

---

## 3) Routes
```
/                 // Landing + location
/services         // Catalogue grid
/service/:id      // Details
/checkout         // Address + price ceiling + Find Now
/track/:jobId     // Live status + OTP cards
/orders           // History + rating
/login            // OTP
```

---

## 4) Pricing Display
- Catalogue/Detail: **“Starting at ₹X”**; note: “Includes up to Y minutes. Parts extra.”
- Checkout **estimate** = Visit fee (base) + Service fee (platform) + Tax(on fee).
- If payout escalates during matching, update estimate; confirm if change > threshold.

**jobs/{jobId}.pricing**
```json
{ "basePaise": 29900, "feePaise": 1900, "taxPaise": 342, "estimatePaise": 32142 }
```

---

## 5) Payment (v1 simple)
- Happens **after Finish OTP**.
- Partner selects **UPI** (QR on partner app) or **COD**.
- Backend records `payment.method`, `payment.status`, `collectedAt` (manual confirm in v1).

**jobs/{jobId}.payment**
```json
{ "method": "UPI" | "COD" | null, "status": "PENDING" | "PAID", "receiverVPA": "haazir@icici", "collectedAt": "..." }
```

---

## 6) Security (user-facing)
- Users can read/write **only their own jobs**.
- Users can set `userRating` once and only when `status == COMPLETED`.
- Users cannot change OTPs, partner assignment, payouts, or payment fields.

---

## 7) Realtime & Notifications
- Live `onSnapshot` on `jobs/{jobId}` for state updates.
- Optional web push for “Assigned”/“Completed”.

---

## 8) Acceptance Checklist
- OTP + first name signup.
- Catalogue & details show “Starting at ₹X” and images.
- Checkout creates job; tracking works.
- Start OTP → IN_PROGRESS; Finish OTP → AWAITING_PAYMENT.
- Payment via UPI QR or COD → COMPLETED.
- Rating once; partner rating updates; review visible to partner.
- Orders list shows correct status; security rules hold.
