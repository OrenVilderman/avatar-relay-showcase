# AvatarRelay Showcase

> **An AI digital twin portfolio system built to demonstrate backend engineering, AI integration, reliability, and recruiter-focused UX.**

[![Java 21](https://img.shields.io/badge/Java-21-orange?logo=openjdk&logoColor=white)](https://www.oracle.com/java/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.1.1-6DB33F?logo=springboot&logoColor=white)](https://spring.io/projects/spring-boot)
[![Angular 22](https://img.shields.io/badge/Angular-22-DD0031?logo=angular&logoColor=white)](https://angular.dev/)
[![TypeScript](https://img.shields.io/badge/TypeScript-6-3178C6?logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Groq](https://img.shields.io/badge/LLM-Groq-111111)](https://groq.com/)

## Source code and IP

The production source repository is intentionally **private** to protect implementation IP.

This showcase repository exposes the engineering story without publishing the full codebase: architecture, API boundaries, Mermaid diagrams, reliability decisions, testing strategy, and the live product.

**Private source repository:** https://github.com/OrenVilderman/avatar-relay

Source access can be provided to hiring managers and engineering interviewers on request.

## Engineering decision example

A small example of how I document architectural trade-offs is included here:

**[ADR 008 — Java 21 Platform Modernization](docs/decisions/ADR-008-java21-platform-modernization.md)**

It shows the structure I use for a real decision: context → decision → trade-offs → verification → guardrails.
The public document intentionally focuses on the reasoning, not private implementation details.

## Executive overview

AvatarRelay is a recruiter-facing AI digital twin for Oren Vilderman's backend engineering portfolio.

A recruiter can:

- Ask questions by text or voice in English or Hebrew.
- Receive CV-grounded answers through a deterministic response layer.
- Hear the response through the avatar experience.
- Paste a real Job Description and receive a structured comparison against the candidate's documented experience.
- Move directly from technical exploration to contact actions such as scheduling a call or viewing the CV.

The engineering focus is deliberately broader than "calling an LLM":

- bounded model responsibility
- deterministic response resolution
- malformed-response recovery
- retry and fallback behavior
- prompt-injection boundaries
- server-owned video content
- safe error handling
- failure-oriented automated testing
- bilingual voice UX

## Live demo

**Live application:** https://avatar-relay.netlify.app/

**GitHub profile:** https://github.com/OrenVilderman

**LinkedIn:** https://linkedin.com/in/oren-vilderman

## Architecture at a glance

The current product uses one Spring Boot backend rather than introducing distributed infrastructure without a measured need.

```mermaid
flowchart LR
    U[Recruiter] --> W[Angular 22 Web App]

    W -->|POST /api/chat| API[Spring Boot API]
    API -->|Intent classification| LLM[Groq LLM]
    LLM --> API
    API --> R[Canonical Response Resolver]
    R --> S[In-memory Reply Store]
    S --> W

    W -->|POST /api/videos| API
    API --> V[Video Job + Budget Controls]
    V --> P[AvatarProvider]
    P --> W

    U -->|Job Description| W
    W -->|POST /api/match-jd| JM[Job Matcher]
    JM --> LLM
    JM --> F[Validated Match Response]
    F --> W
```

### Chat + avatar sequence

```mermaid
sequenceDiagram
    autonumber
    actor Recruiter
    participant App as Angular App
    box rgb(30, 41, 59), "Spring Boot API (Java 21)"
        participant PT as Platform Carrier Thread (OS)
        participant VT as Virtual Thread (Project Loom)
        participant Store as In-Memory Reply Store
    end
    participant Groq as Groq API (External LLM)

    Recruiter->>App: Voice Question (STT completed in browser)
    App->>App: Sanitize & normalize transcript
    App->>+PT: POST /api/chat (HTTP Request)
    
    Note over PT, VT: [Java 21 Optimization] Mounts light Virtual Thread
    PT->>+VT: Hands off request execution
    PT-->>-App: [OS Thread Released] Immediately free to handle next client HTTP requests!

    VT->>VT: Validate request + Apply Rate Limit
    VT->>+Groq: POST /chat/completions (Blocking I/O Call)
    
    Note over VT, Groq: VT is unmounted while waiting. 0 MB OS memory wasted.
    Groq-->>-VT: Returns Structured Intent JSON (e.g., EXPERIENCE)
    
    VT->>VT: FallbackResponsePolicy (Apply defensive token checking)
    VT->>Store: Resolve canonical reply from context
    Store-->>VT: replyId + deterministic portfolio response
    
    VT->>+PT: Re-mounts to complete response
    PT-->>-App: HTTP 200: JSON Response (replyId + reply)
    deactivate VT

```

## Key engineering features

### 1. Bilingual voice UX

The frontend uses the browser Web Speech API for voice input.

The voice path is designed around real browser speech-recognition failure modes rather than assuming a clean transcript:

- English and Hebrew input.
- Continuous recognition with interim results.
- Silence-based automatic submission.
- Generation guards to prevent stale recognition instances from updating current state.
- Safe fallback behavior for empty, noisy, or unusable speech transcripts.
- Transport failures degrade to a useful local response instead of exposing raw API errors.

The goal is a voice interaction that remains predictable even when speech recognition is imperfect.

### 2. High-throughput I/O concurrency (Java 21 Virtual Threads)
The backend architecture scales per-instance concurrency efficiently under I/O-heavy workloads by utilizing Java 21 Virtual Threads (`spring.threads.virtual.enabled=true`).
- The backend keeps its blocking HTTP integration for simplicity while using Virtual Threads for I/O-heavy request execution.
- The goal is better per-instance concurrency efficiency under high numbers of concurrent waits, not a promise of unlimited scale or unlimited memory capacity.

### 3. Bounded LLM responsibility

The model is intentionally not the source of truth for the candidate's profile.

```text
User input
   -> LLM classification / structured analysis
   -> validated application-level result
   -> deterministic application response
```

For recruiter chat, the LLM classifies intent while the application resolves the final response from controlled candidate information.

For Job Matcher, the model analyzes the Job Description but the CV remains the trusted source for claims about Oren's skills and experience.

### 4. Job Matcher resilience

Recruiter Job Descriptions are untrusted and often long, messy, or unexpectedly formatted.

The recovery strategy is staged:

```text
full JD
  -> Unicode normalization
  -> primary model
  -> bounded retry
  -> fallback model
  -> bounded retry
  -> high-signal JD compaction when needed
  -> model cascade again
  -> deterministic local fallback
```

The application validates the response envelope before accepting it:

```json
{
  "category": "JOB_MATCH | TECHNICAL_QA | WITTY_FALLBACK",
  "language": "he | en",
  "headline": "Short title",
  "matchScore": 0,
  "summary": "Validated response text",
  "highlights": ["point 1", "point 2"],
  "cta": {
    "text": "Deterministic presentation text",
    "action": "schedule_video | schedule_phone | view_cv"
  }
}
```

Invalid categories, invalid CTA actions, malformed JSON, empty model responses, missing fields, or invalid scores are recovery events rather than silently accepted data.

### 5. Server-owned avatar text

The browser does not submit arbitrary video text.

The frontend submits a server-generated `replyId`, and the backend resolves the canonical response before creating the video job.

This keeps the avatar layer aligned with the same controlled response that was shown to the recruiter.

### 6. Failure-aware integrations

External providers are treated as unreliable infrastructure.

Handled failure classes include:

- transport errors
- 4xx / 5xx responses
- empty successful HTTP bodies
- malformed JSON
- missing response fields
- empty `choices`
- unexpected enum values
- truncated model output
- rate limiting
- temporary provider failures

The application uses bounded retries, fallback models, validation and deterministic degradation where appropriate.

### 7. Testing rigor

Testing is designed around the actual failure surface, not only the happy path.

Coverage areas include:

- request validation
- structured LLM response parsing
- empty and malformed upstream responses
- retry and fallback behavior
- long Job Descriptions
- Job Matcher validation
- bilingual behavior
- voice interaction state transitions
- transport-failure degradation
- server-owned reply/video mapping
- provider isolation
- rate limiting and error contracts

Automated tests do not depend on live Groq or live D-ID execution. The avatar provider is mock-first, and real D-ID execution is intentionally frozen.

> Test status is intentionally not inferred from README counts. Current pass/fail status should be taken from the exact local test run or CI result for the commit being evaluated.

## Local API Contract Testing (Postman + Newman)

The private implementation includes a local `npm run pipeline` workflow
that coordinates backend startup and automated API contract tests.

The pipeline addresses a practical testing problem: the Spring Boot
application takes approximately nine seconds to start in the observed
local environment, and its Java/Tomcat process may outlive the background
launcher. Without explicit lifecycle management, Newman can run too early
or send requests to a stale application instance.

The pipeline coordinates these stages:

1. **Preflight:** Detect stale listeners on the local development port
   before starting a new run.
2. **Startup:** Launch Spring Boot as a background process and capture
   its launcher PID.
3. **Readiness:** Wait for the API to become reachable before starting
   Newman.
4. **API validation:** Run the Postman collection against the intended
   API instance.
5. **Cleanup:** Release the local server resources at the end of the run
   and propagate startup or test failures to the pipeline result.

Port cleanup is a local-development safeguard, not a production
process-management strategy. The cleanup implementation should be
scoped to the process owned by the pipeline and must not terminate
unrelated applications.

### Security boundary validation

The deployed API has been exercised with an invalid API key, and the
request receives an authorization rejection (`401`/`403`).

This verifies one negative authorization scenario. It does not, by
itself, establish complete security coverage.

The API contract suite should explicitly distinguish:

- Successful requests using valid credentials.
- Requests with missing or invalid credentials.
- Request-validation failures.
- Unexpected server and upstream-provider failures.

### Operational limitation

The deployed service has some cold-start latency. Startup readiness,
API correctness, and upstream-provider availability are separate
concerns and should be diagnosed independently.

The implementation remains private. This public repository documents
the design, verification strategy, limitations, and engineering
trade-offs; the local pipeline is not runnable from the showcase
repository itself.

```mermaid
flowchart LR
    A[Local developer] --> B[npm run pipeline]
    B --> C[Preflight and stale-port check]
    C --> D[Start Spring Boot]
    D --> E{API ready?}
    E -->|No, within timeout| E
    E -->|No, timeout| F[Fail pipeline]
    E -->|Yes| G[Run Newman collection]
    G --> H{Assertions pass?}
    H -->|Yes| I[Cleanup and success]
    H -->|No| J[Cleanup and failure]
```

<a id="security-verification-layers"></a>

## Security Verification Layers

Security verification uses complementary checks at different boundaries. External contract tests validate observable HTTP behavior, while the internal route inventory check detects registered MVC mappings that have not been explicitly approved.

| Verification Layer | Boundary Evaluated | Execution Context | Output Signal |
| :--- | :--- | :--- | :--- |
| **Token Check Unit Tests** | Individual interceptor component correctness | Build-time CI / Local | Validates header parsing logic isolation |
| **Route Inventory Guard** | Live MVC mapping parity & anonymous request leaks | Build-time CI / Local | Catches undeclared controller routes and seals origin surface |
| **Newman Contract Suite** | Black-box HTTP validation of deployed API schemas | Local pipeline automation / Live Prod Check | Verifies active schema definitions match production expectations |

While external black-box contract checks (Postman/Newman) ensure that our running endpoints strictly honor documented API specifications across environments, the internal Route Inventory Guard acts as an introspective runtime backstop. It dynamically queries Spring Boot's live mappings during the test phase to prevent accidental framework endpoint exposure.

### Visual Guard Indicator (Terminal Output Failure Simulation)

When an undocumented controller route attempts to register on the origin server (for example, a temporary `@GetMapping("/debug/env")` endpoint), the integration test halts the pipeline and reports a targeted diagnostic mismatch:

```text
[ERROR] Failures:
[ERROR]   RouteInventoryGuardTest.routeSetMatchesTheDeclaredInventory:97
Route inventory mismatch:
Undeclared route: GET /debug/env
[INFO]
[ERROR] Tests run: 6, Failures: 1, Errors: 0, Skipped: 0
```

The guard covers registered Spring MVC mappings, not every URL a server could serve. Keep separate negative-path checks for authorization and endpoints outside the protected namespace; neither the inventory nor a green contract suite alone proves complete security coverage.

## API surface

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/chat` | Validate and classify a recruiter question, then return a canonical response and `replyId`. |
| `POST` | `/api/videos` | Start a video job from a server-owned `replyId`. |
| `GET` | `/api/videos/{videoId}` | Poll avatar-video job status. |
| `POST` | `/api/match-jd` | Analyze a recruiter Job Description against trusted candidate context. |
| `GET` | `/api/health` | Service health check. |

## Technology choices

| Layer | Technology |
|---|---|
| Backend | Java 21, Spring Boot 4.1.1, Maven (with Virtual Threads & Lombok) |
| Frontend | Angular 22, TypeScript, Signals, RxJS, TailwindCSS |
| LLM | Groq API with structured responses |
| Voice | Browser Web Speech API |
| Avatar | `AvatarProvider` abstraction, mock provider, frozen D-ID provider |
| State | In-memory |
| Testing | JUnit 5, Spring test infrastructure, Vitest |
| Frontend hosting | Netlify |
| Backend hosting | Render |

## Why a monolith?

The system intentionally uses one backend service.

That choice keeps the current product easier to:

- reason about
- test
- secure
- deploy
- debug

The architecture can evolve when traffic, durability, or operational requirements justify it. The project does not introduce Kafka, a database, or additional distributed services simply to make the diagram look more complex.

## Selected engineering decisions

| Decision | Why |
|---|---|
| Deterministic response layer | Keeps candidate claims grounded and responses synchronized with the avatar. |
| Model classification instead of unrestricted answer generation | Limits hallucination surface and makes application behavior testable. |
| Bounded retry + fallback | Improves resilience without creating unbounded retry loops. |
| Server-owned video text | Prevents arbitrary client-controlled avatar content. |
| Mock-first avatar provider | Keeps automated tests deterministic and avoids live provider usage. |
| In-memory state | Keeps portfolio-scale deployment simple while the product is still intentionally lightweight. |
| Polling for video state | Avoids WebSocket/SSE complexity for the current product requirement. |

## Showcase structure

This repository is intentionally lean. The public surface is organized around the questions a recruiter or engineering manager typically needs answered quickly:

```text
avatar-relay-showcase/
├── README.md
└── docs/
    └── decisions/
        └── ADR-008-java21-platform-modernization.md
```

The `docs/decisions/` directory is the public home for concise architecture decision records, following the `ADR-NNN-short-topic.md` naming convention. Add a standalone security-verification decision record there when one is warranted. Keep executable implementation tests and private gateway credentials in the private source repository; do not publish raw Java source files or live tokens in this showcase.

The detailed implementation remains in the private source repository.

## About the engineering approach

AvatarRelay is an example of how I approach backend systems where AI is one component rather than the architecture itself:

1. Define a narrow responsibility for the model.
2. Validate external responses at every boundary.
3. Treat all external and user-provided input as untrusted.
4. Recover from realistic failure modes before failing the user.
5. Keep business-critical behavior deterministic where possible.
6. Build tests around failure scenarios, not only successful requests.
7. Add infrastructure only when the problem justifies it.

## Contact

**Oren Vilderman**

- Email: OrenVilderman@gmail.com
- LinkedIn: https://linkedin.com/in/oren-vilderman
- GitHub: https://github.com/OrenVilderman
- Live demo: https://avatar-relay.netlify.app/

For engineering interviewers who want to review the implementation, private source access is available on request.
