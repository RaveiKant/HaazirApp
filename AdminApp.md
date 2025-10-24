# Haazir Admin Portal — README (v1 POC)

A build-ready specification for the **Haazir Admin** web app. This README captures the minimum features, data shapes, security, and backend hooks to operate the marketplace.

---

## 0) Scope & Goals

Operate catalogue, onboard/verify partners, tune dispatch & pricing, monitor jobs/reviews/KPIs. Admin-only writes; users/partners read where relevant.

---

## 1) Auth & Access

* **Firebase Auth: Email + Password** (enable email verification; recommend MFA).
* Assign custom claim `role: "admin"` via callable `setRole`.
* Client route guard renders admin UI only when `claims.role === "admin"`.

```ts
// Client guard
onIdTokenChanged(auth, async (user) => {
  if (!user) return redirect("/login");
  const tok = await getIdTokenResult(user, true);
  if (tok.claims.role !== "admin") return redirect("/not-authorized");
});
```

---

## 2) Core Modules (v1)

1. **Dashboard** – KPIs: Jobs Today, Fill Rate, Avg Time-to-Assign, Partners Online, Complaints/Flags. Quick filters: Pending KYC, Stuck Jobs.
2. **Services (Catalogue)** – CRUD: `title`, `category`, `description` (short), `details` { inclusions, exclusions, slaMinutes, basePrice (paise), ctaLabel }, `images[]` [{url, alt, role:"hero"|"gallery"}], `isActive`.
3. **Partners** – Table: Name, Phone, **KYC** status, Online, Services, ★ Rating, Total Orders, Total Earnings. KYC drawer: upload **selfie** & **Aadhaar**, set `kyc.status`, edit **`description`** (admin-only bio shown to users), override services if needed.
4. **Jobs Monitor** – Live list + filters; detail drawer shows user/partner cards, payout, timers, **OTP** status; actions: Unassign stale, Force Fail, Re-broadcast (guarded).
5. **Reviews** – Stream of new reviews; soft-hide via `hidden: true`; mark for follow-up.
6. **Config** – Edit `/config/app`: search radius, broadcast size, accept window, escalation step/rounds, platform fee %, tax %, OTP attempt limit, stale ASSIGNED window.
7. **Roles** – Set/revoke admin/partner roles (`setRole({ uid, role })`).

---

## 3) Firestore & Storage Schema (admin touchpoints)

### Firestore

* `/services/{serviceId}` (admin CRUD; public read)

```json
{
  "title": "Electrician",
  "category": "Home Repair",
  "description": "Fan, light, AC, basic wiring.",
  "details": {
    "inclusions": ["Inspection", "Minor wiring fix"],
    "exclusions": ["Major rewiring"],
    "slaMinutes": 60,
    "basePrice": 29900,
    "ctaLabel": "Book Electrician"
  },
  "images": [
    { "url": "https://.../serviceMedia/electrician/hero.webp", "alt": "Electrician fixing fan", "role": "hero" }
  ],
  "isActive": true,
  "updatedAt": "...",
  "updatedBy": "adminUid"
}
```

* `/partners/{uid}` (admin writes **`kyc.*`**, **`description`**, may override `services`)
* `/partners/{uid}/stats/*` (read only; written by Functions)
* `/partners/{uid}/reviews/*` (read; admin may set `hidden: true`)
* `/jobs/{jobId}` (read; writes via Functions only)
* `/config/app` (admin write JSON config)
* `/adminLogs/{logId}` (optional audit trail)

### Storage

* `/serviceMedia/{serviceId}/*` – public **read**, admin **write** (service images)
* `/kyc/{uid}/*` – admin-only read/write (selfie, Aadhaar)

---

## 4) Security Rules (snapshot)

```rules
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {
    function isAdmin() { return request.auth != null && request.auth.token.role == "admin"; }
    function isPartner(uid) { return request.auth != null && request.auth.token.role == "partner" && request.auth.uid == uid; }

    match /services/{sid} {
      allow read: if true;           // user-visible
      allow write: if isAdmin();     // admin-only
    }

    match /partners/{p} {
      allow read: if isAdmin() || isPartner(p);
      // partner limited self-updates happen elsewhere; admins can write everything (kyc.*, description)
      allow write: if isAdmin();
    }

    match /partners/{p}/stats/{doc=**} { allow read: if isAdmin() || isPartner(p); allow write: if false; }
    match /partners/{p}/reviews/{r}    { allow read: if isAdmin() || isPartner(p); allow write: if false; }

    match /jobs/{j} { allow read: if isAdmin(); allow write: if false; }
    match /config/{doc} { allow read: if true; allow write: if isAdmin(); }
    match /adminLogs/{id} { allow read, write: if isAdmin(); }
  }
}
```

```rules
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /serviceMedia/{serviceId}/{fileName} {
      allow read: if true;  // catalog images public
      allow write: if request.auth != null && request.auth.token.role == 'admin';
    }
    match /kyc/{uid}/{fileName} {
      allow read, write: if request.auth != null && request.auth.token.role == 'admin';
    }
  }
}
```

---

## 5) Cloud Functions (admin-related)

* `setRole` – assign custom claims (admin-only callable).
* `verifyKyc` (or Firestore update) – when `kyc.status: VERIFIED`, flip partner `onboardingStep: "READY"` and notify.
* `updateConfig` – validate and write `/config/app`.
* Maintenance (optional): `rebroadcastJob`, `unassignStaleAssigned` (cron) – admin-guarded.

> System functions already in place: `respondToOffer`, `startJob`, `completeJob`, `onJobRated`, `escalate`, `ensurePartnerProfile`.

---

## 6) Admin Workflows

**KYC**

1. Partners → filter `PENDING`.
2. Upload **Selfie** & **Aadhaar** to `/kyc/{uid}`.
3. Set `kyc.status = "VERIFIED"` (or `REJECTED` + `kyc.note`).
4. Write **`description`** (public bio for users). Save → partner notified.

**Catalogue**

* Create/edit service text & details; upload/reorder images; toggle `isActive`.

**Dispatch tuning**

* Edit `/config/app` fields; next cycles pick new values automatically.

**Reviews moderation**

* Set `hidden: true` for harmful comments (soft-hide on user app).

**Jobs triage**

* Identify **ASSIGNED** older than `ASSIGNED_STALE_MIN` → Unassign/Rebroadcast/Fail with reason (guarded ops).

**Roles**

* Use `setRole({ uid, role })` to grant/revoke admin/partner roles.

---

## 7) KPIs (Dashboard)

* Fill Rate (% assigned within N mins)
* Avg Time-to-Assign (mm:ss)
* Acceptance Rate (by service/area)
* Escalation Cost Δ (₹)
* Partners Online (current)
* Complaints / Hidden reviews (today)

---

## 8) Acceptance Checklist (v1 Admin)

* [ ] Admin login via email/password; route-guarded UI.
* [ ] Services editable (text, details, images); visible to users/partners.
* [ ] Partner KYC: upload docs, set status, **edit description**; partner moves to Ready.
* [ ] Roles page sets admin/partner roles.
* [ ] Config page writes `/config/app`; functions honor values.
* [ ] Jobs monitor lists & filters; guarded actions available.
* [ ] Reviews visible; can soft-hide.
* [ ] Rules enforce admin-only writes for KYC/description/services; KYC Storage is admin-only.

---

## 9) Dev Notes

* Currency in **paise** across services/job payout calculations.
* Use **WebP/AVIF** for service images; provide `alt` text; generate responsive sizes (optional Storage trigger).
* Log admin actions to `/adminLogs/*` (who/what/when) for KYC, roles, config edits.
* Prefer **App Check** on Admin UI; limit callable functions to admins.

---

**Status:** Spec complete. Safe to implement Admin portal when ready.
