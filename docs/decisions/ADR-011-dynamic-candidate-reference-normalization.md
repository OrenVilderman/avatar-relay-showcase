# ADR 011: Dynamic Candidate Reference Normalization

- **Status:** Accepted
- **Date:** 2026-10-14
- **Related decisions:** ADR 003, ADR 006, ADR 008

## Context

The original portfolio prototype was centered on one candidate and used candidate-specific references in intent recovery. This made the behavior harder to configure and reuse. Hebrew also permits short clitic prefixes to attach directly to a name, creating variants that need to be recognized without broadly rewriting unrelated text.

The API regression suite also needed broader coverage for multi-turn context, Hebrew and mixed-language requests, prompt-injection boundaries, fallback policy, and input validation.

## Decision

Represent candidate names and aliases through configuration and resolve them through a dedicated candidate-reference normalization boundary. The implementation supports Hebrew clitic prefixes ו/ה/ב/כ/ל/מ/ש when attached to configured candidate references. Examples include "לאורן", "מהאורן", and "שאורן".

Keep normalization candidate-aware and deterministic. Tests should cover both expected variants and false positives. Preserve the established intent categories and deterministic canonical-response layer.

The private implementation's reported Postman/Newman collection contains 38 requests and 55 assertions. It covers multi-turn chat, candidate-reference variants, Hebrew/mixed-language edge cases, security and fallback guardrails, and request boundaries.

## Consequences

**Benefits**

- Candidate references can be changed through configuration rather than intent-recovery code.
- Hebrew clitic-attached forms can be normalized before intent recovery.
- Broader E2E coverage helps protect multi-turn, bilingual, and security-sensitive behavior.

**Trade-offs and boundaries**

- Overly broad prefix stripping can change unrelated words; candidate-aware matching and negative tests are required.
- Configuration-backed references do not by themselves provide full multi-tenant isolation. Session separation, authorization, storage, and tenant-specific data need separate decisions and verification.
- The showcase describes the engineering decision but does not contain the private implementation or runnable pipeline.

## Verification

The implementation maintainer reports a green local pipeline with 148 JUnit 5 backend tests, 122 Vitest frontend tests, and 38 Newman requests / 55 assertions. These are maintainer-reported results, not independently executed by this public documentation change. For current pass/fail status, consult actual test output or CI for the implementation commit being reviewed.

## Guardrails

1. Keep candidate names and aliases out of hardcoded intent-recovery branches.
2. Test supported Hebrew clitics and false-positive normalization cases.
3. Treat all visitor-provided content as untrusted.
4. Do not describe the normalization boundary as complete multi-tenant support.
5. Keep executable tests and private credentials in the implementation repository.
