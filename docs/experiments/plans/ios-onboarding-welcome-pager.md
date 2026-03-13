---
summary: "Plan: add a first-run welcome pager before iOS onboarding so QR scanning never jumps in without context"
read_when:
  - Designing or implementing the iOS onboarding first-run experience
  - Changing QR scan entry points in the iOS app
  - Aligning mobile onboarding with the macOS onboarding tone
owner: "openclaw"
status: "draft"
last_updated: "2026-03-13"
title: "iOS Onboarding Welcome Pager Plan"
---

# iOS Onboarding Welcome Pager Plan

## Context

The iOS app already has a nominal welcome step in `OnboardingWizardView`, but first-run behavior
does not actually land there. On a fresh install, `initializeState()` immediately opens the QR
scanner sheet when no saved gateway or credentials exist. That means the first thing a new user
sees is a camera scanner jumping on screen with no framing, no trust explanation, and no clear
reason for what they are expected to scan.

Current code path:

- `apps/ios/Sources/Onboarding/OnboardingWizardView.swift`
  - `welcomeStep` exists and already renders a basic welcome UI.
  - `initializeState()` bypasses that step on first launch by setting `showQRScanner = true`.

Reference product direction already exists elsewhere:

- macOS onboarding starts with a dedicated welcome page and an explicit security notice before
  moving into setup.
- Android onboarding also keeps gateway scanning behind an explicit welcome-first flow.

This plan adds a dedicated first-run welcome pager ahead of the QR-driven setup flow so the first
launch experience feels intentional and stable.

## Goals

- Never auto-open the QR scanner on first launch.
- Introduce a first-run welcome pager that explains what OpenClaw is doing on this device.
- Mirror the macOS onboarding tone: welcome framing first, trust/security notice second, setup
  action third.
- Keep repeat onboarding flows fast by limiting the pager to true first-run or explicit reset
  cases.
- Avoid introducing extra camera or permission prompts before the user chooses to continue.

## Non-goals

- Redesign the entire iOS onboarding wizard.
- Change gateway pairing mechanics, QR payloads, or auth flows.
- Add a multi-page carousel.
- Reorder the later mode/connect/auth/success steps beyond what is needed to insert the first-run
  pager cleanly.

## Existing Code Map

- `apps/ios/Sources/Onboarding/OnboardingWizardView.swift`
  - Owns the iOS onboarding step enum, welcome screen, QR scanner sheet, and first-run init logic.
- `apps/ios/Sources/Onboarding/QRScannerView.swift`
  - Owns the actual camera-backed QR scanning sheet.
- `apps/ios/Sources/Onboarding/OnboardingStateStore.swift`
  - Decides whether onboarding should be shown on launch.
- `apps/macos/Sources/OpenClaw/OnboardingView+Pages.swift`
  - Provides the product reference for the desired welcome/security framing.
- `apps/android/app/src/main/java/ai/openclaw/app/ui/OnboardingFlow.kt`
  - Provides the mobile reference that already avoids scanner-first presentation.

## Problem Statement

Today the first-run sequence is effectively:

1. app opens
2. onboarding full-screen cover appears
3. QR scanner sheet immediately stacks on top
4. camera view animates in before the user understands what OpenClaw is, what gateway they are
   connecting to, or why camera access is being requested

That creates three product problems:

- The transition feels broken because the first visible state is a sheet jump, not a stable screen.
- The trust model is missing at the moment users are asked to point a camera at a pairing artifact.
- The onboarding flow appears camera-first even though camera scanning is only one entry path.

## Product Decision

Add a dedicated first-run welcome pager before the existing onboarding setup flow.

This pager is a single full-screen page shown only when the user is truly new or has explicitly
reset onboarding. It does not launch the QR scanner. It explains:

- what OpenClaw is on iPhone
- what will happen next
- that the connected agent can use device capabilities depending on granted permissions

After the user taps the primary CTA, the app enters the existing onboarding flow at the gateway
entry screen, where QR scanning remains available as an explicit action.

## Proposed Flow

### First-run launch

1. App launches.
2. If onboarding should present and the first-run pager has not been seen, show the welcome pager.
3. User taps `Continue`.
4. App enters onboarding setup on the existing `welcome` step or a renamed gateway entry step.
5. User explicitly chooses:
   - `Scan QR Code`
   - `Set Up Manually`
6. QR scanner opens only after the explicit tap.

### Returning user with incomplete onboarding

- If the user already dismissed the first-run pager but has not completed onboarding, reopen the
  onboarding setup flow directly without re-showing the pager.

### Reset onboarding

- Resetting onboarding should also clear the pager-seen flag so the user can re-experience the full
  first-run flow.

## Pager Content Spec

### Layout

Use a calm full-screen layout, not a sheet stacked over another screen.

- Top: branded illustration or large symbol-based hero
- Middle: title, supporting copy
- Below: one trust/security card
- Bottom: primary CTA and optional close/skip affordance consistent with `allowSkip`

The page should visually feel closer to the macOS onboarding welcome page than to a utility form.

### Copy

Recommended baseline copy:

- Title: `Welcome to OpenClaw`
- Body: `Turn this iPhone into a secure OpenClaw node for chat, voice, camera, and device tools.`
- Security card title: `Security notice`
- Security card body:
  `The connected OpenClaw agent can use device capabilities you enable, such as camera, microphone,
photos, contacts, calendar, and location. Continue only if you trust the gateway and agent you
connect to.`
- Primary CTA: `Continue`
- Secondary CTA when skip is allowed: `Close`

### Supporting bullets

Under the body or inside a secondary section, show three short bullets:

- `Connect to your gateway`
- `Choose device permissions`
- `Use OpenClaw from your phone`

These should be explanatory, not interactive.

## Visual Direction

Use the existing iOS native visual language, but align with the macOS content hierarchy:

- Large centered welcome title.
- Quiet secondary body copy.
- One bordered warning card with an orange emphasis icon.
- Clear single primary CTA.

Do not make the pager look like a settings form or an empty placeholder screen.

## Motion And Presentation Rules

- No QR scanner should present during `onAppear` of the first-run pager.
- No chained modal animation on first launch.
- Transition from pager -> setup flow should be a normal in-flow state change, not a second modal
  presentation.
- Transition from setup flow -> QR scanner remains a sheet, but only after user intent.

## State Model

Add a lightweight persisted flag for the pager, for example:

- `onboarding.first_run_intro_seen`

Behavior:

- `false` on fresh install
- set to `true` when the user taps `Continue` on the pager
- cleared when onboarding is reset

Recommended presentation rules:

- If onboarding should present on launch and `first_run_intro_seen == false`, show the pager.
- If onboarding should present on launch and `first_run_intro_seen == true`, show the setup flow.
- If onboarding is already complete, show neither.

## iOS Implementation Direction

### 1. Stop scanner auto-presentation

Remove the current first-run auto-open path in `initializeState()`:

- do not set `showQRScanner = true` purely because there is no saved gateway/token/password

Keep the status text update if useful, but do not launch camera UI from init.

### 2. Add a real first-run intro state

Use one of these two approaches:

- Preferred: add a new pre-wizard `WelcomePagerView` shown before `OnboardingWizardView`
- Acceptable: add a new `intro` step before the current `welcome` step inside
  `OnboardingWizardView`

Preferred rationale:

- It cleanly separates product framing from operational setup.
- It avoids overloading the current `welcome` step, which is already functioning as the gateway
  action screen.

### 3. Keep current gateway entry behavior explicit

The existing action screen should remain the point where scanning starts:

- `Scan QR Code`
- `Set Up Manually`

That screen can keep its current mechanics, but copy can be tightened after the pager lands.

### 4. Reset flow

When onboarding is reset from settings, also clear the intro-seen flag so QA and users can verify
the first-run behavior end to end.

## Open Questions

- Should the post-pager setup screen still be titled `Welcome`, or should it be renamed to
  `Connect Gateway` now that the real welcome happens earlier?
- Should the pager include a static product illustration, or is an SF Symbol hero sufficient for
  the first implementation?
- If `allowSkip == false`, should the pager omit any close affordance entirely, or keep a passive
  dismiss gesture only?

Recommended defaults:

- Rename the current operational `welcome` step to `Connect Gateway`.
- Use SF Symbols first; add custom artwork only if design wants more personality later.
- Omit close affordance when `allowSkip == false`.

## Acceptance Criteria

- Fresh install first launch shows a stable welcome pager before any scanner UI.
- Fresh install first launch does not request camera usage until the user explicitly taps `Scan QR
Code`.
- The pager includes a visible security/trust notice modeled after macOS onboarding.
- Tapping `Continue` moves into onboarding setup without modal stacking or UI jumpiness.
- Reset onboarding reproduces the welcome pager.
- Returning to incomplete onboarding after already passing the pager does not show the pager again.

## Validation Plan

- Manual test on a fresh install:
  - launch app
  - verify welcome pager is first visible screen
  - verify no camera scanner appears automatically
- Manual test after tapping `Continue`:
  - verify onboarding setup appears
  - verify QR scanner appears only after explicit tap
- Manual test after reset onboarding:
  - verify pager appears again
- Regression test:
  - verify existing setup code scan, manual host entry, auth retry, and success flows still work

## Reference Notes

- macOS reference: welcome headline plus security notice before setup decisions
- Android reference: gateway scanning remains an explicit action inside onboarding, not an
  automatic first-launch modal
