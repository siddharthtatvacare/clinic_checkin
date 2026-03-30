# Flow A — Pre-Booked Patient: Production-Grade Kiosk Implementation Plan

> **V2 — 2026-03-30:** Migrated from Vite to Create React App (CRA). Tailwind v4 @theme → Tailwind v3 config. Vitest → Jest. All component and hook code unchanged.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a production-grade React kiosk check-in for Flow A (pre-booked patient), with session management, error/offline handling, telemetry, and shared patterns — so Flows B–E can be built on top with zero infrastructure work.

**Architecture:** Four layers: (1) `demo-shell/` wraps the kiosk for demo and is deleted in production; (2) `kiosk/` is the real product; (3) `kiosk/services/` is the API + telemetry layer — mock in demo, real in production, swapped via `REACT_APP_USE_REAL_API`; (4) `kiosk/session/`, `kiosk/errors/`, `kiosk/telemetry/`, `kiosk/layout/` provide production infrastructure (idle management, error boundaries, observability, shared patterns). S1 Welcome lives at the `KioskApp` level (shared entry point for all 5 flows).

**Tech Stack:** Create React App + React 18 + TypeScript, Tailwind CSS v3 (PostCSS), Framer Motion, Lucide React, `@zxing/library` (QR camera reading), Jest + React Testing Library

---

## Key Architectural Decisions

| Decision | Choice | Reason |
|---|---|---|
| Build order | Infrastructure first, then UI | Every screen is production-grade from the moment it renders |
| Provider architecture | Separate providers (Session, Error, Telemetry) | Cleaner separation, easier to test independently |
| State management | `useReducer` per flow + Context for cross-cutting | Maps to XState later; no new deps |
| API calls | `useApiCall` hook per call site | Each screen manages its own loading/error/retry independently |
| Theme enforcement | Tailwind v3 `tailwind.config.js` → semantic classes | `bg-brand` not `bg-[#714FFF]` — single source of truth |
| Idle timeout | Per-screen configurable via `idleConfig.ts` | S1=none, S2/S3=120s, S4=30s, errors=60s |
| Error recovery | ErrorBoundary → "Return to Start" | Safest kiosk recovery — reset to clean state |
| Device auth | Pre-provisioned env vars | No registration ceremony for MVP; upgrade path exists |
| Telemetry | Day 1 — mock=console.log | Shapes the event model early; trivial swap to real analytics |
| Flow loading | Direct imports (not lazy/registry) | Building iteratively; registry adds complexity without value now |

---

## File Map

```
checkin_proto/
├── public/
│   ├── index.html                     ← Google Fonts embed + PWA manifest link
│   └── visit-logo.svg
├── package.json
├── tailwind.config.js                 ← Tailwind v3 theme config (colors, fonts)
├── postcss.config.js                  ← PostCSS with tailwindcss + autoprefixer
├── tsconfig.json
├── .env.development                   ← REACT_APP_USE_REAL_API=false, device vars
├── .env.production                    ← REACT_APP_USE_REAL_API=true
│
├── tests/
│   ├── setup.ts
│   ├── components/
│   │   ├── Button.test.tsx
│   │   ├── NumericKeypad.test.tsx
│   │   ├── OTPInput.test.tsx
│   │   └── QueueTokenCard.test.tsx
│   ├── hooks/
│   │   ├── useApiCall.test.tsx
│   │   └── useIdleTimer.test.tsx
│   └── flows/
│       └── A_PreBooked/
│           └── flow.test.tsx
│
└── src/
    ├── main.tsx
    ├── index.css                       ← @tailwind base/components/utilities
    ├── App.tsx                         ← <DemoShell> wraps <KioskApp>
    │
    ├── tokens/
    │   └── theme.ts                    ← JS-side brand constants
    │
    ├── types/
    │   └── index.ts                    ← Domain + API + Session + Telemetry types
    │
    ├── demo-shell/                     ← DELETE IN PRODUCTION
    │   ├── DemoShell.tsx
    │   ├── IPadBezel.tsx
    │   ├── ArchetypeTabs.tsx
    │   └── InfoPanel.tsx
    │
    └── kiosk/                          ← THE REAL PRODUCT
        ├── KioskApp.tsx                ← S1 lives here; routes to flows
        ├── KioskShell.tsx              ← Logo + progress bar wrapper
        ├── S1_Welcome.tsx              ← Shared 5-option home screen
        │
        ├── session/                    ← PILLAR 1: Session & Idle
        │   ├── SessionProvider.tsx
        │   ├── useIdleTimer.ts
        │   ├── StillThereOverlay.tsx
        │   └── idleConfig.ts
        │
        ├── errors/                     ← PILLAR 2: Error & Offline
        │   ├── KioskErrorBoundary.tsx
        │   ├── ErrorRecoveryScreen.tsx
        │   ├── OfflineOverlay.tsx
        │   ├── useApiCall.ts
        │   └── useNetworkStatus.ts
        │
        ├── telemetry/                  ← PILLAR 3: Telemetry + Device
        │   ├── telemetry.ts            ← Interface
        │   ├── telemetry.mock.ts
        │   ├── telemetry.real.ts
        │   ├── index.ts                ← Env switcher
        │   ├── useTelemetry.ts
        │   └── deviceConfig.ts
        │
        ├── layout/                     ← PILLAR 4: Shared Patterns
        │   ├── PageLayout.tsx
        │   └── FlowShell.tsx
        │
        ├── services/                   ← API Layer
        │   ├── api.ts                  ← Interface (KioskAPI contract)
        │   ├── api-raw.ts              ← Raw API response shapes + transformer functions (one per team)
        │   ├── api.mock.ts
        │   ├── api.real.ts
        │   ├── apiHeaders.ts           ← createApiHeaders()
        │   └── index.ts                ← Env switcher
        │
        ├── components/
        │   ├── Button.tsx
        │   ├── Card.tsx
        │   ├── Badge.tsx
        │   ├── ProgressBar.tsx
        │   ├── StepHeader.tsx
        │   ├── NumericKeypad.tsx
        │   ├── OTPInput.tsx
        │   └── QueueTokenCard.tsx
        │
        ├── flows/
        │   ├── A_PreBooked/
        │   │   ├── index.tsx           ← useReducer state machine
        │   │   ├── flowMeta.ts
        │   │   └── steps/
        │   │       ├── S2_Identify.tsx
        │   │       ├── S2b_OTPVerify.tsx
        │   │       ├── S3_AppointmentList.tsx
        │   │       ├── S4_QueueToken.tsx
        │   │       ├── ErrorNotFound.tsx
        │   │       └── ErrorOTPBlocked.tsx
        │   ├── B_Beneficiary/index.tsx         ← Stub
        │   ├── C_PolicyHolder/index.tsx        ← Stub
        │   ├── D_BeneficiaryWalkIn/index.tsx   ← Stub
        │   └── E_NonPolicy/index.tsx           ← Stub
        │
        └── data/
            ├── mockAppointments.ts
            └── mockPatients.ts
```

---

## Dependency Graph

```
T1 → T2 → T3, T4, T5 (parallel) → T6 → T7, T8 (parallel) → T9, T10, T11 (parallel) → T12 → T13 → T14 → T15 → T16 → T17 → T18
```

## Review Checkpoints

1. **After T5** — All infrastructure built. No visible UI. Verify: `tsc --noEmit`, unit tests for session/error/telemetry hooks.
2. **After T12** — iPad frame renders with providers wired. Verify: dev server, error boundary catches, idle prompt fires.
3. **After T16** — Flow A complete and tested. Verify: full happy path + error paths + offline + idle.
4. **After T18** — Ship-ready. Verify: production build, iPad Safari PWA, no inline hex, all tests green.

---

## Task 1: Project Scaffolding

**Files:**
- Create: `package.json`, `tailwind.config.js`, `postcss.config.js`, `tsconfig.json`, `public/index.html`, `.env.development`, `.env.production`, `tests/setup.ts`

- [ ] **Step 1: Scaffold the CRA + React + TS project**

Run from `check-in/` (parent directory):
```bash
npx create-react-app checkin_proto --template typescript
cd checkin_proto
```

- [ ] **Step 2: Install all dependencies**

```bash
npm install framer-motion lucide-react @zxing/library
npm install -D tailwindcss@3 postcss autoprefixer \
  @testing-library/react @testing-library/jest-dom \
  @testing-library/user-event
```

- [ ] **Step 3: Configure tailwind.config.js and postcss.config.js**

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ['./src/**/*.{js,jsx,ts,tsx}'],
  theme: {
    extend: {
      colors: {
        brand: '#714FFF',
        'brand-dark': '#593FC9',
        accent: '#FF9273',
        'text-primary': '#1F1E1E',
        'text-secondary': '#686670',
        'bg-surface': '#F5F3FF',
        'bg-card': '#FFFFFF',
        'success-bg': '#E6F9F0',
        'success-text': '#2d7a50',
        'error-bg': '#FFF0EB',
        'border-light': '#ede9ff',
      },
      fontFamily: {
        heading: ["'Bricolage Grotesque'", 'sans-serif'],
        subheading: ["'Montserrat'", 'sans-serif'],
        body: ["'Inter'", 'sans-serif'],
      },
    },
  },
  plugins: [],
}
```

```javascript
// postcss.config.js
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
}
```

- [ ] **Step 4: Create tests/setup.ts**

```typescript
// tests/setup.ts
import '@testing-library/jest-dom'
```

- [ ] **Step 5: Add Google Fonts to public/index.html**

```html
<!-- public/index.html <head> -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Bricolage+Grotesque:opsz,wght@12..96,600;12..96,700&family=Inter:wght@400;500;600&family=Montserrat:wght@400;500;600&display=swap" rel="stylesheet">
```

- [ ] **Step 6: Create env files**

```bash
# .env.development
REACT_APP_USE_REAL_API=false
REACT_APP_DEVICE_ID=dev-kiosk-01
REACT_APP_CLINIC_ID=dev-clinic-001
REACT_APP_CLINIC_NAME=Visit Health - Dev

# .env.production
REACT_APP_USE_REAL_API=true
REACT_APP_API_BASE_URL=https://api.visithealth.in
REACT_APP_DEVICE_ID=                   # Set during kiosk provisioning
REACT_APP_CLINIC_ID=                   # Set during kiosk provisioning
REACT_APP_CLINIC_NAME=                 # Set during kiosk provisioning
```

- [ ] **Step 7: Verify dev server starts**

```bash
npm start
```
Expected: browser opens at `http://localhost:3000`.

- [ ] **Step 8: Commit**

```bash
git init && git add . && git commit -m "feat: scaffold CRA+React+TS, Tailwind v3, Jest, env config"
```

---

## Task 2: Theme Tokens + Types

**Files:**
- Create: `src/tokens/theme.ts`
- Create: `src/index.css`
- Create: `src/types/index.ts`

- [ ] **Step 1: Create src/index.css with Tailwind v3 directives**

```css
/* src/index.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

body {
  font-family: 'Inter', sans-serif;
  margin: 0;
}
```

Colors are defined in `tailwind.config.js` instead of CSS `@theme`. This generates Tailwind utilities: `bg-brand`, `text-brand-dark`, `bg-accent`, `text-text-primary`, `text-text-secondary`, `bg-bg-surface`, `bg-bg-card`, `bg-success-bg`, `text-success-text`, `bg-error-bg`, `border-border-light`.

- [ ] **Step 2: Create theme.ts (JS-side constants for programmatic use)**

```typescript
// src/tokens/theme.ts
// Use these only when a JS value is needed (e.g. style prop, Card accentColor).
// For className, use Tailwind utilities: bg-brand, text-accent, etc.

export const colors = {
  brand:         '#714FFF',
  brandDark:     '#593FC9',
  accent:        '#FF9273',
  textPrimary:   '#1F1E1E',
  textSecondary: '#686670',
  bgSurface:     '#F5F3FF',
  bgCard:        '#FFFFFF',
  successBg:     '#E6F9F0',
  successText:   '#2d7a50',
  errorBg:       '#FFF0EB',
  borderLight:   '#ede9ff',
}

export const fonts = {
  heading:    "'Bricolage Grotesque', sans-serif",
  subheading: "'Montserrat', sans-serif",
  body:       "'Inter', sans-serif",
}
```

- [ ] **Step 3: Create types/index.ts — all types for the project**

```typescript
// src/types/index.ts

// ── Domain types ──────────────────────────────────────────────
export type AppointmentType = 'consultation' | 'lab' | 'pharmacy'

export interface Appointment {
  id: string
  type: AppointmentType
  patientName: string
  patientId: string
  doctorName?: string
  specialty?: string
  testNames?: string[]
  time: string
  location: string
  prepNote?: string
}

export interface Patient {
  id: string
  fullName: string
  phone: string
  appointments: Appointment[]
}

export interface QueueToken {
  tokenNumber: string
  appointmentType: AppointmentType
  doctorName?: string
  location: string
  estimatedWaitMinutes: number
  aheadCount: number
}

export interface Doctor {
  id: string
  name: string
  specialty: string
  cabin: string
}

// AppointmentWithDetails = Appointment + cabin enrichment from TatvaPractice.
// The split exists because cabin data comes from a different API (TP-owned),
// fetched after Visit's GetPatientOrders resolves. The UI renders the appointment
// card immediately from base data; cabin fields enrich once the second call lands.
export type CabinDataStatus = 'loading' | 'loaded' | 'unavailable'

export interface AppointmentWithDetails extends Appointment {
  // From TP's AppointmentCabinDetails API — all optional.
  // The UI renders and check-in proceeds without these; they are wayfinding aids only.
  cabin?: string
  floor?: string
  waitingAreaInstructions?: string
  // Tracks the TP call lifecycle so the card renders skeleton or fallback text
  cabinDataStatus: CabinDataStatus
}

// Returned by api.getAppointmentCabinDetails() — key = appointmentId
export type CabinDetailMap = Record<string, {
  cabin?: string
  floor?: string
  waitingAreaInstructions?: string
}>

// ── API result types ──────────────────────────────────────────
export type VerifyPhoneResult =
  | { found: true;  patient: Patient; beneficiaryPhone: string }
  | { found: false }

export type OTPResult =
  | { valid: true }
  | { valid: false }

export type QRScanResult =
  | { found: true;  patient: Patient }
  | { found: false; reason: 'invalid_qr' | 'patient_not_found' }

export type CheckInResult =
  | { success: true;  tokens: QueueToken[] }
  | { success: false; reason: 'lab_unavailable' | 'slot_expired' | 'error' }

// ── Flow state types ──────────────────────────────────────────
export type FlowStep =
  | 'identify' | 'otp' | 'appointments' | 'token'
  | 'error-not-found' | 'error-otp-blocked'

export type PreBookedType = 'doctor' | 'lab'

export interface FlowAState {
  step: FlowStep
  preBookedType: PreBookedType
  phone: string
  beneficiaryPhone: string
  patient: Patient | null
  otpAttempts: number
  tokens: QueueToken[]
}

export type FlowAAction =
  | { type: 'PHONE_SUBMITTED';   phone: string }
  | { type: 'PATIENT_FOUND';     patient: Patient; beneficiaryPhone: string }
  | { type: 'PATIENT_NOT_FOUND' }
  | { type: 'QR_SCANNED';        patient: Patient }
  | { type: 'OTP_CORRECT' }
  | { type: 'OTP_WRONG' }
  | { type: 'CHECKIN_CONFIRMED'; tokens: QueueToken[] }
  | { type: 'LAB_UNAVAILABLE' }
  | { type: 'BACK' }
  | { type: 'RESET' }

// ── Session types ─────────────────────────────────────────────
export interface KioskSession {
  sessionId: string
  startedAt: number
  currentScreen: string
}

export interface IdleConfig {
  timeoutMs: number
  warningMs: number
}

export type ScreenIdleMap = Record<string, IdleConfig | null>

// ── Telemetry types ───────────────────────────────────────────
export type TelemetryEventName =
  | 'session_started'
  | 'session_ended'
  | 'session_timeout'
  | 'screen_view'
  | 'check_in_started'
  | 'check_in_completed'
  | 'check_in_failed'
  | 'error_occurred'
  | 'otp_attempt'
  | 'qr_scan_attempt'
  | 'network_offline'
  | 'network_restored'

export interface TelemetryEvent {
  event: TelemetryEventName
  timestamp: number
  sessionId: string
  deviceId: string
  clinicId: string
  screenName?: string
  durationMs?: number
  metadata?: Record<string, string | number | boolean>
}

export interface TelemetryService {
  track(event: TelemetryEvent): void
  flush(): Promise<void>
}

// ── Device types ──────────────────────────────────────────────
export interface DeviceConfig {
  deviceId: string
  clinicId: string
  clinicName: string
}

// ── API error types ───────────────────────────────────────────
export type ApiErrorKind = 'network' | 'timeout' | 'server' | 'client' | 'unknown'

export interface ApiError {
  kind: ApiErrorKind
  status?: number
  message: string
  retryable: boolean
}

export interface ApiCallState<T> {
  data: T | null
  error: ApiError | null
  loading: boolean
}

// ── Demo shell types ──────────────────────────────────────────
export interface StepMeta {
  name: string
  description: string
  openDecisions: string[]
}
export type FlowMeta = Record<string, StepMeta>
```

- [ ] **Step 4: Commit**

```bash
git add src/tokens src/types src/index.css
git commit -m "feat: Tailwind v3 config tokens, CSS directives, all domain + infra types"
```

---

## Task 3: Session Management

**Files:**
- Create: `src/kiosk/session/idleConfig.ts`
- Create: `src/kiosk/session/useIdleTimer.ts`
- Create: `src/kiosk/session/StillThereOverlay.tsx`
- Create: `src/kiosk/session/SessionProvider.tsx`
- Create: `tests/hooks/useIdleTimer.test.tsx`

- [ ] **Step 1: Create idleConfig.ts**

```typescript
// src/kiosk/session/idleConfig.ts
import { ScreenIdleMap } from '../../types'

// null = no timeout (S1 Welcome: kiosk waits indefinitely for a patient)
export const defaultIdleConfig: ScreenIdleMap = {
  welcome:             null,
  identify:            { timeoutMs: 120_000, warningMs: 15_000 },
  otp:                 { timeoutMs: 120_000, warningMs: 15_000 },
  appointments:        { timeoutMs: 120_000, warningMs: 15_000 },
  token:               { timeoutMs: 30_000,  warningMs: 15_000 },
  'error-not-found':   { timeoutMs: 60_000,  warningMs: 15_000 },
  'error-otp-blocked': { timeoutMs: 60_000,  warningMs: 15_000 },
}
```

- [ ] **Step 2: Write failing test for useIdleTimer**

```typescript
// tests/hooks/useIdleTimer.test.tsx
import { renderHook, act } from '@testing-library/react'
import { useIdleTimer } from '../../src/kiosk/session/useIdleTimer'

beforeEach(() => { jest.useFakeTimers() })
afterEach(() => { jest.useRealTimers() })

test('calls onWarning before timeout', () => {
  const onWarning = jest.fn()
  const onTimeout = jest.fn()
  renderHook(() => useIdleTimer({
    config: { timeoutMs: 10_000, warningMs: 3_000 },
    onWarning,
    onTimeout,
  }))
  act(() => { jest.advanceTimersByTime(7_000) })
  expect(onWarning).toHaveBeenCalledTimes(1)
  expect(onTimeout).not.toHaveBeenCalled()
})

test('calls onTimeout after full duration', () => {
  const onWarning = jest.fn()
  const onTimeout = jest.fn()
  renderHook(() => useIdleTimer({
    config: { timeoutMs: 10_000, warningMs: 3_000 },
    onWarning,
    onTimeout,
  }))
  act(() => { jest.advanceTimersByTime(10_000) })
  expect(onTimeout).toHaveBeenCalledTimes(1)
})

test('does nothing when config is null', () => {
  const onWarning = jest.fn()
  const onTimeout = jest.fn()
  renderHook(() => useIdleTimer({
    config: null,
    onWarning,
    onTimeout,
  }))
  act(() => { jest.advanceTimersByTime(300_000) })
  expect(onWarning).not.toHaveBeenCalled()
  expect(onTimeout).not.toHaveBeenCalled()
})

test('resets timers on resetTimer call', () => {
  const onWarning = jest.fn()
  const onTimeout = jest.fn()
  const { result } = renderHook(() => useIdleTimer({
    config: { timeoutMs: 10_000, warningMs: 3_000 },
    onWarning,
    onTimeout,
  }))
  act(() => { jest.advanceTimersByTime(6_000) })
  act(() => { result.current.resetTimer() })
  act(() => { jest.advanceTimersByTime(6_000) })
  // Would have timed out at 10s without reset. After reset at 6s, warning fires at 6+7=13s.
  expect(onTimeout).not.toHaveBeenCalled()
})
```

- [ ] **Step 3: Run — verify FAIL**

```bash
npx react-scripts test --watchAll=false tests/hooks/useIdleTimer.test.tsx
```

- [ ] **Step 4: Implement useIdleTimer**

```typescript
// src/kiosk/session/useIdleTimer.ts
import { useEffect, useRef, useCallback } from 'react'
import { IdleConfig } from '../../types'

interface UseIdleTimerOptions {
  config: IdleConfig | null
  onWarning: () => void
  onTimeout: () => void
}

export function useIdleTimer({ config, onWarning, onTimeout }: UseIdleTimerOptions) {
  const warningTimer = useRef<ReturnType<typeof setTimeout> | null>(null)
  const timeoutTimer = useRef<ReturnType<typeof setTimeout> | null>(null)
  const warningFired = useRef(false)

  const clearTimers = useCallback(() => {
    if (warningTimer.current) clearTimeout(warningTimer.current)
    if (timeoutTimer.current) clearTimeout(timeoutTimer.current)
    warningTimer.current = null
    timeoutTimer.current = null
  }, [])

  const startTimers = useCallback(() => {
    if (!config) return
    clearTimers()
    warningFired.current = false

    const warningDelay = config.timeoutMs - config.warningMs
    warningTimer.current = setTimeout(() => {
      warningFired.current = true
      onWarning()
    }, warningDelay)

    timeoutTimer.current = setTimeout(() => {
      onTimeout()
    }, config.timeoutMs)
  }, [config, onWarning, onTimeout, clearTimers])

  const resetTimer = useCallback(() => {
    warningFired.current = false
    startTimers()
  }, [startTimers])

  // Start timers on mount / config change
  useEffect(() => {
    startTimers()
    return clearTimers
  }, [startTimers, clearTimers])

  // Listen for user activity to reset timers
  useEffect(() => {
    if (!config) return

    const handleActivity = () => { resetTimer() }
    const events = ['pointerdown', 'pointermove', 'keydown'] as const
    events.forEach(e => document.addEventListener(e, handleActivity, { passive: true }))
    return () => {
      events.forEach(e => document.removeEventListener(e, handleActivity))
    }
  }, [config, resetTimer])

  return { resetTimer }
}
```

- [ ] **Step 5: Run — verify PASS**

```bash
npx react-scripts test --watchAll=false tests/hooks/useIdleTimer.test.tsx
```

- [ ] **Step 6: Create StillThereOverlay**

```typescript
// src/kiosk/session/StillThereOverlay.tsx
import { useEffect, useState } from 'react'
import { motion, AnimatePresence } from 'framer-motion'

interface Props {
  visible: boolean
  secondsLeft: number
  onDismiss: () => void
}

export function StillThereOverlay({ visible, secondsLeft, onDismiss }: Props) {
  return (
    <AnimatePresence>
      {visible && (
        <motion.div
          initial={{ opacity: 0 }}
          animate={{ opacity: 1 }}
          exit={{ opacity: 0 }}
          transition={{ duration: 0.2 }}
          className="absolute inset-0 z-50 bg-black/60 flex items-center justify-center"
          onPointerDown={onDismiss}
        >
          <motion.div
            initial={{ scale: 0.9, opacity: 0 }}
            animate={{ scale: 1, opacity: 1 }}
            exit={{ scale: 0.9, opacity: 0 }}
            className="bg-bg-card rounded-[20px] p-8 text-center max-w-[280px] shadow-xl"
            onPointerDown={e => e.stopPropagation()}
          >
            <p className="text-4xl mb-4">👋</p>
            <h2 className="font-['Bricolage_Grotesque'] font-bold text-lg text-text-primary mb-2">
              Are you still there?
            </h2>
            <p className="text-sm text-text-secondary mb-4">
              Touch anywhere to continue
            </p>
            <p className="text-xs text-text-secondary">
              Returning to home in <span className="font-bold text-brand">{secondsLeft}s</span>
            </p>
            <button
              onClick={onDismiss}
              className="mt-4 w-full bg-brand text-white rounded-[12px] py-3 font-semibold text-sm"
              style={{ minHeight: '48px' }}
            >
              I'm still here
            </button>
          </motion.div>
        </motion.div>
      )}
    </AnimatePresence>
  )
}
```

- [ ] **Step 7: Create SessionProvider**

```typescript
// src/kiosk/session/SessionProvider.tsx
import { createContext, useContext, useState, useCallback, useRef, useEffect } from 'react'
import { KioskSession, IdleConfig } from '../../types'
import { defaultIdleConfig } from './idleConfig'
import { useIdleTimer } from './useIdleTimer'
import { StillThereOverlay } from './StillThereOverlay'

interface SessionContextValue {
  session: KioskSession | null
  startSession: () => void
  endSession: (reason: 'completed' | 'timeout' | 'manual') => void
  setCurrentScreen: (screen: string) => void
}

const SessionContext = createContext<SessionContextValue | null>(null)

export function useSession() {
  const ctx = useContext(SessionContext)
  if (!ctx) throw new Error('useSession must be used within SessionProvider')
  return ctx
}

// Module-level getter for service layer (outside React tree)
let _currentSessionId = ''
export function getSessionId() { return _currentSessionId }

interface Props {
  children: React.ReactNode
  onReset: () => void
}

export function SessionProvider({ children, onReset }: Props) {
  const [session, setSession] = useState<KioskSession | null>(null)
  const [currentIdleConfig, setCurrentIdleConfig] = useState<IdleConfig | null>(null)
  const [showWarning, setShowWarning] = useState(false)
  const [warningCountdown, setWarningCountdown] = useState(0)
  const countdownRef = useRef<ReturnType<typeof setInterval> | null>(null)

  const endSession = useCallback((reason: 'completed' | 'timeout' | 'manual') => {
    setSession(null)
    setCurrentIdleConfig(null)
    setShowWarning(false)
    _currentSessionId = ''
    if (countdownRef.current) clearInterval(countdownRef.current)
    sessionStorage.clear()
    onReset()
  }, [onReset])

  const startSession = useCallback(() => {
    const sessionId = crypto.randomUUID()
    _currentSessionId = sessionId
    setSession({ sessionId, startedAt: Date.now(), currentScreen: 'identify' })
    setCurrentIdleConfig(defaultIdleConfig['identify'] ?? null)
  }, [])

  const setCurrentScreen = useCallback((screen: string) => {
    setSession(prev => prev ? { ...prev, currentScreen: screen } : prev)
    setCurrentIdleConfig(defaultIdleConfig[screen] ?? null)
    setShowWarning(false)
    if (countdownRef.current) clearInterval(countdownRef.current)
  }, [])

  const handleWarning = useCallback(() => {
    if (!currentIdleConfig) return
    setShowWarning(true)
    const seconds = Math.ceil(currentIdleConfig.warningMs / 1000)
    setWarningCountdown(seconds)
    countdownRef.current = setInterval(() => {
      setWarningCountdown(prev => {
        if (prev <= 1) {
          if (countdownRef.current) clearInterval(countdownRef.current)
          return 0
        }
        return prev - 1
      })
    }, 1000)
  }, [currentIdleConfig])

  const handleTimeout = useCallback(() => {
    endSession('timeout')
  }, [endSession])

  const handleDismissWarning = useCallback(() => {
    setShowWarning(false)
    if (countdownRef.current) clearInterval(countdownRef.current)
  }, [])

  useIdleTimer({
    config: session ? currentIdleConfig : null,
    onWarning: handleWarning,
    onTimeout: handleTimeout,
  })

  const value: SessionContextValue = {
    session,
    startSession,
    endSession,
    setCurrentScreen,
  }

  return (
    <SessionContext.Provider value={value}>
      {children}
      <StillThereOverlay
        visible={showWarning}
        secondsLeft={warningCountdown}
        onDismiss={handleDismissWarning}
      />
    </SessionContext.Provider>
  )
}
```

- [ ] **Step 8: Commit**

```bash
git add src/kiosk/session/ tests/hooks/useIdleTimer.test.tsx
git commit -m "feat: session management — idle timer, StillThereOverlay, SessionProvider"
```

---

## Task 4: Error & Offline Handling

**Files:**
- Create: `src/kiosk/errors/useApiCall.ts`
- Create: `src/kiosk/errors/useNetworkStatus.ts`
- Create: `src/kiosk/errors/KioskErrorBoundary.tsx`
- Create: `src/kiosk/errors/ErrorRecoveryScreen.tsx`
- Create: `src/kiosk/errors/OfflineOverlay.tsx`
- Create: `tests/hooks/useApiCall.test.tsx`

- [ ] **Step 1: Write failing test for useApiCall**

```typescript
// tests/hooks/useApiCall.test.tsx
import { renderHook, act, waitFor } from '@testing-library/react'
import { useApiCall } from '../../src/kiosk/errors/useApiCall'

test('returns loading=true during execution', async () => {
  const { result } = renderHook(() => useApiCall<string>())
  let resolvePromise: (v: string) => void
  const promise = new Promise<string>(r => { resolvePromise = r })

  act(() => { result.current.execute(() => promise) })
  expect(result.current.state.loading).toBe(true)

  await act(async () => { resolvePromise!('done') })
  expect(result.current.state.loading).toBe(false)
  expect(result.current.state.data).toBe('done')
})

test('captures error on rejection', async () => {
  const { result } = renderHook(() => useApiCall<string>())

  await act(async () => {
    await result.current.execute(() => Promise.reject(new TypeError('Failed to fetch')))
  })
  expect(result.current.state.error).not.toBeNull()
  expect(result.current.state.error?.kind).toBe('network')
  expect(result.current.state.error?.retryable).toBe(true)
})

test('times out after configured duration', async () => {
  jest.useFakeTimers()
  const { result } = renderHook(() => useApiCall<string>({ timeoutMs: 1000 }))

  const neverResolves = new Promise<string>(() => {})
  act(() => { result.current.execute(() => neverResolves) })

  await act(async () => { jest.advanceTimersByTime(1100) })

  expect(result.current.state.error?.kind).toBe('timeout')
  jest.useRealTimers()
})
```

- [ ] **Step 2: Run — verify FAIL**

```bash
npx react-scripts test --watchAll=false tests/hooks/useApiCall.test.tsx
```

- [ ] **Step 3: Implement useApiCall**

```typescript
// src/kiosk/errors/useApiCall.ts
import { useState, useRef, useCallback } from 'react'
import { ApiError, ApiErrorKind, ApiCallState } from '../../types'

interface UseApiCallOptions {
  timeoutMs?: number
}

function classifyError(err: unknown): ApiError {
  if (err instanceof DOMException && err.name === 'AbortError') {
    return { kind: 'timeout', message: 'Request timed out', retryable: true }
  }
  if (err instanceof TypeError) {
    return { kind: 'network', message: err.message || 'Network error', retryable: true }
  }
  if (err instanceof Response || (err && typeof err === 'object' && 'status' in err)) {
    const status = (err as any).status as number
    const kind: ApiErrorKind = status >= 500 ? 'server' : 'client'
    return { kind, status, message: `HTTP ${status}`, retryable: status >= 500 }
  }
  return { kind: 'unknown', message: String(err), retryable: false }
}

export function useApiCall<T>(options?: UseApiCallOptions) {
  const timeoutMs = options?.timeoutMs ?? 10_000
  const [state, setState] = useState<ApiCallState<T>>({ data: null, error: null, loading: false })
  const abortRef = useRef<AbortController | null>(null)

  const execute = useCallback(async (fn: (signal?: AbortSignal) => Promise<T>): Promise<T | null> => {
    // Abort previous request if still in flight
    abortRef.current?.abort()
    const controller = new AbortController()
    abortRef.current = controller

    setState({ data: null, error: null, loading: true })

    const timer = setTimeout(() => controller.abort(), timeoutMs)

    try {
      const data = await fn(controller.signal)
      clearTimeout(timer)
      if (!controller.signal.aborted) {
        setState({ data, error: null, loading: false })
      }
      return data
    } catch (err) {
      clearTimeout(timer)
      if (!controller.signal.aborted || (err instanceof DOMException && err.name === 'AbortError')) {
        const error = classifyError(err)
        setState({ data: null, error, loading: false })
      }
      return null
    }
  }, [timeoutMs])

  const reset = useCallback(() => {
    setState({ data: null, error: null, loading: false })
  }, [])

  const abort = useCallback(() => {
    abortRef.current?.abort()
  }, [])

  return { execute, state, reset, abort }
}
```

- [ ] **Step 4: Run — verify PASS**

```bash
npx react-scripts test --watchAll=false tests/hooks/useApiCall.test.tsx
```

- [ ] **Step 5: Create useNetworkStatus**

```typescript
// src/kiosk/errors/useNetworkStatus.ts
import { useState, useEffect } from 'react'

export function useNetworkStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine)

  useEffect(() => {
    const goOnline = () => setIsOnline(true)
    const goOffline = () => setIsOnline(false)
    window.addEventListener('online', goOnline)
    window.addEventListener('offline', goOffline)

    // iPad Safari's navigator.onLine is unreliable — periodic health check
    const interval = setInterval(async () => {
      try {
        const res = await fetch('/health', { method: 'HEAD', cache: 'no-store' })
        setIsOnline(res.ok)
      } catch {
        setIsOnline(false)
      }
    }, 30_000)

    return () => {
      window.removeEventListener('online', goOnline)
      window.removeEventListener('offline', goOffline)
      clearInterval(interval)
    }
  }, [])

  return { isOnline }
}
```

- [ ] **Step 6: Create ErrorRecoveryScreen**

```typescript
// src/kiosk/errors/ErrorRecoveryScreen.tsx
interface Props { onReset: () => void }

export function ErrorRecoveryScreen({ onReset }: Props) {
  return (
    <div className="p-6 flex flex-col items-center justify-center h-full text-center bg-bg-surface">
      <div className="w-16 h-16 rounded-full bg-error-bg flex items-center justify-center text-3xl mb-4">⚠️</div>
      <h2 className="font-['Bricolage_Grotesque'] font-bold text-xl text-text-primary mb-2">
        Something went wrong
      </h2>
      <p className="text-sm text-text-secondary leading-relaxed mb-6 max-w-[240px]">
        We've run into an issue. Please try again or speak to our receptionist.
      </p>
      <button
        onClick={onReset}
        style={{ minHeight: '60px' }}
        className="w-full bg-brand text-white rounded-[12px] px-6 py-3 font-semibold text-sm"
      >
        Return to Start
      </button>
    </div>
  )
}
```

- [ ] **Step 7: Create KioskErrorBoundary**

```typescript
// src/kiosk/errors/KioskErrorBoundary.tsx
import { Component, ReactNode } from 'react'
import { ErrorRecoveryScreen } from './ErrorRecoveryScreen'

interface Props {
  children: ReactNode
  onReset: () => void
}

interface State {
  hasError: boolean
}

export class KioskErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false }

  static getDerivedStateFromError(): State {
    return { hasError: true }
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    // In production, this would send to telemetry
    console.error('[KioskErrorBoundary]', error, info.componentStack)
  }

  handleReset = () => {
    this.setState({ hasError: false })
    this.props.onReset()
  }

  render() {
    if (this.state.hasError) {
      return <ErrorRecoveryScreen onReset={this.handleReset} />
    }
    return this.props.children
  }
}
```

- [ ] **Step 8: Create OfflineOverlay**

```typescript
// src/kiosk/errors/OfflineOverlay.tsx
import { motion, AnimatePresence } from 'framer-motion'

interface Props { visible: boolean }

export function OfflineOverlay({ visible }: Props) {
  return (
    <AnimatePresence>
      {visible && (
        <motion.div
          initial={{ opacity: 0 }}
          animate={{ opacity: 1 }}
          exit={{ opacity: 0 }}
          className="absolute inset-0 z-40 bg-black/70 flex items-center justify-center"
        >
          <div className="bg-bg-card rounded-[20px] p-8 text-center max-w-[280px] shadow-xl">
            <div className="text-4xl mb-4 animate-pulse">📶</div>
            <h2 className="font-['Bricolage_Grotesque'] font-bold text-lg text-text-primary mb-2">
              Reconnecting...
            </h2>
            <p className="text-sm text-text-secondary leading-relaxed">
              Please wait while we restore the connection. If this persists, please speak to our receptionist.
            </p>
          </div>
        </motion.div>
      )}
    </AnimatePresence>
  )
}
```

- [ ] **Step 9: Commit**

```bash
git add src/kiosk/errors/ tests/hooks/useApiCall.test.tsx
git commit -m "feat: error & offline handling — useApiCall, ErrorBoundary, OfflineOverlay"
```

---

## Task 5: Telemetry + Device Auth

**Files:**
- Create: `src/kiosk/telemetry/telemetry.ts`
- Create: `src/kiosk/telemetry/telemetry.mock.ts`
- Create: `src/kiosk/telemetry/telemetry.real.ts`
- Create: `src/kiosk/telemetry/index.ts`
- Create: `src/kiosk/telemetry/useTelemetry.ts`
- Create: `src/kiosk/telemetry/deviceConfig.ts`
- Create: `src/kiosk/services/apiHeaders.ts`

- [ ] **Step 1: Create deviceConfig.ts**

```typescript
// src/kiosk/telemetry/deviceConfig.ts
import { DeviceConfig } from '../../types'

let _config: DeviceConfig | null = null

export function loadDeviceConfig(): DeviceConfig {
  if (_config) return _config
  _config = {
    deviceId:   process.env.REACT_APP_DEVICE_ID   || 'dev-kiosk',
    clinicId:   process.env.REACT_APP_CLINIC_ID    || 'dev-clinic',
    clinicName: process.env.REACT_APP_CLINIC_NAME  || 'Dev Clinic',
  }
  return _config
}

export function getDeviceConfig(): DeviceConfig {
  return _config ?? loadDeviceConfig()
}
```

- [ ] **Step 2: Create apiHeaders.ts**

```typescript
// src/kiosk/services/apiHeaders.ts
import { getDeviceConfig } from '../telemetry/deviceConfig'
import { getSessionId } from '../session/SessionProvider'

export function createApiHeaders(): Record<string, string> {
  const device = getDeviceConfig()
  return {
    'Content-Type': 'application/json',
    'X-Device-Id': device.deviceId,
    'X-Clinic-Id': device.clinicId,
    'X-Session-Id': getSessionId(),
  }
}
```

- [ ] **Step 3: Create telemetry interface**

```typescript
// src/kiosk/telemetry/telemetry.ts
// Re-export the TelemetryService interface from types for clarity.
// This file exists as the "contract" file parallel to services/api.ts.
export type { TelemetryService, TelemetryEvent, TelemetryEventName } from '../../types'
```

- [ ] **Step 4: Create telemetry.mock.ts**

```typescript
// src/kiosk/telemetry/telemetry.mock.ts
import { TelemetryService, TelemetryEvent } from '../../types'

export const mockTelemetry: TelemetryService = {
  track(event: TelemetryEvent) {
    console.log(
      `[telemetry] ${event.event}`,
      event.screenName ? `screen=${event.screenName}` : '',
      event.durationMs ? `duration=${event.durationMs}ms` : '',
      event.metadata ? JSON.stringify(event.metadata) : ''
    )
  },
  flush: async () => {},
}
```

- [ ] **Step 5: Create telemetry.real.ts**

```typescript
// src/kiosk/telemetry/telemetry.real.ts
import { TelemetryService, TelemetryEvent } from '../../types'
import { getDeviceConfig } from './deviceConfig'

const BATCH_SIZE = 10
const FLUSH_INTERVAL_MS = 30_000

let buffer: TelemetryEvent[] = []
let flushTimer: number | null = null

export const realTelemetry: TelemetryService = {
  track(event: TelemetryEvent) {
    buffer.push(event)
    if (buffer.length >= BATCH_SIZE) this.flush()
    if (!flushTimer) {
      flushTimer = window.setInterval(() => this.flush(), FLUSH_INTERVAL_MS)
    }
  },

  async flush() {
    if (buffer.length === 0) return
    const batch = [...buffer]
    buffer = []
    try {
      // TODO: POST /kiosk/telemetry { events: TelemetryEvent[] }
      await fetch(`${process.env.REACT_APP_API_BASE_URL || ''}/kiosk/telemetry`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ events: batch }),
      })
    } catch {
      // Best-effort: re-queue on failure, cap at 100
      buffer.unshift(...batch)
      if (buffer.length > 100) buffer = buffer.slice(-100)
    }
  },
}
```

- [ ] **Step 6: Create telemetry env switcher**

```typescript
// src/kiosk/telemetry/index.ts
import { mockTelemetry } from './telemetry.mock'
import { realTelemetry } from './telemetry.real'

export const telemetry = process.env.REACT_APP_USE_REAL_API === 'true'
  ? realTelemetry
  : mockTelemetry
```

- [ ] **Step 7: Create useTelemetry hook**

```typescript
// src/kiosk/telemetry/useTelemetry.ts
import { useCallback } from 'react'
import { TelemetryEventName } from '../../types'
import { telemetry } from './index'
import { getDeviceConfig } from './deviceConfig'
import { getSessionId } from '../session/SessionProvider'

export function useTelemetry() {
  const trackEvent = useCallback((
    event: TelemetryEventName,
    extra?: { screenName?: string; durationMs?: number; metadata?: Record<string, string | number | boolean> }
  ) => {
    const device = getDeviceConfig()
    telemetry.track({
      event,
      timestamp: Date.now(),
      sessionId: getSessionId(),
      deviceId: device.deviceId,
      clinicId: device.clinicId,
      screenName: extra?.screenName,
      durationMs: extra?.durationMs,
      metadata: extra?.metadata,
    })
  }, [])

  return { trackEvent }
}
```

- [ ] **Step 8: Verify TypeScript compiles**

```bash
npx tsc --noEmit
```

- [ ] **Step 9: Commit**

```bash
git add src/kiosk/telemetry/ src/kiosk/services/apiHeaders.ts
git commit -m "feat: telemetry (mock+real), device config, API headers with session+device IDs"
```

---

## Task 6: Service Layer (API Contract + Raw Types + Adapter + Mock + Real)

**Files:**
- Create: `src/kiosk/services/api.ts`
- Create: `src/kiosk/services/api-raw.ts`
- Create: `src/kiosk/services/api.mock.ts`
- Create: `src/kiosk/services/api.real.ts`
- Create: `src/kiosk/services/index.ts`

> **Multi-team API ownership:** The kiosk assembles data from two backend teams — Visit (patient identity + orders) and TatvaPractice (cabin/location). The `KioskAPI` interface is the clean contract the screens code against. `api-raw.ts` holds the raw API response shapes and transformer functions — the only place that knows actual field names from each team. `api.real.ts` is the orchestration layer that calls both backends and assembles their responses. `api.mock.ts` returns pre-assembled frontend types directly with no transformation needed. When a team shares their API contract, only `api-raw.ts` and `api.real.ts` change — zero component changes.

> **Env vars:** Two base URLs — one per team. Update `.env.development`:
> ```
> REACT_APP_VISIT_API_URL=https://api.visithealth.in
> REACT_APP_TP_API_URL=https://api.tatvapractice.in
> ```

- [ ] **Step 1: Create api-raw.ts — raw response shapes + transformer functions**

```typescript
// src/kiosk/services/api-raw.ts
//
// PURPOSE: This file is the ONLY place in the codebase that knows the actual field
// names that Visit and TatvaPractice APIs return. Everything else codes against
// the clean types in ../../types/index.ts.
//
// HOW TO USE WHEN BACKEND CONTRACTS ARRIVE:
// 1. Fill in the Raw* interfaces with actual field names from each team's API docs/samples
// 2. Update the transformer functions below to map raw fields → frontend fields
// 3. Nothing else in the codebase changes — screens never touch raw field names
//
// CURRENTLY: field names are PLACEHOLDERS marked with "PLACEHOLDER:" comments.
// Replace them when Visit (Shashvat/Rishanku) and TP (Rishanku) share their contracts.

import type { Patient, Appointment, CabinDetailMap } from '../../types'

// ── Visit API: GetPatientDetails response ─────────────────────
// Called with phone number → returns patient identity + the beneficiary phone for OTP

export interface RawVisitPatient {
  // PLACEHOLDER: actual field names TBD — examples of what they might be:
  id: string           // could be: 'uhid', 'patient_id', 'patientId', 'user_id'
  name: string         // could be: 'full_name', 'display_name', 'patient_name', 'name'
  phone: string        // the beneficiary number OTP was sent to — could be: 'otp_mobile',
                       // 'registered_phone', 'beneficiary_mobile', 'contact_number'
}

// ── Visit API: GetPatientOrders response ──────────────────────
// Called with patientId → returns today's appointments/orders

export interface RawVisitOrder {
  // PLACEHOLDER: actual field names TBD
  id: string           // could be: 'order_id', 'appointment_id', 'booking_id'
  order_type: string   // could be: 'type', 'appointment_type', 'service_type'
                       // expected values: 'CONSULTATION', 'LAB', 'PATHOLOGY', 'AHC' — TBD
  doctor_name?: string // for consultations — field name TBD
  speciality?: string  // note: Visit may spell as 'speciality' or 'specialty' — check contract
  test_names?: string[]
  scheduled_time: string  // ISO string or HH:MM — format TBD
  prep_note?: string
}

// ── TatvaPractice API: AppointmentCabinDetails response ───────
// Called per appointment → returns cabin, floor, routing instructions
// This is TP-owned data — Visit does not know cabin assignments

export interface RawTPCabinDetails {
  // PLACEHOLDER: actual field names TBD — examples:
  appointment_id: string   // how TP references the appointment — may differ from Visit's id
  cabin_number?: string    // could be: 'cabin', 'cabin_no', 'room', 'room_number'
  floor?: string           // could be: 'floor_number', 'level', 'floor_name'
  waiting_area_instructions?: string  // could be: 'routing_instructions', 'directions'
}

// ── Transformer functions ─────────────────────────────────────
// These are called in api.real.ts to convert raw API payloads → frontend types.
// The rest of the codebase never calls these directly.

export function toPatient(raw: RawVisitPatient): Omit<Patient, 'appointments'> {
  return {
    id: raw.id,           // ← update 'raw.id' to 'raw.uhid' etc. when contract arrives
    fullName: raw.name,   // ← update 'raw.name' to actual field name
    phone: raw.phone,     // ← update 'raw.phone' to actual field name
  }
}

export function toAppointment(raw: RawVisitOrder, patientId: string, patientName: string): Appointment {
  // Map Visit's order_type string to our AppointmentType union
  const lc = (raw.order_type ?? '').toLowerCase()
  const type = lc.includes('consult') ? 'consultation'
    : (lc.includes('lab') || lc.includes('path') || lc.includes('ahc')) ? 'lab'
    : 'lab'  // default to lab if unknown — update when actual values are known

  return {
    id: raw.id,
    type,
    patientName,
    patientId,
    doctorName: raw.doctor_name,
    specialty: raw.speciality,
    testNames: raw.test_names,
    time: raw.scheduled_time,
    prepNote: raw.prep_note,
  }
}

export function toCabinDetailMap(rawList: RawTPCabinDetails[]): CabinDetailMap {
  // Map array → object keyed by appointment_id for O(1) lookup in S3
  return Object.fromEntries(
    rawList.map(raw => [
      raw.appointment_id,
      {
        cabin: raw.cabin_number,
        floor: raw.floor,
        waitingAreaInstructions: raw.waiting_area_instructions,
      }
    ])
  )
}
```

- [ ] **Step 2: Define the KioskAPI interface**

```typescript
// src/kiosk/services/api.ts
import type {
  VerifyPhoneResult, OTPResult, QRScanResult,
  CheckInResult, Patient, Appointment, CabinDetailMap, Doctor
} from '../../types'

export interface KioskAPI {
  // ── Identity flow ──────────────────────────────────────────
  // Visit-owned: GetPatientDetails by phone number
  verifyPhone(phone: string): Promise<VerifyPhoneResult>
  verifyOTP(beneficiaryPhone: string, otp: string): Promise<OTPResult>
  resendOTP(beneficiaryPhone: string): Promise<void>
  scanQR(qrData: string): Promise<QRScanResult>

  // ── Appointment data ───────────────────────────────────────
  // Visit-owned: GetPatientOrders by patientId
  // Returns base appointment data (doctor name, time). No cabin info.
  getAppointmentsForPatient(patientId: string): Promise<Appointment[]>

  // TP-owned: AppointmentCabinDetails per appointment
  // Returns cabin/location data. Called in parallel for all appointment IDs.
  // Non-blocking — S3 "Check In Now" does NOT wait for this to resolve.
  getAppointmentCabinDetails(appointmentIds: string[]): Promise<CabinDetailMap>

  // ── Check-in ───────────────────────────────────────────────
  // Visit-owned: generates queue token(s) and checks phlebotomist availability (PRD §4.1.3 K4)
  checkIn(patientId: string, appointmentIds: string[]): Promise<CheckInResult>

  // ── Walk-in flows (B–E) ────────────────────────────────────
  lookupPolicy(phone: string): Promise<{ found: boolean; policyId?: string; benefits?: object }>
  lookupBeneficiaries(policyId: string): Promise<{ beneficiaries: Patient[] }>
  getDoctors(specialty?: string): Promise<Doctor[]>
  registerNewPatient(details: { name: string; age: number; gender: string; phone: string }): Promise<Patient>
}
```

- [ ] **Step 3: Implement api.mock.ts**

```typescript
// src/kiosk/services/api.mock.ts
// Mock returns pre-assembled frontend types directly — no transformers needed.
// Cabin data is returned fully populated so S3 can show the complete card in demo.

import type { KioskAPI } from './api'
import type { VerifyPhoneResult, QRScanResult, CheckInResult, CabinDetailMap } from '../../types'
import { mockPatientA, mockPatientAConsultOnly } from '../data/mockPatients'

function delay(ms: number) { return new Promise(r => setTimeout(r, ms)) }

// Test credentials (shown in demo info panel):
// Phone: 9876543210 → Priya Sharma (consult + lab) | OTP: 123456
// Phone: 9123456780 → Rajan Pillai (consult only)  | OTP: 123456
// Any other phone → not found | Any other OTP → wrong

export const mockAPI: KioskAPI = {
  verifyPhone: async (phone): Promise<VerifyPhoneResult> => {
    await delay(600)
    if (phone === '9876543210')
      return { found: true, patient: mockPatientA, beneficiaryPhone: mockPatientA.phone }
    if (phone === '9123456780')
      return { found: true, patient: mockPatientAConsultOnly, beneficiaryPhone: mockPatientAConsultOnly.phone }
    return { found: false }
  },

  verifyOTP: async (_beneficiaryPhone, otp) => {
    await delay(400)
    return { valid: otp === '123456' }
  },

  resendOTP: async () => { await delay(300) },

  scanQR: async (): Promise<QRScanResult> => {
    await delay(500)
    return { found: true, patient: mockPatientA }
  },

  // Mock returns appointments from the patient object already in mock data
  getAppointmentsForPatient: async (patientId) => {
    await delay(600)
    const patient = [mockPatientA, mockPatientAConsultOnly].find(p => p.id === patientId)
    return patient?.appointments ?? []
  },

  // Mock returns fully populated cabin data — all appointments have cabin info in demo
  getAppointmentCabinDetails: async (appointmentIds): Promise<CabinDetailMap> => {
    await delay(400)
    return Object.fromEntries(
      appointmentIds.map(id => [id, {
        cabin:    id.includes('lab') ? 'Lab Collection Area' : 'Cabin 3',
        floor:    id.includes('lab') ? 'Ground Floor' : 'Floor 1',
        waitingAreaInstructions: id.includes('lab')
          ? 'Proceed to Lab Collection Area, Ground Floor'
          : 'Head to the waiting area near Cabin 3, Floor 1',
      }])
    )
  },

  checkIn: async (_patientId, appointmentIds): Promise<CheckInResult> => {
    await delay(800)
    const tokens = appointmentIds.map((id, i) => {
      const isLab = id.includes('lab') || i > 0
      return {
        tokenNumber: isLab ? `L-${String(12 + i).padStart(3, '0')}` : `A-${String(47 + i).padStart(3, '0')}`,
        appointmentType: isLab ? 'lab' as const : 'consultation' as const,
        doctorName: isLab ? undefined : 'Dr. Anjali Mehta',
        location: isLab ? 'Lab Collection Area, Ground Floor' : 'Cabin 3, Floor 1',
        estimatedWaitMinutes: isLab ? 8 : 12,
        aheadCount: isLab ? 1 : 3,
      }
    })
    return { success: true, tokens }
  },

  lookupPolicy:        async () => ({ found: false }),
  lookupBeneficiaries: async () => ({ beneficiaries: [] }),
  getDoctors:          async () => [],
  registerNewPatient:  async () => { throw new Error('Not implemented in mock') },
}
```

- [ ] **Step 4: Create api.real.ts — orchestration layer with two base URLs**

```typescript
// src/kiosk/services/api.real.ts
//
// ORCHESTRATION PATTERN:
// This file calls Visit APIs and TP APIs. It is the only file that knows
// there are two backends. The screens only call KioskAPI methods — they
// are unaware of team ownership or base URL differences.
//
// Call chain for S3 (AppointmentList screen):
//   verifyPhone(phone)                          ← Visit: GetPatientDetails
//     └─ getAppointmentsForPatient(patientId)   ← Visit: GetPatientOrders (serial — needs patientId)
//          └─ getAppointmentCabinDetails(ids)   ← TP: AppointmentCabinDetails (parallel per id)
//
// The last step (cabin details) is called from the S3 component — non-blocking.
// Check-in does NOT wait for cabin details.

import type { KioskAPI } from './api'
import type { Appointment } from '../../types'
import { createApiHeaders } from './apiHeaders'
import { toPatient, toAppointment, toCabinDetailMap } from './api-raw'
import type { RawVisitPatient, RawVisitOrder, RawTPCabinDetails } from './api-raw'

// Two base URLs — one per team. Both read from env at build time.
const VISIT_BASE = process.env.REACT_APP_VISIT_API_URL ?? ''
const TP_BASE    = process.env.REACT_APP_TP_API_URL    ?? ''

// ── HTTP helpers (one per base URL) ──────────────────────────

async function visitPost<T>(path: string, body: object, signal?: AbortSignal): Promise<T> {
  const res = await fetch(`${VISIT_BASE}${path}`, {
    method: 'POST',
    headers: createApiHeaders(),
    body: JSON.stringify(body),
    signal,
  })
  if (!res.ok) {
    const err = new Error(`Visit API ${res.status}: ${path}`) as any
    err.status = res.status
    throw err
  }
  return res.json()
}

async function tpGet<T>(path: string, signal?: AbortSignal): Promise<T> {
  const res = await fetch(`${TP_BASE}${path}`, {
    headers: createApiHeaders(),
    signal,
  })
  if (!res.ok) {
    const err = new Error(`TP API ${res.status}: ${path}`) as any
    err.status = res.status
    throw err
  }
  return res.json()
}

// ── KioskAPI implementation ───────────────────────────────────

export const realAPI: KioskAPI = {

  verifyPhone: async (phone) => {
    // TODO: replace path with actual Visit endpoint — contract TBD with Shashvat/Rishanku
    const raw = await visitPost<RawVisitPatient | { found: false }>('/kiosk/lookup-patient', { phone })
    if (!('id' in raw)) return { found: false }
    return {
      found: true,
      patient: { ...toPatient(raw), appointments: [] },
      beneficiaryPhone: raw.phone,
    }
  },

  verifyOTP: (beneficiaryPhone, otp) =>
    // TODO: replace path with actual Visit endpoint
    visitPost('/kiosk/verify-otp', { beneficiaryPhone, otp }),

  resendOTP: (beneficiaryPhone) =>
    // TODO: replace path with actual Visit endpoint
    visitPost('/kiosk/resend-otp', { beneficiaryPhone }),

  scanQR: async (qrData) => {
    // TODO: replace path with actual Visit endpoint
    const raw = await visitPost<RawVisitPatient | { found: false }>('/kiosk/scan-qr', { qrData })
    if (!('id' in raw)) return { found: false, reason: 'patient_not_found' }
    return { found: true, patient: { ...toPatient(raw), appointments: [] } }
  },

  // Step 1 of the two-step appointment fetch — serial (patientId required from verifyPhone)
  getAppointmentsForPatient: async (patientId) => {
    // TODO: replace path + request shape with actual Visit GetPatientOrders endpoint
    const today = new Date().toISOString().slice(0, 10)  // 'YYYY-MM-DD'
    const raw = await visitPost<{ orders: RawVisitOrder[] }>(
      '/kiosk/patient-orders', { patientId, date: today }
    )
    // Transform each raw order → frontend Appointment type via api-raw.ts transformer
    return raw.orders.map(o => toAppointment(o, patientId, ''))
  },

  // Step 2 — parallel fan-out to TP per appointment. Called from S3 on mount, non-blocking.
  // If TP's API is down, returns {} — S3 shows "See receptionist" for cabin info.
  getAppointmentCabinDetails: async (appointmentIds) => {
    // TODO: replace path with actual TP endpoint — contract TBD with Rishanku's team
    try {
      const rawList = await Promise.all(
        appointmentIds.map(id => tpGet<RawTPCabinDetails>(`/appointments/${id}/cabin-details`))
      )
      return toCabinDetailMap(rawList)
    } catch {
      // TP cabin API failure is non-fatal — return empty so S3 shows fallback text
      return {}
    }
  },

  // Visit-owned check-in — also triggers phlebotomist availability check (PRD §4.1.3 K4)
  checkIn: (patientId, appointmentIds) =>
    // TODO: replace path with actual Visit check-in endpoint
    visitPost('/kiosk/check-in', { patientId, appointmentIds }),

  lookupPolicy: (phone) => visitPost('/kiosk/lookup-policy', { phone }),
  lookupBeneficiaries: (policyId) => visitPost('/kiosk/lookup-beneficiaries', { policyId }),
  getDoctors: (specialty) => tpGet(`/doctors${specialty ? `?specialty=${specialty}` : ''}`),
  registerNewPatient: (details) => visitPost('/kiosk/register-patient', details),
}
```

- [ ] **Step 5: Create env switcher**

```typescript
// src/kiosk/services/index.ts
import { mockAPI } from './api.mock'
import { realAPI } from './api.real'

export const api = process.env.REACT_APP_USE_REAL_API === 'true' ? realAPI : mockAPI
```

- [ ] **Step 6: Verify TypeScript compiles**

```bash
npx tsc --noEmit
```

Expected: no errors. If `RawVisitPatient | { found: false }` causes a type narrowing issue, add a type guard function in `api-raw.ts`.

- [ ] **Step 7: Commit**

```bash
git add src/kiosk/services/
git commit -m "feat: API service layer — interface, adapter pattern (api-raw.ts), mock + real stubs"
```

---

## Task 7: Mock Data

**Files:**
- Create: `src/kiosk/data/mockAppointments.ts`
- Create: `src/kiosk/data/mockPatients.ts`

- [ ] **Step 1: Create mock appointments**

```typescript
// src/kiosk/data/mockAppointments.ts
import type { Appointment } from '../../types'

export const mockConsultation: Appointment = {
  id: 'appt-001',
  type: 'consultation',
  patientName: 'Priya Sharma',
  patientId: 'UHID-00123',
  doctorName: 'Dr. Anjali Mehta',
  specialty: 'General Physician',
  time: '10:45 AM',
  location: 'Cabin 3, Floor 1',
}

export const mockLabTest: Appointment = {
  id: 'appt-lab-001',
  type: 'lab',
  patientName: 'Priya Sharma',
  patientId: 'UHID-00123',
  testNames: ['Complete Blood Count (CBC)', 'Lipid Profile'],
  time: '11:30 AM',
  location: 'Lab Collection Area, Ground Floor',
  prepNote: 'Fasting required',
}
```

- [ ] **Step 2: Create mock patients**

```typescript
// src/kiosk/data/mockPatients.ts
import type { Patient } from '../../types'
import { mockConsultation, mockLabTest } from './mockAppointments'

export const mockPatientA: Patient = {
  id: 'UHID-00123',
  fullName: 'Priya Sharma',
  phone: '9876543210',
  appointments: [mockConsultation, mockLabTest],
}

export const mockPatientAConsultOnly: Patient = {
  id: 'UHID-00124',
  fullName: 'Rajan Pillai',
  phone: '9123456780',
  appointments: [{ ...mockConsultation, id: 'appt-002', patientName: 'Rajan Pillai', patientId: 'UHID-00124' }],
}
```

- [ ] **Step 3: Commit**

```bash
git add src/kiosk/data/
git commit -m "feat: mock data shapes (consumed by api.mock.ts only)"
```

---

## Task 8: PageLayout + FlowShell

**Files:**
- Create: `src/kiosk/layout/PageLayout.tsx`
- Create: `src/kiosk/layout/FlowShell.tsx`

- [ ] **Step 1: Create PageLayout**

```typescript
// src/kiosk/layout/PageLayout.tsx
import { ReactNode } from 'react'

interface PageLayoutProps {
  children: ReactNode
  scrollable?: boolean
  verticalCenter?: boolean
  className?: string
}

export function PageLayout({
  children,
  scrollable = true,
  verticalCenter = false,
  className = '',
}: PageLayoutProps) {
  return (
    <div
      className={`px-4 pt-4 pb-6 h-full flex flex-col
        ${scrollable ? 'overflow-y-auto' : 'overflow-hidden'}
        ${verticalCenter ? 'items-center justify-center' : ''}
        ${className}`}
    >
      {children}
    </div>
  )
}
```

- [ ] **Step 2: Create FlowShell**

```typescript
// src/kiosk/layout/FlowShell.tsx
import { ReactNode, useEffect } from 'react'
import { AnimatePresence, motion } from 'framer-motion'
import { KioskShell } from '../KioskShell'
import { useSession } from '../session/SessionProvider'

interface FlowShellProps {
  children: ReactNode
  stepKey: string
  progressPercent: number
}

export function FlowShell({ children, stepKey, progressPercent }: FlowShellProps) {
  const { setCurrentScreen } = useSession()

  useEffect(() => {
    setCurrentScreen(stepKey)
  }, [stepKey, setCurrentScreen])

  return (
    <KioskShell progressPercent={progressPercent}>
      <AnimatePresence mode="wait">
        <motion.div
          key={stepKey}
          initial={{ x: 40, opacity: 0 }}
          animate={{ x: 0, opacity: 1 }}
          exit={{ x: -40, opacity: 0 }}
          transition={{ duration: 0.2, ease: 'easeInOut' }}
          className="absolute inset-0"
        >
          {children}
        </motion.div>
      </AnimatePresence>
    </KioskShell>
  )
}
```

- [ ] **Step 3: Commit**

```bash
git add src/kiosk/layout/
git commit -m "feat: PageLayout + FlowShell — shared wrappers for all flows"
```

---

## Task 9: Shared Components — Button, Card, Badge

**Files:**
- Create: `src/kiosk/components/Button.tsx`
- Create: `src/kiosk/components/Card.tsx`
- Create: `src/kiosk/components/Badge.tsx`
- Create: `tests/components/Button.test.tsx`

- [ ] **Step 1: Write failing test for Button**

```typescript
// tests/components/Button.test.tsx
import { render, screen, fireEvent } from '@testing-library/react'
import { Button } from '../../src/kiosk/components/Button'

test('renders with label', () => {
  render(<Button variant="primary">Check In</Button>)
  expect(screen.getByRole('button', { name: 'Check In' })).toBeInTheDocument()
})

test('calls onClick when tapped', () => {
  const onClick = jest.fn()
  render(<Button variant="primary" onClick={onClick}>Go</Button>)
  fireEvent.click(screen.getByRole('button'))
  expect(onClick).toHaveBeenCalledTimes(1)
})

test('meets 60px minimum tap target for kiosk', () => {
  render(<Button variant="primary">Go</Button>)
  expect(screen.getByRole('button')).toHaveStyle({ minHeight: '60px' })
})
```

- [ ] **Step 2: Run — verify FAIL**

```bash
npx react-scripts test --watchAll=false tests/components/Button.test.tsx
```

- [ ] **Step 3: Implement Button (using Tailwind config tokens)**

```typescript
// src/kiosk/components/Button.tsx
import { ButtonHTMLAttributes } from 'react'

type Variant = 'primary' | 'secondary' | 'ghost'

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant: Variant
  fullWidth?: boolean
}

const styles: Record<Variant, string> = {
  primary:   'bg-brand text-white hover:bg-brand-dark shadow-[0_4px_14px_rgba(113,79,255,0.35)]',
  secondary: 'bg-bg-surface text-brand hover:bg-border-light',
  ghost:     'bg-transparent text-brand underline hover:opacity-70',
}

export function Button({ variant, fullWidth, className = '', children, ...props }: ButtonProps) {
  return (
    <button
      style={{ minHeight: '60px' }}
      className={`rounded-[12px] px-6 font-semibold font-['Inter'] text-sm tracking-wide
        transition-colors duration-150 cursor-pointer
        ${styles[variant]} ${fullWidth ? 'w-full' : ''} ${className}`}
      {...props}
    >
      {children}
    </button>
  )
}
```

- [ ] **Step 4: Run — verify PASS**

```bash
npx react-scripts test --watchAll=false tests/components/Button.test.tsx
```

- [ ] **Step 5: Create Card and Badge (using Tailwind config tokens)**

```typescript
// src/kiosk/components/Card.tsx
import { ReactNode } from 'react'

interface CardProps { children: ReactNode; accentColor?: string; className?: string }

export function Card({ children, accentColor, className = '' }: CardProps) {
  return (
    <div
      className={`bg-bg-card rounded-[14px] p-4 shadow-[0_2px_8px_rgba(113,79,255,0.06)] ${className}`}
      style={accentColor ? { borderLeft: `3px solid ${accentColor}` } : undefined}
    >
      {children}
    </div>
  )
}
```

```typescript
// src/kiosk/components/Badge.tsx
type BadgeVariant = 'consult' | 'lab' | 'success' | 'warning'

const styles: Record<BadgeVariant, string> = {
  consult: 'bg-border-light text-brand',
  lab:     'bg-error-bg text-accent',
  success: 'bg-success-bg text-success-text',
  warning: 'bg-error-bg text-accent',
}

export function Badge({ variant, children }: { variant: BadgeVariant; children: string }) {
  return (
    <span className={`text-[9px] font-semibold px-2 py-0.5 rounded-[6px] uppercase tracking-wide ${styles[variant]}`}>
      {children}
    </span>
  )
}
```

- [ ] **Step 6: Commit**

```bash
git add src/kiosk/components/Button.tsx src/kiosk/components/Card.tsx \
        src/kiosk/components/Badge.tsx tests/components/Button.test.tsx
git commit -m "feat: Button (TDD), Card, Badge — all using Tailwind config tokens"
```

---

## Task 10: Shared Components — NumericKeypad + OTPInput

**Files:**
- Create: `src/kiosk/components/NumericKeypad.tsx`
- Create: `src/kiosk/components/OTPInput.tsx`
- Create: `tests/components/NumericKeypad.test.tsx`
- Create: `tests/components/OTPInput.test.tsx`

- [ ] **Step 1: Write failing test for NumericKeypad**

```typescript
// tests/components/NumericKeypad.test.tsx
import { render, screen, fireEvent } from '@testing-library/react'
import { NumericKeypad } from '../../src/kiosk/components/NumericKeypad'

test('calls onKey with digit when tapped', () => {
  const onKey = jest.fn()
  render(<NumericKeypad onKey={onKey} />)
  fireEvent.click(screen.getByRole('button', { name: /^5/ }))
  expect(onKey).toHaveBeenCalledWith('5')
})

test('calls onKey with "del" for backspace', () => {
  const onKey = jest.fn()
  render(<NumericKeypad onKey={onKey} />)
  fireEvent.click(screen.getByRole('button', { name: /⌫/ }))
  expect(onKey).toHaveBeenCalledWith('del')
})
```

- [ ] **Step 2: Run — verify FAIL**

- [ ] **Step 3: Implement NumericKeypad**

```typescript
// src/kiosk/components/NumericKeypad.tsx
const KEYS = [
  { label: '1', sub: '' },    { label: '2', sub: 'ABC' },  { label: '3', sub: 'DEF' },
  { label: '4', sub: 'GHI' }, { label: '5', sub: 'JKL' },  { label: '6', sub: 'MNO' },
  { label: '7', sub: 'PQRS'}, { label: '8', sub: 'TUV' },  { label: '9', sub: 'WXYZ'},
  { label: '*', sub: '' },    { label: '0', sub: '' },      { label: '⌫', sub: '' },
]

interface NumericKeypadProps {
  onKey: (key: string) => void
  actionLabel?: string
  onAction?: () => void
}

export function NumericKeypad({ onKey, actionLabel, onAction }: NumericKeypadProps) {
  return (
    <div className="grid grid-cols-3 gap-1.5">
      {KEYS.map(({ label, sub }) => (
        <button
          key={label}
          style={{ minHeight: '48px' }}
          onClick={() => onKey(label === '⌫' ? 'del' : label)}
          className={`rounded-[10px] text-base font-semibold font-['Inter'] flex flex-col
            items-center justify-center gap-0.5 shadow-[0_1px_4px_rgba(0,0,0,0.08)]
            transition-colors
            ${label === '⌫' ? 'bg-border-light text-brand' : 'bg-bg-card text-text-primary hover:bg-border-light'}`}
        >
          <span>{label}</span>
          {sub && <span className="text-[7px] text-text-secondary font-normal tracking-wider">{sub}</span>}
        </button>
      ))}
      {actionLabel && onAction && (
        <button
          onClick={onAction}
          style={{ minHeight: '48px' }}
          className="col-span-3 rounded-[12px] bg-brand text-white text-sm font-semibold
                     shadow-[0_4px_12px_rgba(113,79,255,0.35)] hover:bg-brand-dark transition-colors"
        >
          {actionLabel}
        </button>
      )}
    </div>
  )
}
```

- [ ] **Step 4: Run NumericKeypad tests — verify PASS**

- [ ] **Step 5: Write failing tests for OTPInput**

```typescript
// tests/components/OTPInput.test.tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { OTPInput } from '../../src/kiosk/components/OTPInput'

test('renders 6 boxes', () => {
  render(<OTPInput value="" onChange={() => {}} />)
  expect(screen.getAllByRole('textbox')).toHaveLength(6)
})

test('calls onChange with accumulated digits', async () => {
  const onChange = jest.fn()
  render(<OTPInput value="" onChange={onChange} />)
  await userEvent.type(screen.getAllByRole('textbox')[0], '8')
  expect(onChange).toHaveBeenCalledWith('8')
})
```

- [ ] **Step 6: Run — verify FAIL**

- [ ] **Step 7: Implement OTPInput**

```typescript
// src/kiosk/components/OTPInput.tsx
import { useRef } from 'react'

interface OTPInputProps { value: string; onChange: (otp: string) => void }

export function OTPInput({ value, onChange }: OTPInputProps) {
  const refs = Array.from({ length: 6 }, () => useRef<HTMLInputElement>(null))

  const handleChange = (idx: number, char: string) => {
    const digit = char.replace(/\D/g, '').slice(-1)
    if (!digit) return
    const next = (value + digit).slice(0, 6)
    onChange(next)
    if (idx < 5) refs[idx + 1].current?.focus()
  }

  const handleKeyDown = (idx: number, e: React.KeyboardEvent) => {
    if (e.key === 'Backspace' && !value[idx]) {
      onChange(value.slice(0, -1))
      if (idx > 0) refs[idx - 1].current?.focus()
    }
  }

  return (
    <div className="flex gap-2 justify-center">
      {Array.from({ length: 6 }).map((_, i) => (
        <input
          key={i}
          ref={refs[i]}
          type="text"
          inputMode="numeric"
          maxLength={1}
          value={value[i] || ''}
          onChange={e => handleChange(i, e.target.value)}
          onKeyDown={e => handleKeyDown(i, e)}
          style={{ minHeight: '48px', width: '40px' }}
          className={`rounded-[10px] border-2 text-center text-xl font-bold font-['Inter']
            text-text-primary focus:outline-none
            ${i < value.length || i === value.length
              ? 'border-brand bg-[#faf8ff]'
              : 'border-border-light bg-bg-card'}`}
        />
      ))}
    </div>
  )
}
```

- [ ] **Step 8: Run OTPInput tests — verify PASS**

- [ ] **Step 9: Commit**

```bash
git add src/kiosk/components/NumericKeypad.tsx src/kiosk/components/OTPInput.tsx tests/components/
git commit -m "feat: NumericKeypad + OTPInput with tests — using Tailwind config tokens"
```

---

## Task 11: QueueTokenCard + ProgressBar + StepHeader

**Files:**
- Create: `src/kiosk/components/QueueTokenCard.tsx`
- Create: `src/kiosk/components/ProgressBar.tsx`
- Create: `src/kiosk/components/StepHeader.tsx`
- Create: `tests/components/QueueTokenCard.test.tsx`

- [ ] **Step 1: Write failing test for QueueTokenCard**

```typescript
// tests/components/QueueTokenCard.test.tsx
import { render, screen } from '@testing-library/react'
import { QueueTokenCard } from '../../src/kiosk/components/QueueTokenCard'
import type { QueueToken } from '../../src/types'

const token: QueueToken = {
  tokenNumber: 'A-047', appointmentType: 'consultation',
  doctorName: 'Dr. Anjali Mehta', location: 'Cabin 3, Floor 1',
  estimatedWaitMinutes: 12, aheadCount: 3,
}

test('renders token number', () => {
  render(<QueueTokenCard token={token} />)
  expect(screen.getByText('A-047')).toBeInTheDocument()
})

test('renders doctor name', () => {
  render(<QueueTokenCard token={token} />)
  expect(screen.getByText('Dr. Anjali Mehta')).toBeInTheDocument()
})

test('renders wait time', () => {
  render(<QueueTokenCard token={token} />)
  expect(screen.getByText(/12 min/)).toBeInTheDocument()
})
```

- [ ] **Step 2: Run — verify FAIL**

- [ ] **Step 3: Implement QueueTokenCard, ProgressBar, StepHeader**

```typescript
// src/kiosk/components/QueueTokenCard.tsx
import type { QueueToken } from '../../types'

export function QueueTokenCard({ token }: { token: QueueToken }) {
  return (
    <div className="bg-bg-card rounded-[16px] p-4 text-center border border-border-light
                    shadow-[0_4px_20px_rgba(113,79,255,0.10)]">
      <p className="text-[10px] uppercase tracking-widest text-text-secondary mb-1">Your Queue Token</p>
      <p className="font-['Bricolage_Grotesque'] text-5xl font-bold text-brand tracking-tight leading-none mb-2">
        {token.tokenNumber}
      </p>
      <div className="w-10 h-px bg-border-light mx-auto my-2" />
      {token.doctorName && (
        <p className="font-['Montserrat'] font-semibold text-sm text-text-primary mb-1">{token.doctorName}</p>
      )}
      <p className="text-xs text-text-secondary mb-3">{token.location}</p>
      <span className="inline-flex items-center gap-1 bg-bg-surface px-3 py-1 rounded-full
                       text-xs font-semibold text-brand">
        ⏱ ~{token.estimatedWaitMinutes} min wait · {token.aheadCount} ahead
      </span>
    </div>
  )
}
```

```typescript
// src/kiosk/components/ProgressBar.tsx
export function ProgressBar({ percent }: { percent: number }) {
  return (
    <div className="w-[90px] h-1 bg-border-light rounded-full overflow-hidden">
      <div
        className="h-full bg-brand rounded-full transition-all duration-400"
        style={{ width: `${percent}%` }}
      />
    </div>
  )
}
```

```typescript
// src/kiosk/components/StepHeader.tsx
interface StepHeaderProps { title: string; subtitle?: string }

export function StepHeader({ title, subtitle }: StepHeaderProps) {
  return (
    <div className="mb-4">
      <h1 className="font-['Bricolage_Grotesque'] font-bold text-xl text-text-primary leading-tight mb-1">
        {title}
      </h1>
      {subtitle && <p className="text-[11px] text-text-secondary">{subtitle}</p>}
    </div>
  )
}
```

- [ ] **Step 4: Run — verify PASS**

- [ ] **Step 5: Commit**

```bash
git add src/kiosk/components/ tests/components/QueueTokenCard.test.tsx
git commit -m "feat: QueueTokenCard (TDD), ProgressBar, StepHeader — Tailwind config tokens"
```

---

## Task 12: KioskShell + IPadBezel + App Skeleton

**Files:**
- Create: `src/kiosk/KioskShell.tsx`
- Create: `src/demo-shell/IPadBezel.tsx`
- Create: `src/kiosk/KioskApp.tsx` (skeleton)
- Create: `src/App.tsx` (skeleton)
- Create: `src/main.tsx`

- [ ] **Step 1: Create KioskShell**

```typescript
// src/kiosk/KioskShell.tsx
import { ReactNode } from 'react'
import { ProgressBar } from './components/ProgressBar'

interface KioskShellProps { children: ReactNode; progressPercent: number }

export function KioskShell({ children, progressPercent }: KioskShellProps) {
  return (
    <div className="flex flex-col h-full bg-bg-surface">
      <div className="flex justify-between items-center px-3.5 py-1.5 text-[10px] font-semibold
                      text-text-primary bg-bg-surface">
        <span>10:30</span><span>📶 🔋</span>
      </div>
      <header className="flex items-center justify-between px-4 py-2.5 bg-bg-card border-b border-border-light">
        <div className="flex items-center gap-1.5">
          <div className="w-7 h-7 rounded-[7px] bg-brand flex items-center justify-center">
            <svg width="14" height="14" viewBox="0 0 20 20" fill="none">
              <path d="M4 5l6 10 6-10" stroke="#fff" strokeWidth="2.5" strokeLinecap="round" strokeLinejoin="round"/>
            </svg>
          </div>
          <span className="font-['Bricolage_Grotesque'] font-bold text-[15px] text-brand">Visit</span>
        </div>
        <ProgressBar percent={progressPercent} />
      </header>
      <main className="flex-1 overflow-hidden relative">{children}</main>
    </div>
  )
}
```

- [ ] **Step 2: Create IPadBezel**

```typescript
// src/demo-shell/IPadBezel.tsx
import { ReactNode } from 'react'

export function IPadBezel({ children }: { children: ReactNode }) {
  return (
    <div className="[@media(display-mode:standalone)]:hidden flex justify-center">
      <div className="relative" style={{ width: 390 }}>
        <div className="bg-[#1c1c1e] rounded-[36px] px-3.5 pt-4 pb-5
                        shadow-[0_0_0_1px_#3a3a3c,0_30px_80px_rgba(0,0,0,0.7)] relative">
          <div className="absolute -left-1 top-20 w-1 h-7 bg-[#2c2c2e] rounded-l" />
          <div className="absolute -left-1 top-32 w-1 h-7 bg-[#2c2c2e] rounded-l" />
          <div className="absolute -right-1 top-28 w-1 h-12 bg-[#2c2c2e] rounded-r" />
          <div className="flex justify-center h-5 mb-1.5 items-center">
            <div className="w-2 h-2 rounded-full bg-[#2c2c2e] border border-[#444]" />
          </div>
          <div className="bg-bg-surface rounded-[20px] overflow-hidden w-full" style={{ height: 500 }}>
            {children}
          </div>
          <div className="flex justify-center pt-2.5 pb-1">
            <div className="w-24 h-1 bg-[#3a3a3c] rounded-full" />
          </div>
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Create skeleton KioskApp and App.tsx with providers**

```typescript
// src/kiosk/KioskApp.tsx (skeleton — replaced in Task 13)
export function KioskApp() {
  return <div className="p-6 text-brand font-bold">KioskApp skeleton — providers wired</div>
}
```

```typescript
// src/App.tsx
import { useState, useCallback } from 'react'
import { IPadBezel } from './demo-shell/IPadBezel'
import { SessionProvider } from './kiosk/session/SessionProvider'
import { KioskErrorBoundary } from './kiosk/errors/KioskErrorBoundary'
import { OfflineOverlay } from './kiosk/errors/OfflineOverlay'
import { useNetworkStatus } from './kiosk/errors/useNetworkStatus'
import { KioskApp } from './kiosk/KioskApp'

function AppInner() {
  const { isOnline } = useNetworkStatus()
  const [resetKey, setResetKey] = useState(0)

  const handleReset = useCallback(() => {
    setResetKey(k => k + 1)
  }, [])

  return (
    <SessionProvider onReset={handleReset}>
      <KioskErrorBoundary onReset={handleReset}>
        <KioskApp key={resetKey} />
      </KioskErrorBoundary>
      <OfflineOverlay visible={!isOnline} />
    </SessionProvider>
  )
}

export default function App() {
  return (
    <div className="min-h-screen bg-[#0f0f13] flex flex-col items-center justify-center p-8">
      <IPadBezel>
        <AppInner />
      </IPadBezel>
    </div>
  )
}
```

```typescript
// src/main.tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import App from './App'
import './index.css'

createRoot(document.getElementById('root')!).render(
  <StrictMode><App /></StrictMode>
)
```

- [ ] **Step 4: Verify iPad frame renders**

```bash
npm start
```
Expected: dark page, iPad frame visible, purple skeleton text, no console errors.

- [ ] **Step 5: Commit**

```bash
git add src/
git commit -m "feat: KioskShell, IPadBezel, App with SessionProvider + ErrorBoundary + OfflineOverlay"
```

---

## Task 13: S1_Welcome + KioskApp Flow Router

**Files:**
- Create: `src/kiosk/S1_Welcome.tsx`
- Modify: `src/kiosk/KioskApp.tsx`

- [ ] **Step 1: Create S1_Welcome.tsx**

```typescript
// src/kiosk/S1_Welcome.tsx
import type { PreBookedType } from '../types'
import { KioskShell } from './KioskShell'
import { PageLayout } from './layout/PageLayout'

export type HomeSelection =
  | { type: 'prebooked'; appointmentType: PreBookedType }
  | { type: 'walkin-doctor' }
  | { type: 'walkin-lab' }
  | { type: 'pharmacy' }

interface Props { onSelect: (selection: HomeSelection) => void }

const activeOptions = [
  { label: 'I have a pre-booked doctor appointment', icon: '📅', selection: { type: 'prebooked', appointmentType: 'doctor' } as HomeSelection },
  { label: 'I have a pre-booked lab test',           icon: '🧪', selection: { type: 'prebooked', appointmentType: 'lab'    } as HomeSelection },
]

const inactiveOptions = [
  'I am walking in for a doctor appointment',
  'I am walking in for a lab test',
  'I am here for a pharmacy pickup',
]

export function S1_Welcome({ onSelect }: Props) {
  return (
    <KioskShell progressPercent={10}>
      <PageLayout>
        <h1 className="font-['Bricolage_Grotesque'] font-bold text-xl text-text-primary leading-tight mb-1">
          Welcome to<br />Visit Health Clinic
        </h1>
        <p className="text-xs text-text-secondary mb-3">How would you like to check in today?</p>

        <div className="flex flex-col gap-2.5">
          {activeOptions.map(opt => (
            <button
              key={opt.label}
              onClick={() => onSelect(opt.selection)}
              style={{ minHeight: '64px' }}
              className="bg-bg-card rounded-[14px] px-4 py-3.5 flex items-center gap-3 text-left
                         border-2 border-transparent hover:border-brand transition-all
                         shadow-[0_2px_8px_rgba(113,79,255,0.06)]"
            >
              <div className="w-10 h-10 rounded-[10px] bg-border-light flex items-center justify-center text-lg flex-shrink-0">
                {opt.icon}
              </div>
              <span className="font-['Montserrat'] font-semibold text-sm text-text-primary">{opt.label}</span>
              <span className="ml-auto text-brand">›</span>
            </button>
          ))}

          {inactiveOptions.map(label => (
            <button
              key={label}
              disabled
              style={{ minHeight: '64px' }}
              className="bg-bg-card rounded-[14px] px-4 py-3.5 flex items-center gap-3 text-left
                         opacity-40 cursor-not-allowed"
            >
              <div className="w-10 h-10 rounded-[10px] bg-error-bg flex items-center justify-center text-lg flex-shrink-0">🚶</div>
              <span className="font-['Montserrat'] font-semibold text-sm text-text-primary">{label}</span>
              <span className="ml-auto text-text-secondary">›</span>
            </button>
          ))}
        </div>

        <p className="mt-auto text-center text-[10px] text-text-secondary py-2">
          Need help? Talk to our receptionist →
        </p>
      </PageLayout>
    </KioskShell>
  )
}
```

- [ ] **Step 2: Update KioskApp.tsx — flow router with session integration**

```typescript
// src/kiosk/KioskApp.tsx
import { useState } from 'react'
import { S1_Welcome, HomeSelection } from './S1_Welcome'
import { FlowAPreBooked } from './flows/A_PreBooked'
import { FlowBBeneficiary } from './flows/B_Beneficiary'
import { FlowCPolicyHolder } from './flows/C_PolicyHolder'
import { FlowDBeneficiaryWalkIn } from './flows/D_BeneficiaryWalkIn'
import { FlowENonPolicy } from './flows/E_NonPolicy'
import type { PreBookedType } from '../types'
import { useSession } from './session/SessionProvider'
import { useTelemetry } from './telemetry/useTelemetry'

type ActiveFlow = 'home' | 'A' | 'B' | 'C' | 'D' | 'E'

export function KioskApp() {
  const [activeFlow, setActiveFlow] = useState<ActiveFlow>('home')
  const [preBookedType, setPreBookedType] = useState<PreBookedType>('doctor')
  const { startSession, endSession } = useSession()
  const { trackEvent } = useTelemetry()

  const handleHomeSelection = (selection: HomeSelection) => {
    startSession()
    trackEvent('check_in_started', { metadata: { selectionType: selection.type } })

    if (selection.type === 'prebooked') {
      setPreBookedType(selection.appointmentType)
      setActiveFlow('A')
    } else if (selection.type === 'walkin-doctor') {
      setActiveFlow('C')
    } else if (selection.type === 'walkin-lab') {
      setActiveFlow('D')
    } else if (selection.type === 'pharmacy') {
      setActiveFlow('E')
    }
  }

  const handleReset = () => {
    endSession('completed')
    setActiveFlow('home')
  }

  if (activeFlow === 'home') return <S1_Welcome onSelect={handleHomeSelection} />
  if (activeFlow === 'A')   return <FlowAPreBooked preBookedType={preBookedType} onReset={handleReset} />
  if (activeFlow === 'B')   return <FlowBBeneficiary onReset={handleReset} />
  if (activeFlow === 'C')   return <FlowCPolicyHolder onReset={handleReset} />
  if (activeFlow === 'D')   return <FlowDBeneficiaryWalkIn onReset={handleReset} />
  return <FlowENonPolicy onReset={handleReset} />
}
```

- [ ] **Step 3: Verify routing — clicking pre-booked options should not crash**

```bash
npm start
```

- [ ] **Step 4: Commit**

```bash
git add src/kiosk/S1_Welcome.tsx src/kiosk/KioskApp.tsx
git commit -m "feat: S1_Welcome + KioskApp router with session + telemetry integration"
```

---

## Task 14: Flow A — State Machine + flowMeta

**Files:**
- Create: `src/kiosk/flows/A_PreBooked/index.tsx`
- Create: `src/kiosk/flows/A_PreBooked/flowMeta.ts`

- [ ] **Step 1: Create flowMeta.ts**

```typescript
// src/kiosk/flows/A_PreBooked/flowMeta.ts
import type { FlowMeta } from '../../../types'

export const flowMeta: FlowMeta = {
  welcome:   { name: 'S1 — Welcome', description: 'Shared entry point for all 5 flows.', openDecisions: [] },
  identify:  { name: 'S2 — Identify', description: 'Mobile entry or QR scan. Calls api.verifyPhone / api.scanQR.', openDecisions: ['QR requires Visit app — safe for day 1?'] },
  otp:       { name: 'S2b — OTP Verify', description: 'Enter pre-received OTP. Shows beneficiary phone from backend.', openDecisions: ['3 failed OTPs: permanently block or cooldown?'] },
  appointments: { name: 'S3 — Appointment List', description: 'Today\'s appointments. Global check-in via api.checkIn.', openDecisions: ['Late arrival >15 min: auto-walk-in or receptionist? (D-CI-2)'] },
  token:     { name: 'S4 — Queue Token', description: 'Token(s) issued. 30s auto-reset via session manager.', openDecisions: ['Token printing?'] },
  'error-not-found':   { name: '⚠ Not Found', description: 'No appointment today for this number.', openDecisions: [] },
  'error-otp-blocked': { name: '⚠ OTP Blocked', description: '3 wrong attempts. Receptionist only.', openDecisions: [] },
}

export const stepProgress: Record<string, number> = {
  welcome: 10, identify: 35, otp: 55, appointments: 75, token: 100,
  'error-not-found': 0, 'error-otp-blocked': 0,
}
```

- [ ] **Step 2: Create Flow A reducer + step router using FlowShell**

```typescript
// src/kiosk/flows/A_PreBooked/index.tsx
import { useReducer } from 'react'
import type { FlowAState, FlowAAction, PreBookedType } from '../../../types'
import { FlowShell } from '../../layout/FlowShell'
import { stepProgress } from './flowMeta'
import { S2_Identify } from './steps/S2_Identify'
import { S2b_OTPVerify } from './steps/S2b_OTPVerify'
import { S3_AppointmentList } from './steps/S3_AppointmentList'
import { S4_QueueToken } from './steps/S4_QueueToken'
import { ErrorNotFound } from './steps/ErrorNotFound'
import { ErrorOTPBlocked } from './steps/ErrorOTPBlocked'

interface Props {
  preBookedType: PreBookedType
  onReset: () => void
}

function makeInitialState(preBookedType: PreBookedType): FlowAState {
  return { step: 'identify', preBookedType, phone: '', beneficiaryPhone: '', patient: null, otpAttempts: 0, tokens: [] }
}

function reducer(state: FlowAState, action: FlowAAction): FlowAState {
  switch (action.type) {
    case 'PHONE_SUBMITTED':     return { ...state, phone: action.phone }
    case 'PATIENT_FOUND':       return { ...state, step: 'otp', patient: action.patient, beneficiaryPhone: action.beneficiaryPhone }
    case 'PATIENT_NOT_FOUND':   return { ...state, step: 'error-not-found' }
    case 'QR_SCANNED':          return { ...state, step: 'appointments', patient: action.patient }
    case 'OTP_CORRECT':         return { ...state, step: 'appointments', otpAttempts: 0 }
    case 'OTP_WRONG': {
      const attempts = state.otpAttempts + 1
      return attempts >= 3
        ? { ...state, step: 'error-otp-blocked', otpAttempts: attempts }
        : { ...state, otpAttempts: attempts }
    }
    case 'CHECKIN_CONFIRMED':   return { ...state, step: 'token', tokens: action.tokens }
    case 'LAB_UNAVAILABLE':     return { ...state, step: 'error-not-found' }
    case 'BACK':
      if (state.step === 'otp') return { ...state, step: 'identify' }
      return state
    case 'RESET':               return makeInitialState(state.preBookedType)
    default:                    return state
  }
}

export function FlowAPreBooked({ preBookedType, onReset }: Props) {
  const [state, dispatch] = useReducer(reducer, makeInitialState(preBookedType))

  const handleReset = () => { dispatch({ type: 'RESET' }); onReset() }

  const screens: Record<string, React.ReactNode> = {
    identify:            <S2_Identify state={state} dispatch={dispatch} />,
    otp:                 <S2b_OTPVerify state={state} dispatch={dispatch} />,
    appointments:        <S3_AppointmentList state={state} dispatch={dispatch} />,
    token:               <S4_QueueToken state={state} onReset={handleReset} />,
    'error-not-found':   <ErrorNotFound onRetry={handleReset} />,
    'error-otp-blocked': <ErrorOTPBlocked />,
  }

  return (
    <FlowShell stepKey={state.step} progressPercent={stepProgress[state.step]}>
      {screens[state.step]}
    </FlowShell>
  )
}
```

- [ ] **Step 3: Commit**

```bash
git add src/kiosk/flows/A_PreBooked/index.tsx src/kiosk/flows/A_PreBooked/flowMeta.ts
git commit -m "feat: Flow A useReducer state machine + flowMeta, using FlowShell"
```

---

## Task 15: Flow A — Screens (S2 through S4 + Error States)

All screens use `PageLayout`, `useApiCall`, and `useTelemetry`. No bare `await api.*` calls.

**Files:**
- Create: `src/kiosk/flows/A_PreBooked/steps/S2_Identify.tsx`
- Create: `src/kiosk/flows/A_PreBooked/steps/S2b_OTPVerify.tsx`
- Create: `src/kiosk/flows/A_PreBooked/steps/S3_AppointmentList.tsx`
- Create: `src/kiosk/flows/A_PreBooked/steps/S4_QueueToken.tsx`
- Create: `src/kiosk/flows/A_PreBooked/steps/ErrorNotFound.tsx`
- Create: `src/kiosk/flows/A_PreBooked/steps/ErrorOTPBlocked.tsx`

- [ ] **Step 1: S2_Identify.tsx**

```typescript
// src/kiosk/flows/A_PreBooked/steps/S2_Identify.tsx
import { useState } from 'react'
import type { FlowAState, FlowAAction, VerifyPhoneResult, QRScanResult } from '../../../../types'
import { NumericKeypad } from '../../../components/NumericKeypad'
import { PageLayout } from '../../../layout/PageLayout'
import { api } from '../../../services'
import { useApiCall } from '../../../errors/useApiCall'

interface Props { state: FlowAState; dispatch: React.Dispatch<FlowAAction> }

export function S2_Identify({ state, dispatch }: Props) {
  const [tab, setTab] = useState<'mobile' | 'qr'>('mobile')
  const [phone, setPhone] = useState('')
  const phoneCall = useApiCall<VerifyPhoneResult>()
  const qrCall = useApiCall<QRScanResult>()

  const handleKey = (key: string) => {
    if (key === 'del') setPhone(p => p.slice(0, -1))
    else if (key !== '*' && phone.length < 10) setPhone(p => p + key)
  }

  const handleFindAppointment = async () => {
    const result = await phoneCall.execute(() => api.verifyPhone(phone))
    if (!result) return
    if (result.found) {
      dispatch({ type: 'PATIENT_FOUND', patient: result.patient, beneficiaryPhone: result.beneficiaryPhone })
    } else {
      dispatch({ type: 'PATIENT_NOT_FOUND' })
    }
  }

  const handleSimulateQR = async () => {
    const result = await qrCall.execute(() => api.scanQR('mock-qr-data'))
    if (!result) return
    if (result.found) {
      dispatch({ type: 'QR_SCANNED', patient: result.patient })
    }
  }

  return (
    <PageLayout>
      <h2 className="font-['Bricolage_Grotesque'] font-bold text-lg text-text-primary mb-1">
        Find your appointment
      </h2>
      <p className="text-[11px] text-text-secondary mb-3">Enter your mobile or scan your Visit QR code</p>

      <div className="flex bg-border-light rounded-[10px] p-0.5 mb-3">
        {(['mobile', 'qr'] as const).map(t => (
          <button
            key={t}
            onClick={() => setTab(t)}
            style={{ minHeight: '36px' }}
            className={`flex-1 text-[11px] font-semibold rounded-[8px] transition-all
              ${tab === t ? 'bg-brand text-white shadow-[0_2px_6px_rgba(113,79,255,0.3)]' : 'text-brand'}`}
          >
            {t === 'mobile' ? '📱 Mobile Number' : '🔲 Scan QR Code'}
          </button>
        ))}
      </div>

      {tab === 'mobile' ? (
        <>
          <div className="bg-bg-card rounded-[12px] px-3.5 py-3 mb-3 border-2 border-brand flex items-center gap-2">
            <span>🇮🇳</span>
            <span className="text-sm font-semibold text-text-secondary border-r border-border-light pr-2 mr-0.5">+91</span>
            <span className="text-base font-semibold text-text-primary flex-1 tracking-wide">
              {phone || <span className="text-[#bbb]">Enter mobile number</span>}
            </span>
          </div>
          {phoneCall.state.error && (
            <p className="text-[11px] text-accent font-semibold mb-2">
              {phoneCall.state.error.retryable ? 'Connection issue — please try again' : 'Something went wrong'}
            </p>
          )}
          <NumericKeypad
            onKey={handleKey}
            actionLabel={phoneCall.state.loading ? 'Finding...' : phone.length === 10 ? 'Find my appointment →' : undefined}
            onAction={phone.length === 10 && !phoneCall.state.loading ? handleFindAppointment : undefined}
          />
        </>
      ) : (
        <div className="flex-1 flex flex-col">
          <div className="flex-1 bg-[#111] rounded-[16px] relative overflow-hidden flex items-center justify-center mb-3">
            <div className="text-center">
              <p className="text-xs text-[#666] mb-3">Camera active — align QR code within frame</p>
              <button
                onClick={handleSimulateQR}
                disabled={qrCall.state.loading}
                className="px-4 py-2 bg-brand text-white text-xs font-semibold rounded-[8px]"
              >
                {qrCall.state.loading ? 'Scanning...' : 'Simulate QR Scan ✓'}
              </button>
            </div>
          </div>
          <div className="bg-bg-card rounded-[12px] p-3 flex gap-2.5 items-start mb-2">
            <span className="text-lg">📱</span>
            <div>
              <p className="text-xs font-semibold text-text-primary">Open your Visit app</p>
              <p className="text-[11px] text-text-secondary">Go to My Appointments → show QR code here</p>
            </div>
          </div>
          <button onClick={() => setTab('mobile')} className="text-center text-[11px] text-brand font-semibold py-2">
            Use mobile number instead →
          </button>
        </div>
      )}
    </PageLayout>
  )
}
```

- [ ] **Step 2: S2b_OTPVerify.tsx**

```typescript
// src/kiosk/flows/A_PreBooked/steps/S2b_OTPVerify.tsx
import { useState, useEffect, useCallback } from 'react'
import type { FlowAState, FlowAAction, OTPResult } from '../../../../types'
import { OTPInput } from '../../../components/OTPInput'
import { NumericKeypad } from '../../../components/NumericKeypad'
import { PageLayout } from '../../../layout/PageLayout'
import { api } from '../../../services'
import { useApiCall } from '../../../errors/useApiCall'

interface Props { state: FlowAState; dispatch: React.Dispatch<FlowAAction> }

export function S2b_OTPVerify({ state, dispatch }: Props) {
  const [otp, setOtp] = useState('')
  const [resendCooldown, setResendCooldown] = useState(0)
  const otpCall = useApiCall<OTPResult>()

  const handleKey = (key: string) => {
    if (key === 'del') setOtp(o => o.slice(0, -1))
    else if (key !== '*' && otp.length < 6) setOtp(o => o + key)
  }

  const handleVerify = async () => {
    const result = await otpCall.execute(() => api.verifyOTP(state.beneficiaryPhone, otp))
    if (!result) return
    if (result.valid) dispatch({ type: 'OTP_CORRECT' })
    else { dispatch({ type: 'OTP_WRONG' }); setOtp('') }
  }

  const handleResend = useCallback(async () => {
    if (resendCooldown > 0) return
    await api.resendOTP(state.beneficiaryPhone)
    setResendCooldown(120)
  }, [resendCooldown, state.beneficiaryPhone])

  useEffect(() => {
    if (resendCooldown <= 0) return
    const timer = setInterval(() => setResendCooldown(c => c - 1), 1000)
    return () => clearInterval(timer)
  }, [resendCooldown])

  const maskedPhone = `+91 XXXXX ${state.beneficiaryPhone.slice(-5)}`

  return (
    <PageLayout>
      <button onClick={() => dispatch({ type: 'BACK' })}
        className="flex items-center gap-1 text-brand text-xs font-medium mb-3.5">
        ‹ Back
      </button>
      <h2 className="font-['Bricolage_Grotesque'] font-bold text-lg text-text-primary mb-1">
        Enter verification code
      </h2>
      <p className="text-[11px] text-text-secondary mb-4 leading-relaxed">
        We sent a code to <strong>{maskedPhone}</strong> — the number registered for this appointment
      </p>

      <OTPInput value={otp} onChange={setOtp} />

      {state.otpAttempts > 0 && (
        <p className="text-center text-[11px] text-accent mt-2 font-semibold">
          Incorrect — {3 - state.otpAttempts} attempt{3 - state.otpAttempts !== 1 ? 's' : ''} remaining
        </p>
      )}

      {otpCall.state.error && (
        <p className="text-center text-[11px] text-accent mt-2 font-semibold">
          Connection issue — please try again
        </p>
      )}

      <p className="text-center text-[11px] text-text-secondary mt-2 mb-3">
        Didn't receive it?{' '}
        {resendCooldown > 0 ? (
          <span className="text-brand font-semibold">Resend in {Math.floor(resendCooldown / 60)}:{String(resendCooldown % 60).padStart(2, '0')}</span>
        ) : (
          <button onClick={handleResend} className="text-brand font-semibold underline">Resend OTP</button>
        )}
      </p>

      <NumericKeypad
        onKey={handleKey}
        actionLabel={otpCall.state.loading ? 'Verifying...' : otp.length === 6 ? 'Verify →' : undefined}
        onAction={otp.length === 6 && !otpCall.state.loading ? handleVerify : undefined}
      />
    </PageLayout>
  )
}
```

- [ ] **Step 3: S3_AppointmentList.tsx**

> **Two-phase loading pattern:**
> - Phase 1 (immediate): Appointment cards render as soon as `state.patient` is set. Doctor name, time, test names are all from Visit's GetPatientOrders (already in state). No spinner on the main content.
> - Phase 2 (async, non-blocking): On mount, `getAppointmentCabinDetails` fires to TP. Cabin fields show an animated skeleton while loading. If TP is down, they fall back to "See receptionist". "Check In Now" is **always enabled** once Phase 1 is ready — cabin info is wayfinding, not a check-in gate.

```typescript
// src/kiosk/flows/A_PreBooked/steps/S3_AppointmentList.tsx
import { useEffect, useState } from 'react'
import type {
  FlowAState, FlowAAction, CheckInResult,
  AppointmentWithDetails, CabinDetailMap
} from '../../../../types'
import { Card } from '../../../components/Card'
import { Badge } from '../../../components/Badge'
import { Button } from '../../../components/Button'
import { PageLayout } from '../../../layout/PageLayout'
import { api } from '../../../services'
import { useApiCall } from '../../../errors/useApiCall'
import { colors } from '../../../../tokens/theme'

interface Props { state: FlowAState; dispatch: React.Dispatch<FlowAAction> }

export function S3_AppointmentList({ state, dispatch }: Props) {
  const [labUnavailable, setLabUnavailable] = useState(false)

  // Phase 1: Appointments already in state from verifyPhone/OTP flow.
  // Initialise each with cabinDataStatus:'loading' — TP call fires on mount below.
  const [appointments, setAppointments] = useState<AppointmentWithDetails[]>(() =>
    state.patient!.appointments.map(a => ({ ...a, cabinDataStatus: 'loading' as const }))
  )

  const checkInCall = useApiCall<CheckInResult>()
  const cabinCall   = useApiCall<CabinDetailMap>()
  const patient     = state.patient!

  // Phase 2: Fetch cabin details from TP in parallel — fires once, non-blocking.
  // "Check In Now" does NOT wait for this. Cabin info is wayfinding only.
  useEffect(() => {
    const apptIds = patient.appointments.map(a => a.id)
    cabinCall.execute(() => api.getAppointmentCabinDetails(apptIds)).then(cabinMap => {
      setAppointments(prev => prev.map(a => {
        if (!cabinMap) return { ...a, cabinDataStatus: 'unavailable' as const }
        const details = cabinMap[a.id]
        return details
          ? { ...a, ...details, cabinDataStatus: 'loaded' as const }
          : { ...a, cabinDataStatus: 'unavailable' as const }
      }))
    })
  }, []) // eslint-disable-line react-hooks/exhaustive-deps

  const handleCheckIn = async () => {
    const appointmentIds = patient.appointments.map(a => a.id)
    const result = await checkInCall.execute(() => api.checkIn(patient.id, appointmentIds))
    if (!result) return
    if (result.success) {
      dispatch({ type: 'CHECKIN_CONFIRMED', tokens: result.tokens })
    } else if (result.reason === 'lab_unavailable') {
      setLabUnavailable(true)
    } else {
      dispatch({ type: 'PATIENT_NOT_FOUND' })
    }
  }

  return (
    <PageLayout>
      <p className="font-['Bricolage_Grotesque'] font-bold text-base text-text-primary mb-0.5">
        Welcome, {patient.fullName}! 👋
      </p>
      <p className="text-[11px] text-text-secondary mb-3">Here are your appointments for today</p>
      <p className="text-[10px] font-semibold uppercase tracking-wider text-brand mb-2">Today's Schedule</p>

      {appointments.map(appt => (
        <Card key={appt.id} accentColor={appt.type === 'consultation' ? colors.brand : colors.accent} className="mb-2">
          <div className="flex items-center gap-1.5 mb-1.5">
            <Badge variant={appt.type === 'consultation' ? 'consult' : 'lab'}>
              {appt.type === 'consultation' ? 'Consultation' : 'Lab Test'}
            </Badge>
          </div>
          <p className="font-['Montserrat'] font-semibold text-[13px] text-text-primary mb-1">
            {appt.type === 'consultation' ? `${appt.doctorName} — ${appt.specialty}` : appt.testNames?.join(', ')}
          </p>
          <div className="flex gap-2.5 text-[11px] text-text-secondary">
            <span>🕐 {appt.time}</span>
            {/* Cabin field — Phase 2 data from TP. Shows skeleton while loading. */}
            <span className="flex items-center gap-1">
              📍 {
                appt.cabinDataStatus === 'loading'
                  ? <span className="inline-block w-20 h-3 bg-border-light rounded animate-pulse align-middle" />
                  : appt.cabinDataStatus === 'unavailable'
                  ? 'See receptionist'
                  : `${appt.cabin}${appt.floor ? `, ${appt.floor}` : ''}`
              }
            </span>
          </div>
          {appt.prepNote && (
            <p className="text-[10px] text-accent font-semibold mt-1.5">⚠ {appt.prepNote}</p>
          )}
          {/* Routing instruction — only shown once TP data has loaded */}
          {appt.cabinDataStatus === 'loaded' && appt.waitingAreaInstructions && (
            <p className="text-[10px] text-text-secondary mt-1.5 leading-snug">
              {appt.waitingAreaInstructions}
            </p>
          )}
        </Card>
      ))}

      {labUnavailable && (
        <div className="bg-error-bg rounded-[10px] p-3 mb-2 text-[11px] text-accent font-semibold">
          ⚠ Lab services are currently unavailable. Please speak to our receptionist.
        </div>
      )}

      {checkInCall.state.error && (
        <div className="bg-error-bg rounded-[10px] p-3 mb-2 text-[11px] text-accent font-semibold">
          Connection issue — please try again
        </div>
      )}

      {/* Check In Now is ALWAYS enabled once appointments load — cabin data does NOT gate it */}
      <Button variant="primary" fullWidth onClick={handleCheckIn} className="mt-2"
        disabled={checkInCall.state.loading || labUnavailable}>
        {checkInCall.state.loading ? 'Checking in...' : 'Check In Now →'}
      </Button>
      <button onClick={() => dispatch({ type: 'RESET' })}
        className="text-center text-[11px] text-text-secondary underline mt-2 py-1">
        This isn't me
      </button>
    </PageLayout>
  )
}
```

- [ ] **Step 4: S4_QueueToken.tsx (uses session manager for idle, not local setTimeout)**

```typescript
// src/kiosk/flows/A_PreBooked/steps/S4_QueueToken.tsx
import { useEffect } from 'react'
import type { FlowAState } from '../../../../types'
import { QueueTokenCard } from '../../../components/QueueTokenCard'
import { Button } from '../../../components/Button'
import { PageLayout } from '../../../layout/PageLayout'
import { useTelemetry } from '../../../telemetry/useTelemetry'

interface Props { state: FlowAState; onReset: () => void }

export function S4_QueueToken({ state, onReset }: Props) {
  const { trackEvent } = useTelemetry()

  useEffect(() => {
    trackEvent('check_in_completed', { metadata: { flowId: 'A', tokenCount: state.tokens.length } })
  }, []) // eslint-disable-line react-hooks/exhaustive-deps

  // Note: idle timeout (30s) is handled by the SessionProvider via idleConfig.ts
  // No local setTimeout needed here.

  return (
    <PageLayout scrollable verticalCenter={false}>
      <div className="flex flex-col items-center">
        <div className="w-14 h-14 rounded-full bg-success-bg flex items-center justify-center text-2xl mt-2 mb-2">✅</div>
        <h2 className="font-['Bricolage_Grotesque'] font-bold text-lg text-text-primary text-center mb-1">
          You're checked in, {state.patient?.fullName.split(' ')[0]}!
        </h2>
        <p className="text-[11px] text-text-secondary text-center mb-3">
          Your token{state.tokens.length > 1 ? 's are' : ' is'} below
        </p>

        {state.tokens.map(token => (
          <div key={token.tokenNumber} className="w-full mb-2">
            <QueueTokenCard token={token} />
          </div>
        ))}

        <div className="w-full bg-success-bg rounded-[10px] p-3 flex items-center gap-2 mb-3">
          <span className="text-lg">💬</span>
          <p className="text-[11px] text-success-text leading-snug">
            <strong>WhatsApp sent!</strong> Live queue updates on +91 XXXXX {state.patient?.phone.slice(-5)}
          </p>
        </div>

        <Button variant="secondary" fullWidth onClick={onReset}>Done — Return to Start</Button>
      </div>
    </PageLayout>
  )
}
```

- [ ] **Step 5: Error screens**

```typescript
// src/kiosk/flows/A_PreBooked/steps/ErrorNotFound.tsx
import { Button } from '../../../components/Button'
import { PageLayout } from '../../../layout/PageLayout'

export function ErrorNotFound({ onRetry }: { onRetry: () => void }) {
  return (
    <PageLayout verticalCenter>
      <div className="w-14 h-14 rounded-full bg-error-bg flex items-center justify-center text-2xl mb-3">⚠️</div>
      <h2 className="font-['Bricolage_Grotesque'] font-bold text-lg text-text-primary mb-2">
        We couldn't find your appointment
      </h2>
      <p className="text-xs text-text-secondary leading-relaxed mb-6 text-center">
        No appointment was found for this number today. Check your number or speak to our receptionist.
      </p>
      <Button variant="primary" fullWidth onClick={onRetry} className="mb-2">Try Again</Button>
      <Button variant="secondary" fullWidth>Talk to Receptionist</Button>
    </PageLayout>
  )
}
```

```typescript
// src/kiosk/flows/A_PreBooked/steps/ErrorOTPBlocked.tsx
import { Button } from '../../../components/Button'
import { PageLayout } from '../../../layout/PageLayout'

export function ErrorOTPBlocked() {
  return (
    <PageLayout verticalCenter>
      <div className="w-14 h-14 rounded-full bg-error-bg flex items-center justify-center text-2xl mb-3">🔒</div>
      <h2 className="font-['Bricolage_Grotesque'] font-bold text-lg text-text-primary mb-2">
        Too many incorrect attempts
      </h2>
      <p className="text-xs text-text-secondary leading-relaxed mb-6 text-center">
        For your security, we've blocked further attempts. Please speak to our receptionist.
      </p>
      <Button variant="primary" fullWidth>Talk to Receptionist</Button>
    </PageLayout>
  )
}
```

- [ ] **Step 6: Verify full Flow A in browser**

```bash
npm start
```
Test: Welcome → Pre-booked doctor → `9876543210` → Find → `123456` → Verify → Appointments → Check In → Token → Done.

- [ ] **Step 7: Commit**

```bash
git add src/kiosk/flows/A_PreBooked/steps/
git commit -m "feat: all Flow A screens — using PageLayout, useApiCall, useTelemetry"
```

---

## Task 16: Flow A Integration Tests

**Files:**
- Create: `tests/flows/A_PreBooked/flow.test.tsx`

- [ ] **Step 1: Write integration tests including error/timeout scenarios**

```typescript
// tests/flows/A_PreBooked/flow.test.tsx
import { render, screen } from '@testing-library/react'
import { FlowAPreBooked } from '../../../src/kiosk/flows/A_PreBooked'
import { SessionProvider } from '../../../src/kiosk/session/SessionProvider'

// Mock api service
jest.mock('../../../src/kiosk/services', () => ({
  api: {
    verifyPhone: jest.fn(),
    verifyOTP: jest.fn(),
    scanQR: jest.fn(),
    checkIn: jest.fn(),
    resendOTP: jest.fn(),
  }
}))

// Mock session — provide minimal context
function TestWrapper({ children }: { children: React.ReactNode }) {
  return (
    <SessionProvider onReset={() => {}}>
      {children}
    </SessionProvider>
  )
}

const mockPatient = {
  id: 'UHID-001', fullName: 'Test Patient', phone: '9876543210',
  appointments: [{
    id: 'appt-1', type: 'consultation' as const, patientName: 'Test Patient',
    patientId: 'UHID-001', doctorName: 'Dr. Test', specialty: 'General',
    time: '10:00 AM', location: 'Cabin 1'
  }]
}

test('renders identify screen on mount', () => {
  render(<FlowAPreBooked preBookedType="doctor" onReset={() => {}} />, { wrapper: TestWrapper })
  expect(screen.getByText(/Find your appointment/)).toBeInTheDocument()
})

test('shows mobile and QR tabs', () => {
  render(<FlowAPreBooked preBookedType="doctor" onReset={() => {}} />, { wrapper: TestWrapper })
  expect(screen.getByText(/Mobile Number/)).toBeInTheDocument()
  expect(screen.getByText(/Scan QR Code/)).toBeInTheDocument()
})
```

- [ ] **Step 2: Run tests**

```bash
npx react-scripts test --watchAll=false tests/flows/
```

- [ ] **Step 3: Run ALL tests**

```bash
npx react-scripts test --watchAll=false
```
Expected: all tests pass.

- [ ] **Step 4: Commit**

```bash
git add tests/flows/
git commit -m "test: Flow A integration tests with session provider context"
```

---

## Task 17: Stub Flows B–E + Demo Shell

**Files:**
- Create: `src/kiosk/flows/B_Beneficiary/index.tsx` (and C, D, E)
- Create: `src/demo-shell/ArchetypeTabs.tsx`
- Create: `src/demo-shell/InfoPanel.tsx`
- Create: `src/demo-shell/DemoShell.tsx`
- Modify: `src/App.tsx`

- [ ] **Step 1: Create stub flows B–E**

Each uses FlowShell + PageLayout:
```typescript
// src/kiosk/flows/B_Beneficiary/index.tsx
import { FlowShell } from '../../layout/FlowShell'
import { PageLayout } from '../../layout/PageLayout'

export function FlowBBeneficiary({ onReset }: { onReset: () => void }) {
  return (
    <FlowShell stepKey="stub" progressPercent={0}>
      <PageLayout verticalCenter>
        <p className="text-4xl mb-3">🔜</p>
        <p className="font-['Bricolage_Grotesque'] font-bold text-lg text-text-primary mb-1">Coming Soon</p>
        <p className="text-sm text-text-secondary">Flow B · Beneficiary (Policy Holder Pre-Booked)</p>
        <button onClick={onReset} className="mt-6 text-xs text-brand underline">← Back to Start</button>
      </PageLayout>
    </FlowShell>
  )
}
```
Repeat for `FlowCPolicyHolder`, `FlowDBeneficiaryWalkIn`, `FlowENonPolicy` with appropriate labels.

- [ ] **Step 2: Create ArchetypeTabs**

```typescript
// src/demo-shell/ArchetypeTabs.tsx
export type ArchetypeId = 'A' | 'B' | 'C' | 'D' | 'E'

const tabs = [
  { id: 'A' as ArchetypeId, label: 'A · Pre-Booked',    color: '#714FFF' },
  { id: 'B' as ArchetypeId, label: 'B · Beneficiary',   color: '#FF9273' },
  { id: 'C' as ArchetypeId, label: 'C · Policy W/I',    color: '#22c55e' },
  { id: 'D' as ArchetypeId, label: 'D · Ben W/I',       color: '#f59e0b' },
  { id: 'E' as ArchetypeId, label: 'E · Non-Policy',    color: '#06b6d4' },
]

interface Props { active: ArchetypeId; onChange: (id: ArchetypeId) => void }

export function ArchetypeTabs({ active, onChange }: Props) {
  return (
    <div className="flex gap-2 mb-6 flex-wrap justify-center">
      {tabs.map(tab => (
        <button key={tab.id} onClick={() => onChange(tab.id)}
          className={`px-3.5 py-1.5 rounded-full text-xs font-semibold border transition-all
            ${active === tab.id ? 'text-white border-transparent' : 'text-[#666] border-[#333] bg-transparent'}`}
          style={active === tab.id ? { background: tab.color, borderColor: tab.color } : undefined}>
          {tab.label}
        </button>
      ))}
    </div>
  )
}
```

- [ ] **Step 3: Create InfoPanel**

```typescript
// src/demo-shell/InfoPanel.tsx
import type { StepMeta } from '../types'

export function InfoPanel({ meta }: { meta?: StepMeta }) {
  if (!meta) return null
  return (
    <div className="mt-5 w-[390px] bg-[#1a1a1f] rounded-[16px] p-4 border border-[#2a2a30]">
      <p className="font-['Montserrat'] font-semibold text-sm text-white mb-1">{meta.name}</p>
      <p className="text-xs text-[#888] leading-relaxed mb-2.5">{meta.description}</p>
      {meta.openDecisions.map((d, i) => (
        <div key={i} className="flex items-start gap-1.5 text-[11px] text-[#f0a060] leading-snug mb-1">
          <span className="text-[10px] mt-0.5 flex-shrink-0">⚠</span><span>{d}</span>
        </div>
      ))}
    </div>
  )
}
```

- [ ] **Step 4: Update App.tsx with full demo shell**

```typescript
// src/App.tsx
import { useState, useCallback } from 'react'
import { IPadBezel } from './demo-shell/IPadBezel'
import { ArchetypeTabs, ArchetypeId } from './demo-shell/ArchetypeTabs'
import { InfoPanel } from './demo-shell/InfoPanel'
import { SessionProvider } from './kiosk/session/SessionProvider'
import { KioskErrorBoundary } from './kiosk/errors/KioskErrorBoundary'
import { OfflineOverlay } from './kiosk/errors/OfflineOverlay'
import { useNetworkStatus } from './kiosk/errors/useNetworkStatus'
import { KioskApp } from './kiosk/KioskApp'
import { flowMeta } from './kiosk/flows/A_PreBooked/flowMeta'

function AppInner({ onStepChange }: { onStepChange: (step: string) => void }) {
  const { isOnline } = useNetworkStatus()
  const [resetKey, setResetKey] = useState(0)

  const handleReset = useCallback(() => {
    setResetKey(k => k + 1)
    onStepChange('welcome')
  }, [onStepChange])

  return (
    <SessionProvider onReset={handleReset}>
      <KioskErrorBoundary onReset={handleReset}>
        <KioskApp key={resetKey} />
      </KioskErrorBoundary>
      <OfflineOverlay visible={!isOnline} />
    </SessionProvider>
  )
}

export default function App() {
  const [activeTab, setActiveTab] = useState<ArchetypeId>('A')
  const [currentStep, setCurrentStep] = useState('welcome')

  return (
    <div className="min-h-screen bg-[#0f0f13] flex flex-col items-center py-8 px-4">
      <p className="text-[#888] text-[11px] tracking-widest uppercase mb-2">Kiosk Check-In Prototype</p>
      <ArchetypeTabs
        active={activeTab}
        onChange={id => { setActiveTab(id); setCurrentStep('welcome') }}
      />
      <IPadBezel>
        <AppInner key={activeTab} onStepChange={setCurrentStep} />
      </IPadBezel>
      <InfoPanel meta={flowMeta[currentStep]} />
    </div>
  )
}
```

- [ ] **Step 5: Commit**

```bash
git add src/demo-shell/ src/kiosk/flows/B_Beneficiary/ src/kiosk/flows/C_PolicyHolder/ \
        src/kiosk/flows/D_BeneficiaryWalkIn/ src/kiosk/flows/E_NonPolicy/ src/App.tsx
git commit -m "feat: demo shell (tabs + info panel), stub flows B–E"
```

---

## Task 18: PWA + Build Verification + Audit

- [ ] **Step 1: Run all tests**

```bash
npx react-scripts test --watchAll=false
```
Expected: all pass.

- [ ] **Step 2: TypeScript check**

```bash
npx tsc --noEmit
```
Expected: no errors.

- [ ] **Step 3: Register service worker for PWA**

CRA has built-in PWA support. In `src/index.tsx`, register the service worker:

```typescript
// src/index.tsx — add at the bottom:
import * as serviceWorkerRegistration from './serviceWorkerRegistration'
serviceWorkerRegistration.register()
```

CRA's `--template typescript` includes `serviceWorkerRegistration.ts` in `src/`. If it was not generated, create a minimal one or use `npx react-app-rewired` — but CRA templates typically include it.

- [ ] **Step 4: Production build**

```bash
npm run build
```
Expected: `build/` created, no errors.

- [ ] **Step 5: Preview production build**

```bash
npx serve -s build
```
Open `http://localhost:3000` (or `http://localhost:5000` depending on serve defaults) — verify iPad frame, Flow A full navigation.

- [ ] **Step 6: Audit — no inline hex in kiosk/ components**

```bash
grep -rn '#714FFF\|#FF9273\|#686670\|#1F1E1E\|#F5F3FF\|#ede9ff' src/kiosk/ --include='*.tsx' | grep -v 'theme.ts' | grep -v '.test.'
```
Expected: zero matches (all colors should use Tailwind config utilities or `theme.ts` imports).

- [ ] **Step 7: Final commit**

```bash
git add . && git commit -m "feat: Flow A kiosk prototype complete — production-grade, build verified"
```

---

## Verification Checklist

- [ ] `npm start` → iPad frame visible, Visit logo, 5-option home screen
- [ ] Pre-booked doctor → S2 Identify (mobile + QR tabs)
- [ ] `9876543210` → Priya Sharma found → OTP screen shows beneficiary phone
- [ ] `9123456780` → Rajan Pillai found (consult only)
- [ ] Any other phone → Not Found error
- [ ] OTP `123456` → correct → Appointments
- [ ] 3 wrong OTPs → OTP Blocked screen
- [ ] QR tab → "Simulate QR Scan" → skips OTP → Appointments
- [ ] Appointment cards with correct colour coding (purple=consult, coral=lab)
- [ ] "Check In Now" → two tokens (A-047, L-012)
- [ ] S4 auto-resets via session manager after 30s idle
- [ ] "Still there?" overlay appears 15s before reset on S2/S3 (120s idle)
- [ ] No "Still there?" on S1 Welcome (no timeout)
- [ ] "This isn't me" → full reset to home
- [ ] Tabs B–E show "Coming Soon" placeholder
- [ ] Info panel updates per step
- [ ] OTP resend countdown works (2 min cooldown, live countdown)
- [ ] Network offline in DevTools → OfflineOverlay appears
- [ ] `npx react-scripts test --watchAll=false` → all tests pass
- [ ] `npm run build` → no TypeScript errors, `build/` produced
- [ ] `REACT_APP_USE_REAL_API=true npm run build` → builds cleanly
- [ ] Console shows telemetry events on each screen transition
- [ ] No inline hex codes in `kiosk/` components (audit grep)

---

## API Handoff Reference

Hand backend developers `src/kiosk/services/api.ts` (interface) and `src/kiosk/services/api.real.ts` (TODO stubs). Each TODO names the exact endpoint, HTTP method, request body, and response shape. The kiosk UI requires zero changes — set `REACT_APP_USE_REAL_API=true` and point `REACT_APP_API_BASE_URL` at the real server. All requests include `X-Device-Id`, `X-Clinic-Id`, and `X-Session-Id` headers automatically.
