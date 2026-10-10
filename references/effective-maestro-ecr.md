# Effective Maestro Testing for ECR Mobile Apps

## Core Strategy

For an ECR mobile app, prioritize tests around record integrity, role permissions, persistence, submission, and sync. A beautiful automation suite that does not protect these workflows is low value.

Start with a small, reliable suite:

1. Login with a known QA user.
2. Create a new ECR record with minimum valid fields.
3. Save as draft and verify confirmation.
4. Reopen the draft and verify persisted values.
5. Submit the record and verify submitted state.
6. Attempt submission with required fields missing and verify validation.
7. Verify role-specific access for at least one restricted action.

Only expand after these flows run reliably on local devices and in CI.

## What the Team Needs

Before writing serious Maestro flows, align with developers and product owners on:

- QA/staging environment URL or build target.
- App ID or bundle ID.
- Dedicated test users by role.
- Seeded records that can be safely edited.
- Data cleanup or reset mechanism.
- Accessibility IDs or test IDs for important controls.
- Expected messages and state transitions.
- Offline, sync, and timeout behavior.

If these are missing, write the test with explicit assumptions and list the blockers. Do not hide uncertainty inside brittle selectors or arbitrary waits.

## Recommended Folder Layout

```text
.maestro/
  config.yaml
  smoke/
    login.yaml
    create-ecr-record.yaml
    submit-ecr-record.yaml
  regression/
    edit-record.yaml
    validation-errors.yaml
    role-access.yaml
    offline-sync.yaml
  subflows/
    login.yaml
    logout.yaml
    reset-app.yaml
    create-basic-record.yaml
  data/
    users.yaml
```

Use subflows for repeated setup and navigation. Keep actual test files focused on the behavior being proven.

## Selector Guidance

Prefer selectors in this order:

1. Stable accessibility ID or test ID.
2. Stable, unique visible text.
3. Relative selectors only when necessary.
4. Coordinates only as a last resort.

Ask developers for IDs on:

- Login fields and buttons.
- Main navigation tabs.
- New record button.
- Patient or case search.
- Required ECR fields.
- Save draft button.
- Submit button.
- Confirmation banners.
- Error messages.
- Record status labels.

Good selector example:

```yaml
- tapOn:
    id: "new_record_button"
```

Acceptable selector example when text is stable:

```yaml
- assertVisible: "Record submitted"
```

Avoid using generic text like `Save`, `OK`, or `Continue` when the screen can contain multiple matches.

## Good ECR Flow Pattern

```yaml
appId: com.company.ecr
name: Create ECR record successfully
tags:
  - smoke
  - ecr
env:
  PATIENT_NAME: "QA Test Patient"
---
- launchApp
- runFlow: ../subflows/login.yaml
- tapOn:
    id: "new_record_button"
- tapOn:
    id: "patient_name_input"
- inputText: ${PATIENT_NAME}
- tapOn:
    id: "chief_complaint_input"
- inputText: "Fever and headache"
- tapOn:
    id: "save_draft_button"
- assertVisible: "Draft saved"
- tapOn:
    id: "submit_record_button"
- assertVisible: "Record submitted"
```

This is only a starter. Strengthen it by reopening the submitted record and asserting the record status and key entered values.

## Assertion Guidance

Every test should prove one main thing and possibly a few supporting facts.

Good assertions:

- Confirmation message appears after save.
- Submitted status appears after submission.
- Required-field error appears when a field is empty.
- Record appears in search results.
- Edited value persists after app restart.
- Restricted action is hidden or blocked for a lower-permission role.

Weak assertions:

- Only asserting that a page title is visible after many actions.
- Only checking that a button exists.
- Checking implementation details the user would never observe.

## Stability Rules

Avoid fixed sleeps except as a temporary diagnostic. Prefer waiting for stable UI state with visible assertions or Maestro wait behavior.

Reset state before flows using one of:

- A reset subflow.
- Logout plus login.
- App clear state, when safe.
- Backend fixture reset.
- Unique generated test data.

Keep flows short. A single giant flow covering login, onboarding, create, edit, upload, submit, logout, and sync will be slow to debug. Split into focused flows and share setup through subflows.

## Knowledge Learning Path

Improve the Maestro suite and this skill through a repeatable learning path whenever a real issue is found and resolved.

Use this template for durable lessons:

```text
Date:
Context:
Failed or risky behavior:
Root cause:
Fix applied:
Reusable lesson:
Where applied:
Validation:
```

Examples of lessons worth recording:

- A selector became flaky because multiple controls shared the same visible text, so the app now needs unique accessibility IDs for repeated actions.
- A record submission test passed even when data was not persisted, so future submission flows must reopen the record and assert key saved values.
- CI failures were hard to diagnose, so failed runs must retain screenshots, device details, app build version, and environment name.
- A test polluted staging data, so flows creating ECR records must use unique identifiers or backend cleanup.
- Offline sync behavior differed by platform, so sync tests need platform tags and platform-specific assertions where behavior intentionally differs.

Do not record:

- A temporary network outage.
- A workaround for a screen that is already being replaced.
- A personal preference with no reliability impact.
- A raw error message without root-cause analysis.

Use accumulated lessons to refine future Maestro work in this order:

1. Improve selectors and app testability.
2. Improve state setup and cleanup.
3. Improve assertions and escaped-defect coverage.
4. Improve CI visibility and failure triage.
5. Improve suite organization and tags.
6. Expand coverage only after reliability improves.

## Accepted ECR Automation Lessons

### 2026-10-08 — A successful text tap does not prove the control acted

- **Context:** Cash-with-change automation on the Axis ECR Cash Payment dialog.
- **Observed failure:** Maestro reported a successful tap on a regex matching
  `100`, but the tender display remained at `0` and the `Change` result never
  appeared.
- **Root cause:** The dialog exposed more than one identical `100` label, and
  matching visible text was insufficient to identify a unique actionable
  control. A completed `tapOn` command only proved that Maestro found and
  touched a matching semantic node; it did not prove that business state
  changed.
- **Resolution pattern:** Prefer a stable accessibility ID for each quick-
  amount control. When IDs are unavailable, use an unambiguous input path such
  as the numeric keypad and immediately wait for a business-state postcondition
  such as the tendered amount, computed change, or enabled confirmation state.
- **Reusable rule:** After interacting with repeated labels or generic text,
  assert the resulting state before continuing. Never infer success solely
  from a green Maestro step.
- **Applied in:** `complete-cash-with-change.yaml`, replacing the duplicated
  quick-amount text selector with keypad entry for `1`, `0`, `0` and a wait for
  the `Change` state.
- **Validation:** The original failure and unchanged `0` tender were confirmed
  by the device screenshot and Maestro trace. A later continuous execution
  advanced beyond the cash-with-change and card-scheme cases into Phase 2,
  confirming the keypad remediation on the device.

The same failure mode was later observed on a unique `Cash` label while the
device was offline: Maestro logged the tap as successful, but the payment-
method dialog remained open. For controls whose text node is not reliably
clickable, use a guarded fallback against the unchanged parent state. The
offline cash flow first taps `Cash`, then taps the calibrated button center
only if `Select payment method:` is still visible, and finally waits for
`Cash Payment`. Guarding on the parent state prevents the fallback from firing
after a successful first tap.

### 2026-10-08 — Keep business scenarios visible above reusable subflows

- **Context:** Exact cash, cash-with-change, and five manual-terminal card
  schemes were initially composed only through a root journey and generic
  helper flows.
- **Risk:** Hiding distinct business scenarios inside one large flow makes
  coverage difficult to inventory, rerun, report, and diagnose. One failure
  can obscure every later scenario.
- **Resolution pattern:** Give each business scenario its own numbered flow in
  the appropriate `smoke/` or `regression/` group, while extracting repeated
  mechanics into `subflows/`. A root orchestrator may still compose the
  end-to-end journey, but it is not a replacement for named scenario files.
- **Reusable rule:** Subflows represent reusable actions; numbered test files
  represent independently reportable business behavior.
- **Applied in:** Cash-with-change and VISA, MASTERCARD, AMEX, JCB, and DISCOVER
  were exposed as numbered smoke scenarios. Resilience checks were assigned to
  smoke, while safe retry, manager-authorized void, and fiscal receipt checks
  were assigned to regression.
- **Validation:** The resulting smoke/regression inventory and all added YAML
  documents parsed successfully. Device calibration remains required for the
  newly added Phase 2 scenarios.

### 2026-10-08 — Payment completion needs outcome-specific assertions

- **Context:** Manual-terminal card payments finish on a Payment Complete
  receipt view before the cashier taps `Done`.
- **Risk:** Asserting only that the completion dialog opened can pass even when
  the wrong payment type or card scheme was recorded.
- **Resolution pattern:** Before dismissing the result, assert the visible
  payment method and scheme, such as `Paid via Card (JCB)` and `Card: JCB`.
  For cash overpayment, assert tendered amount and computed change. Only then
  tap `Done` and verify the completion view closes.
- **Reusable rule:** A payment test must prove the recorded tender attributes,
  not merely successful navigation.
- **Validation:** The JCB completion view supplied during calibration visibly
  exposed both the payment method and card scheme, providing stable observable
  outcomes for scheme-specific assertions.

### 2026-10-08 — Continuous flows require explicit state handoff contracts

- **Context:** A numbered ECR journey continues across cash and card payments,
  app-restart recovery, offline sale capture, and reconnect synchronization.
- **Observed risk:** A later test relaunched the app and expected table
  selection even though the preceding test had stopped inside the active table
  after `Done`. That mismatch makes a nominally continuous suite fail before it
  reaches the behavior under test.
- **Resolution pattern:** For every chained test, document an `ENTRY STATE` and
  `EXIT STATE` covering the visible screen, selected table/session, order
  contents, authentication state, network state, and any queued data needed by
  the next test. Start from the predecessor's actual exit state instead of
  repeating generic login or navigation setup.
- **Reusable rule:** A continuous suite is a sequence of explicit state
  contracts. Test N+1 must consume the exact state produced by Test N. If a
  scenario intentionally requires a fresh process, logout, clean state, or
  device restart, perform and explain that reset inside the scenario rather
  than assuming it happened between files.
- **Applied in:** Test 12 now records that it ends inside TABLE1 with no order;
  Test 13 begins from that view, performs its required stop/relaunch internally,
  and Tests 14–16 document the online → offline → queued sale → online handoff.
- **Validation:** The continuous root suite now invokes Tests 13–16 directly
  after Test 12, all changed YAML documents parse successfully, and device
  execution confirmed the Test 12 → 13 → 14 state progression. Later Phase 2
  handoffs remain subject to their validation boundary below.

### 2026-10-08 — Assert controls only after their business-state gate is met

- **Context:** An app-restart recovery test added an item and immediately tried
  to assert the payable `TOTAL` state before saving the order and completing
  the kitchen-receipt step.
- **Observed failure:** The item was visibly present in Order 1, but Maestro
  could not find the expected payable-total state because the order was still
  a draft and the `Save` action had not yet committed it.
- **Root cause:** The test treated a displayed amount as equivalent to a
  business-enabled payment state. In this ECR workflow, `Save` commits/sends
  the order and the kitchen-receipt dialog must be handled before billing is
  available.
- **Resolution pattern:** Model the domain transition explicitly: add item →
  assert draft/save state → Save → handle kitchen receipt → assert TOTAL/payable
  state. After restarting, verify the saved unpaid order and proceed directly
  to payment without saving or printing it a second time.
- **Reusable rule:** Assertions must correspond to the current business state,
  not merely to visually rendered text. Never assert or operate a downstream
  control until the action that unlocks that state has completed.
- **Applied in:** Test 13 now saves the Iced Tea order, cancels the kitchen-
  receipt dialog, verifies TOTAL, restarts the app, verifies the saved unpaid
  order, and settles it without a duplicate Save.
- **Validation:** The invalid pre-Save TOTAL assertion was reproduced in
  Maestro Studio and confirmed by the supplied screenshot. A later device run
  completed Test 13 successfully, confirming the corrected Save → kitchen
  receipt → TOTAL → restart sequence.

### 2026-10-08 — Visual proximity does not imply one semantic text node

- **Context:** After restarting during Test 17, the recovered order visibly
  showed a single TOTAL button containing `TOTAL` and `₱85.00`.
- **Observed failure:** `assertVisible: 'TOTAL.*85\.00'` failed even though both
  strings were clearly visible on the same control.
- **Root cause:** Compose exposed the label and amount as separate accessibility
  semantics nodes. Maestro text regexes match an individual node; they do not
  concatenate neighboring descendants merely because they render inside one
  button or row.
- **Resolution pattern:** Assert each semantic value independently (`TOTAL`
  and `.*85\.00.*`), or request a stable parent content description/test ID
  that expresses the combined business value.
- **Reusable rule:** Build selectors from the accessibility hierarchy, not
  visual layout assumptions. When two values are visually adjacent, confirm
  whether Maestro sees one node or multiple nodes before combining them into a
  single regex.
- **Applied in:** Tests 13 and 17 now verify the TOTAL label and ₱85.00 amount
  separately.
- **Validation:** The recovered order and both visual values were confirmed in
  the device screenshot while the combined selector failed. Updated YAML
  parsing passed; Test 17 remains pending rerun beyond this corrected step.

### 2026-10-08 — Diagnose the first failed step and clean polluted state

- **Context:** A rerun of the safe-retry regression appeared to have failed to
  tap Cash, but the trace showed the Cash command had never executed.
- **Observed failure:** The first red step was the controlled-total assertion.
  A prior aborted run had left an Iced Tea order attached to TABLE1, so the new
  setup added another item and produced ₱170 rather than ₱85. Every Cash step
  below the failed assertion was skipped.
- **Root cause:** The test assumed a clean table but did not reset or reconcile
  business data left by a failed run. Interpreting the final screen instead of
  the first failed command misidentified the failure as a tap problem.
- **Resolution pattern:** Read the execution trace from the first failed step,
  then inspect the visible business state. Before a controlled-value scenario,
  add a safe cleanup/reset precondition that detects and settles stale QA data.
  Extract already-calibrated interactions, such as guarded Cash selection,
  into a shared subflow rather than copying fixes between tests.
- **Reusable rule:** A repeatable ECR regression must recover from its own
  previously interrupted state. Never assume TABLE1, a session, an outbox, or
  transaction history is clean merely because the app relaunched.
- **Applied in:** Test 17 now runs `ensure-table1-empty.yaml` before creating its
  ₱85 order, and both Test 15 and Test 17 use `select-cash-payment.yaml`.
- **Validation:** The screenshot confirmed the stale order, duplicated item
  quantity, ₱170 total, and the earlier failed assertion. Updated cleanup and
  normalization YAML validates successfully; device rerun remains pending.

## 2026-10-08 Axis ECR Session Knowledge Backup

This section consolidates the reusable decisions learned while calibrating the
continuous Axis ECR payment and resilience suites. Use it as the starting point
for future Axis flows rather than rediscovering the same UI and state behavior.

### Confirmed application workflow contracts

- After `Payment Complete` → `Done`, the app returns inside the active table
  with no order. It does not automatically return to table selection.
- The top-left Back control returns from the active table to table selection.
- A newly selected item remains a draft until `Save` is tapped. The Save action
  enters the kitchen-receipt path; handling that dialog is required before the
  order becomes payable through `TOTAL`.
- A saved, unpaid order can be recovered after an app stop/relaunch. After
  recovery, do not Save it a second time; proceed to `TOTAL` and payment.
- Manual-terminal success ends on a Payment Complete receipt view. Validate
  `Paid via Card (<scheme>)` and `Card: <scheme>` before tapping `Done`.
- Cash overpayment must prove both the tendered amount and computed change, not
  only that payment completed.

### Confirmed Maestro/UI behavior

- A green `tapOn` step proves only that Maestro touched a matching semantics
  node. It does not prove the surrounding Compose control executed.
- Repeated quick-amount labels can resolve to the wrong or non-actionable node.
  Numeric keypad entry is more deterministic when stable IDs are absent.
- Even a unique label such as `Cash` may be exposed as a non-actionable child
  of its clickable button. Verify the destination state and use a guarded
  fallback only while the unchanged parent dialog remains visible.
- Text rendered in one visual component can be split across accessibility
  nodes. Assert `TOTAL` and `₱85.00` separately instead of assuming
  `TOTAL.*85.00` can match a combined node.
- Prefer postconditions that prove business state: `Cash Payment`, `Change`,
  `Payment Complete`, receipt number, recorded card scheme, table availability,
  or synchronization state.

### Continuous-suite rules

- Every chained case must declare its `ENTRY STATE` and `EXIT STATE`, including
  screen/view, table/session, order contents, authentication, connectivity, and
  queued data needed by the next case.
- Test N+1 consumes the actual state produced by Test N. Do not add generic
  launch/login/navigation steps that contradict the predecessor's exit state.
- If restart, logout, network loss, or clean-state initialization is the
  behavior under test, perform it explicitly inside that case and document why.
- Keep each business scenario as a numbered test file for reporting and triage;
  keep repeated mechanics in subflows. A root journey may chain the scenarios
  but must not hide their individual identities.
- Put fast, critical operability checks in `smoke/`. Put deeper idempotency,
  authorization, transaction-history, and fiscal comparisons in `regression/`.

### Repeatability and failure-triage rules

- Diagnose from the first red Maestro command, not from the final screen or a
  downstream step that never ran.
- Failed ECR runs can leave occupied tables, saved orders, payment dialogs,
  offline state, or queued outbox data. Relaunching the app does not guarantee
  clean business data.
- Controlled-value tests must normalize the entry screen and reconcile stale QA
  orders before creating new test data. Otherwise an expected ₱85 order may
  become ₱170 or more and invalidate every downstream assertion.
- Shared recovery patterns currently used by Axis are:
  `normalize-to-table-selection.yaml`, `ensure-table1-empty.yaml`, and
  `select-cash-payment.yaml`.
- A cleanup sale is acceptable for QA recovery only when it is explicit and
  the test does not treat transaction history as pristine afterward. Tests
  asserting exact transaction counts need backend fixtures, unique identifiers,
  or an authoritative cleanup API instead.

### 2026-10-10 — Maestro `scroll` does not accept a `direction` property

- **Context:** Z-Reading test (24) used `- scroll: direction: DOWN` to page
  through a long report view.
- **Observed failure:** Maestro Studio showed "Unknown Property: direction File:
  …/24-RDG-BIR-04-z-reading-fields.yaml Line: 122 Column: 1 The property
  'direction' is not recognized."
- **Root cause:** The `scroll` command in Maestro is a bare action with no
  properties — it always scrolls down. The `direction` property belongs to
  `swipe`, not `scroll`.
- **Resolution pattern:** Use plain `- scroll` (scrolls down by default). To
  scroll up, use `- swipe: direction: DOWN` (swipe direction is the finger
  gesture, opposite of visual scroll). To scroll to a specific element, use
  `- scrollUntilVisible`.
- **Reusable rule:** Never pass properties to `- scroll`. For directional
  scrolling, use `- swipe` with `direction: UP` (to scroll content down) or
  `direction: DOWN` (to scroll content up).
- **Applied in:** `24-RDG-BIR-04-z-reading-fields.yaml` — replaced three
  `scroll: direction: DOWN` blocks with plain `- scroll`.
- **Validation:** Error was reproduced in Maestro Studio; fix confirmed by
  removing the invalid property.

### 2026-10-10 — Maestro Studio caches files and can overwrite disk edits

- **Context:** Tests 17-41 were edited on disk via CLI while Maestro Studio
  (Electron app, v0.9.7) was open or had been open earlier.
- **Observed failure:** All regression test files (17-41) appeared deleted from
  the `regression/` directory. Files were found in a renamed directory instead.
- **Root cause:** Maestro Studio caches the `.maestro/` directory tree in memory.
  When Studio writes back its cached state, it can overwrite or replace files
  that were created or modified on disk outside Studio. The Studio process may
  also hold the gRPC device connection, preventing headless `maestro test` from
  running (DeviceServerDiedException / gRPC UNAVAILABLE).
- **Resolution pattern:** Always close Maestro Studio before editing test files
  on disk or running headless `maestro test`. Kill all related processes:
  `pkill -f "maestro-studio"` and `pkill -f "studio-server.jar"`.
- **Reusable rule:** Maestro Studio and CLI/disk edits are mutually exclusive.
  Close Studio before switching to CLI workflows. If files go missing after a
  Studio session, check whether Studio overwrote the directory with its cache.
- **Validation:** Files were recovered from the renamed directory. Headless
  tests ran successfully after killing all Studio processes.

### Current validation boundary

- Exact cash, cash with change, manual-terminal card schemes, Test 13 restart
  recovery, and Test 14 offline-warning behavior have progressed through device
  execution during this calibration session.
- Test 15's guarded Cash-button fallback is YAML-valid but still requires a
  successful device rerun.
- Test 16 still requires device confirmation of reconnect timing and a stable
  visible `LOCAL_ONLY` → `UPLOADED`/`Synced` outcome.
- Test 17's normalization and stale-order cleanup are YAML-valid but require a
  device rerun. Its final exactly-once proof still needs a stable transaction or
  receipt identifier in the Transactions UI or a backend assertion.
- Tests 18–20 remain calibration-stage regression flows. Their manager-void,
  operator identity, fiscal serial comparison, and DUPLICATE-marker selectors
  must be confirmed against the current app build and receipt surface.

Do not promote a pending item above to device-validated based only on YAML
parsing or a green navigation step. Update this boundary only after observing
the required business outcome on the device or authoritative backend.

## CI Recommendations

Run smoke tests on every build:

```bash
maestro test .maestro/smoke
```

Run regression before release:

```bash
maestro test .maestro/regression
```

Use tags once the suite grows:

```bash
maestro test .maestro --include-tags smoke
```

Keep CI failures actionable. Store screenshots, logs, app build version, device model, OS version, and environment name with each failed run.

## Review Checklist

Use this checklist when reviewing a Maestro test:

- Does the test cover a real business risk?
- Can it run alone?
- Does it start from a known state?
- Does it use stable selectors?
- Does it avoid unnecessary sleeps?
- Does it have a meaningful assertion?
- Does it clean up or isolate its data?
- Is the test name specific?
- Are tags useful?
- Would a developer understand the failure within five minutes?

If the answer is no, improve the test before adding more coverage.
