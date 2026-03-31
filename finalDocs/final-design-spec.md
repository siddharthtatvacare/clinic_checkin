# Flow A — Pre-Booked Patient: Design Spec
> Date: 2026-03-24
> Status: Approved (visual prototype reviewed and signed off)
> Prototype: `finalDocs/all-flows-ipad.html`
> PRD reference: `01-prd-check-in.md` §4.1

---

## What This Is

A high-fidelity React prototype of the **Flow A kiosk check-in** experience for a pre-booked patient at a Visit Health clinic. The prototype runs inside a simulated iPad frame and is intended as a demo artifact for stakeholders and a handoff spec for production developers.

---

## Architecture Separation

The prototype (and eventual production code) has two clean layers:

| Layer | Purpose | In production |
|---|---|---|
| `demo-shell/` | iPad bezel, step pills, info panel | **Delete** |
| `kiosk/` | All screen content, flows, components | **Ship as-is** |

`App.tsx` wraps `<KioskApp>` in `<DemoShell>`. Remove `DemoShell` for production deployment.

---

## Screen Flow

```
S1 Welcome (5-option home screen, showing pre-booked doctor + lab active)
  ├── Tap "Pre-booked doctor appointment" → S2 Identify
  ├── Tap "Pre-booked lab test" → S2 Identify (same identity flow, different appointment type)
  └── Walk-in / Pharmacy → out of scope for Flow A (greyed out in prototype)

S2 Identify
  ├── Mobile tab → Phone entry + keypad
  │     ├── Phone number not found in system → Error: Not Found
  │     └── Phone found → "Enter the OTP sent to your phone" → S2b OTP Verify
  │           ├── OTP correct → S3 Appointment List (or disambiguation if multiple patients)
  │           └── OTP wrong 3x → Error: OTP Blocked
  └── QR tab → QR Scanner viewfinder
        ├── QR valid → skip OTP → S3 Appointment List
        └── QR invalid / wrong patient → stay on scanner with error message

S3 Appointment List
  ├── Multiple patients on one number → disambiguation: "Which appointment?" → S3 filtered
  ├── "Check In Now" → S4 Queue Token (per appointment type: consult or lab)
  └── "← Back" (top-left) → S2 Mobile Number entry (not OTP)

S4 Queue Token → auto-reset to S1 after 30s idle timeout

Error: Not Found
  ├── "Try Again" → S2
  └── "Talk to Receptionist" → (concierge, no further kiosk screens)

Error: OTP Blocked (3 wrong attempts)
  └── "Talk to Receptionist" → (no retry from this screen)
```

---

## Screens

### S1 — Welcome (5-Option Home Screen)

Per PRD §4.0.1, the kiosk home screen has **exactly 5 types of users** and **three options of appointments**:

1. I am the policy holder and have a pre-booked **doctor** appointment or a **lab** appointment or **annual health check** appointment
2. I am a beneficiary of the policy holder and I have a a pre-booked **doctor** appointment or a **lab** appointment or **annual health check** appointment
3. I am a policy holder and I do not have a prebooked appointment and am walking in for a **doctor** appointment or a **lab** appointment or **annual health check** appointment
4. I am a beneficiary of a policy holder and am walking in for a **doctor** appointment or a **lab** appointment or **annual health check** appointment
5. I am neither a policy holder nor a beneficiary of a policy holder and am walking in for a **doctor** appointment or a **lab** appointment

**Prototype simplification (deliberate):** The prototype shows options 1 and 3 as active (pre-booked only). Walk-in options and pharmacy are visible but greyed out — they will be active in Flows B–E. This is a prototype scope decision, not a product decision; in production all 5 are fully functional.

Language toggle (English / Hindi) shown in header — language switching deferred to later phase.

Visit logo top-left (`/public/visit-logo.svg`). No progress bar. No step count label.

### S2 — Identify: Mobile Number

- Segmented tab control: Mobile Number | Scan QR Code
- Large-touch numeric keypad (no system keyboard — kiosk context)
- Phone display field: +91 prefix, blinking cursor
- **"Find my appointment →"** button (not "Send OTP") — this calls the backend to verify a booking exists for this number today
  - If no booking found: error screen ("not found")
  - If booking found: advance to S2b OTP entry

> **OTP mechanism (from PRD §4.1.0):** The OTP is sent automatically **30 minutes before the appointment** by the Visit backend — not on-demand when the patient enters their number at the kiosk. The kiosk asks the patient to enter the OTP they already received on their phone. A "Resend OTP" option is available if they didn't receive it, rate-limited to **1 per 2 minutes**.

### S2 — Identify: QR Scan

- Same tab control as mobile screen, QR tab active
- Dark camera viewfinder containing:
  - Four **purple corner bracket** guides marking the scan target area
  - Animated **scan beam**: purple → coral → purple gradient, 2-second loop, travels top-to-bottom through the frame
  - A faint **QR code pattern** (SVG, 30% opacity) inside the brackets — shows patients where to position their phone
  - Hint text at bottom of viewfinder: "Camera active — align QR code within the frame"
- Instruction card below viewfinder: "Open your Visit app → My Appointments → tap your booking → show the QR code here"
- "Use mobile number instead →" link below the instruction card
- On valid QR: skip OTP, advance directly to S3
- On invalid/unrecognised QR: show inline error "QR not recognised — try again or use mobile number"
- No auto-timeout on the QR screen (patient can take their time)

### S2b — OTP Entry

- "← Back" returns to S2 Mobile
- Heading: **"Enter the OTP sent to your phone"** (not "we're sending you an OTP")
- Sub-label: "We sent a code to +91 XXXXX XXXXX 30 minutes before your appointment"
- Sub-label shows the **beneficiary's number on file** (fetched from the backend at the S2 booking-exists check — PRD §4.1.0), not the number the patient typed. These can differ (e.g., parent books on their own number but the OTP is sent to the child's number). Copy: "We sent a code to +91 XXXXX XXXXX — the number registered for this appointment."
- 6-digit OTP boxes — auto-advance on each digit entry
- Resend option: "Didn't receive it?" → "Resend OTP" — **rate-limited to 1 per 2 minutes**, countdown shown
- 3 consecutive wrong OTPs → OTP Blocked error screen

### S3 — Appointment List / Disambiguation

**If multiple patients on same number** (e.g., parent booked for two children — PRD §4.1.2):
- Show disambiguation screen first: "We found multiple appointments — which one is you?"
- Patient selects their appointment → advance to the appointment list for that patient only

**Appointment list:**
- Heading: "Welcome, {full name}!" (full name from API, per PRD/A_PreBooked.md)
- Sub: "Here are your appointments for today"
- Each appointment card shows:
  - **Consultation:** doctor name, specialty, time (no location shown at this step)
  - **Lab test:** test name(s), time (no location shown at this step; prep reminder if fasting required)
  - Cards colour-coded: purple left-border = consult, coral = lab
- "← Back" button (top-left) → returns to S2 Mobile Number entry (not OTP — patient restarts identification from scratch)
- Single "Check In Now →" CTA — check-in is **global** (all appointments at once)

**Late arrival handling (per PRD §4.1.2):**
- Early (>30 min before slot): allowed, show "You're a bit early — your appointment is at {time}. You're checked in."
- Late within grace period (<15 min past): allowed, show "Your appointment was at {time}. You may experience a wait."
- Late past grace period (>15 min past, slot reassigned): show "Your slot has been reassigned. Would you like to join the walk-in queue?" → redirects to walk-in flow (out of scope for this prototype — show message + "Talk to Receptionist" in v1)

### S4 — Queue Token

**For consultation:**
- Success icon + "You're checked in, {first name}!"
- Token card: large token number (e.g. A-047), doctor name, cabin + floor, estimated wait
- Routing instruction: "Head to the waiting area — your consultation will be in Cabin 3"

**For lab test:**

> **Phlebotomist availability check (PRD §4.1.3, step K4):** Before issuing a lab token, the system checks an admin flag in TP for phlebotomist availability. If unavailable, show an error state: "Lab services are currently unavailable. Please speak to our receptionist." No token is issued. This check happens server-side when the patient taps "Check In Now" on S3.

- Success icon + "You're checked in, {first name}!"
- Token card: large token number (e.g. L-012), test name(s), lab location, estimated wait
- Routing instruction: "Proceed to the Lab Collection Area (Room 3, Ground Floor)"

**For both consultation + lab on same visit:**
- Show two token cards (consultation + lab) stacked
- Each with its own routing instruction

WhatsApp confirmation nudge: "A confirmation has been sent to +91 XXXXX XXXXX" — display only, not a confirmed delivery state.

"Done — Return to Start" resets to S1. Auto-reset after **30 seconds** of inactivity.

### Error — Patient / Appointment Not Found
- Shown when: phone number entered but no appointment found for today
- Heading: "We couldn't find your appointment"
- Body: "No appointment was found for this number today. Please check your number or speak to our receptionist."
- CTAs: "Try Again" (→ S2) | "Talk to Receptionist"

### Error — OTP Blocked
- Shown when: 3 consecutive wrong OTPs entered
- Heading: "Too many incorrect attempts"
- Body: "For your security, we've blocked further attempts. Please speak to our receptionist."
- CTA: "Talk to Receptionist" only — no self-serve retry from this screen

---

## UI Patterns

| Element | Spec |
|---|---|
| Tap target min | 60px height on all interactive elements |
| Progress indicator | **Removed** — no progress bar in header |
| Font — headings | Bricolage Grotesque 700 |
| Font — card titles | Montserrat 600 |
| Font — body/labels | Inter 400/500 |
| Primary colour | `#714FFF` |
| Accent colour | `#FF9273` |
| Background | `#F5F3FF` |
| Button radius | 12px |
| Card radius | 12–16px |
| Screen transitions | Framer Motion slide (left→right advance, right→left back) |
| Idle reset timeout | 30 seconds on S4, then return to S1 |
| OTP resend rate limit | 1 per 2 minutes (PRD §4.1.2) |
| PWA viewport | Full-screen adaptive — use `100dvh`/`100vw` not fixed px heights. The kiosk PWA must fill the device screen regardless of iPad model. No fixed pixel heights in production components. |

---

## Open Decisions (surface at demo)

1. **Grace period duration (D-CI-1):** 10 min vs 15 min vs 20 min after scheduled slot before it's reassigned?
2. **Late arrival conversion (D-CI-2):** Should late arrivals be auto-converted to walk-in queue, or given the option to choose?
3. **OTP failure policy:** After 3 wrong attempts: permanently block + require concierge (current design), or allow retry after a cooldown?
4. **QR path assumption:** QR lives inside the Visit app — patient must have the app installed. Is this safe for day-1?
5. **Multiple appointments — separate tokens?** If a patient has both a consultation and a lab test, they get two tokens. Should the kiosk show both at once or handle them sequentially?
6. **"Talk to Receptionist" persistence:** Should a small "Need help?" link appear on every screen, or only on error screens?
7. **Vitals prompt:** PRD flow K8 mentions routing to vitals station. Should the kiosk show a "Proceed to Vitals Station" instruction before the queue token, or handle this only in the Visit app?

---

## Out of Scope for Flow A (This Prototype)

- Walk-in patients — Flows B, C (doctor); Flows D, E (lab)
- Pharmacy pickup — shows "Proceed to Pharmacy Counter" message only
- Beneficiary check-in under a policy holder's booking
- Corporate policy / wallet display
- New patient registration / UHID creation
- Language selector (Hindi) — deferred
- Token printing — deferred pending hardware confirmation
- Vitals prompt on kiosk — deferred (shown in Visit app; pending decision on kiosk trigger)
- Late arrival → walk-in conversion (v1: show message + "Talk to Receptionist"; full flow in later phase)
