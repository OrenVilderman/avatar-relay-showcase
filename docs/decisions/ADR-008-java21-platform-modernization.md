# ADR 008 — Java 21 Platform Modernization

> **Status:** Active  
> **Scope:** Backend runtime and engineering conventions

## Context

AvatarRelay is a recruiter-facing AI digital twin with a voice-driven flow.

The browser performs Speech-to-Text, then the backend makes blocking calls to external AI services. That makes I/O concurrency important, while keeping the current single-service architecture is preferable to adding infrastructure without a measured need.

The project also accumulated some straightforward constructor boilerplate as the backend grew.

## Decision

Modernize the backend from Java 17 to **Java 21** with **Spring Boot 4.1.1**.

Enable Virtual Threads through:

`spring.threads.virtual.enabled=true`

Use Virtual Threads for the existing blocking I/O model rather than introducing a more complex reactive architecture.

Add **Lombok** selectively, primarily for dependency-constructor boilerplate where it improves readability without hiding meaningful logic.

## Why

The goal is not "Java 21 because it is newer."

The goal is to make the existing I/O-heavy workload more concurrency-efficient per instance while keeping the architecture simple.

Virtual Threads also let the application keep its straightforward blocking HTTP integrations with external AI providers.

Lombok is intentionally limited. Existing records stay records; annotations are not added merely for consistency.

## Trade-offs

### What this improves

- More efficient handling of many concurrent blocking waits.
- A cleaner concurrency model for the current voice/AI workload.
- Less constructor boilerplate in dependency-only components.
- Access to Java 21 APIs and language/runtime improvements.

### What this does not solve

- Virtual Threads do not provide unlimited scale.
- They do not replace horizontal scaling.
- They do not make external AI providers reliable.
- They do not justify adding distributed infrastructure without a measured need.

The system still relies on bounded retries, validation, fallback behavior, and defensive input handling.

## Reliability work included with the modernization

The platform upgrade was treated as an engineering change, not an isolated version bump.

The backend also hardened the voice path against:

- phonetic STT errors
- duplicated or mixed-language transcription
- malformed upstream AI responses
- empty successful responses
- fallback and recovery scenarios

Known cases receive deterministic handling; unclear input falls back to a safer engineering-focused response.

## Verification

The repository tracks backend and frontend automated test suites separately.

Test counts are **not** treated as proof of a green build. Pass/fail status belongs to the actual test runner output or CI result for the source state being evaluated.

## Guardrails

- Keep Virtual Thread enablement configuration-driven.
- Prefer explicit constructors when they communicate important logic; use Lombok when it only removes noise.
- Do not convert records to Lombok classes for stylistic consistency.
- Preserve the current single-service architecture unless measured requirements justify expansion.

## The point

A platform upgrade is more useful when it leaves behind a clearer system:

**better concurrency model + better failure handling + documented reasoning + tests protecting the change.**
