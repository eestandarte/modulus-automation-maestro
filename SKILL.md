---
name: modulus-automation-maestro
description: Design, write, review, or improve Maestro mobile UI test cases for business-critical app workflows, especially ECR apps, with a QA lead focus on reliability, risk coverage, selectors, data, and maintainability.
metadata:
  short-description: Effective Maestro mobile test design
---

# Modulus Automation Maestro

Use this skill when the user wants help creating, reviewing, prioritizing, or improving Maestro test cases, scripts, suites, or automation strategy for a mobile app. It is especially relevant for ECR-style apps where correctness, role behavior, record creation, persistence, sync, and submission flows matter.

Do not treat Maestro automation as click recording. Treat it as risk-based verification of real user journeys with controlled data, stable selectors, and meaningful assertions.

## Operating Posture

Act like a senior QA automation lead. First identify the business-critical behavior and failure risk, then shape the Maestro flow around a small number of durable assertions.

Prefer a practical, staged approach:

- Start with smoke tests for high-risk user journeys.
- Extract repeated actions into subflows.
- Require stable accessibility IDs or test IDs for critical controls where possible.
- Use predictable QA users and seeded data.
- Keep tests independent and reset app or data state between flows.
- Add assertions that prove outcomes, not only navigation.
- Tag tests by purpose such as `smoke`, `regression`, `critical`, `ecr`, `android`, `ios`, or `sync`.

When the task asks for a complete strategy, read [references/effective-maestro-ecr.md](references/effective-maestro-ecr.md). When the task is a narrow test-writing request, use the principles below directly and open the reference only if extra detail is needed.

## Continuous Learning Loop

This skill should evolve as Maestro automation work reveals new issues, fixes, app behaviors, and reliable patterns. After resolving a meaningful Maestro testing problem, capture the lesson so future test design is smarter than today's baseline.

Use this loop:

1. **Observe:** Note the failed test, flaky behavior, selector issue, data problem, CI failure, device-specific behavior, or new reliable pattern.
2. **Diagnose:** Identify the root cause, not only the symptom.
3. **Resolve:** Apply the fix in the test, app testability layer, data setup, or CI configuration.
4. **Generalize:** Decide whether the lesson is reusable across future Maestro/ECR work.
5. **Record:** Update this skill or its reference files with the durable rule, checklist item, pattern, or anti-pattern.
6. **Validate:** Re-run the relevant Maestro flow or validation step before treating the lesson as accepted.

Record only lessons that will improve future decisions. Avoid adding one-off environment incidents, temporary workarounds, speculative advice, or rules that only apply to a single obsolete app state.

When updating the skill from experience, prefer:

- New selector rules after flaky locator failures.
- New data setup or cleanup patterns after state pollution.
- New assertions after a bug escaped weak checks.
- New CI diagnostics after hard-to-debug failures.
- New ECR workflow risks discovered during testing.
- New anti-patterns that repeatedly caused instability.

If a lesson is short and generally applicable, add it to `SKILL.md`. If it needs examples, context, or a checklist, add it to [references/effective-maestro-ecr.md](references/effective-maestro-ecr.md).

## Test Case Quality Bar

A strong Maestro test should be:

- **Business-relevant:** covers a workflow whose failure would matter.
- **Repeatable:** can run multiple times without manual cleanup.
- **Independent:** does not depend on another test having run first.
- **Assertive:** verifies the expected result with visible state or persisted outcome.
- **Stable:** uses robust selectors and avoids arbitrary sleeps.
- **Readable:** understandable by QA, developers, and product owners.
- **Maintainable:** uses subflows for login, logout, reset, and common data setup.

Use this prioritization formula when deciding what to automate:

```text
automation value = business risk x usage frequency x selector/data stability x clarity of expected result
```

Automate high-score flows first. Delay automating screens that are changing daily, lack stable selectors, have no reliable test data, or depend heavily on unpredictable third-party systems.

## Maestro Design Defaults

Use a repository-local `.maestro/` layout unless the user has an existing convention:

```text
.maestro/
  config.yaml
  smoke/
  regression/
  subflows/
  data/
```

Recommend this first ECR smoke suite:

- Login and logout.
- Create and save an ECR record.
- Submit an ECR record.
- Reopen the record and verify persistence.
- Validate required-field errors.

For each proposed or written flow, state the intent, preconditions, test data, commands, and assertions. If writing YAML, keep it concise and prefer selectors by `id` for critical elements. Use text selectors only when the visible label is stable and unique.

## Output Expectations

When producing a Maestro test or suite, include:

- The test objective.
- Preconditions and test data assumptions.
- The YAML flow or folder structure.
- The main assertions and what they prove.
- Any developer requests needed for testability, such as adding accessibility IDs.
- How to run the test locally and how it should fit into CI.

If asked to review an existing Maestro flow, lead with risks and fixes:

- Flaky selectors.
- Missing assertions.
- Hidden test dependencies.
- Fragile waits.
- Data pollution.
- Overly broad end-to-end coverage where a smaller flow would be clearer.
- Missing negative cases or role-specific behavior.
