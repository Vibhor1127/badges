# 🎯 ACI — Interview Answers (STAR in Story Form)

> Each answer is written as **one flowing paragraph** that follows **STAR** — *Situation → Task → Action → Result* — but told like a story, in simple words mixed with the right technical terms. Read it aloud once and it should sound natural.

---

## 1. Summary of the Project
**Answer:**
The situation was that modern e-commerce teams sit on tons of data but still can't answer simple questions fast — like *"which products are losing margin because deliveries are late"* — because they have to scroll through dozens of static charts, and the usual "AI copilot" shortcut is dangerous since it lets a language model write raw SQL, which can hallucinate numbers or get injected. So my task was to build one platform that is both a working online store and a safe, fast business-intelligence layer on top of it. I built **ACI (AI-Powered E-Commerce Intelligence)** as a full-stack system: a **React 18 + TypeScript** single-page app on the front, a **Java 21 / Spring Boot** REST API in the middle, **MySQL (TiDB Cloud)** for transactional data, **Redis** as a distributed cache, and **Spring AI calling NVIDIA NIM's Nemotron 550B model** for language understanding. The flow is simple — the browser sends a JWT-protected request, Spring Security validates it, the controller hands it to a service, the service reads from MySQL or Redis, and where language is needed the LLM only *classifies intent* and *explains verified data*. The result is a platform where a natural-language question turns into a real, cached, audited answer in milliseconds instead of seconds, and it is deployed live — React on Vercel, backend on Render, and the whole stack runnable locally with one `docker compose up`.

**📐 Diagram for this answer — End-to-end request journey (one question, five hops)**

```
BROWSER            FRONTEND             BACKEND (Spring Boot)          DATA / AI
───────            ─────────            ─────────────────────          ─────────
User opens  ──▶  Vercel CDN serves
the app            React build
                     │
                     │  fetch('/analytics/dashboard')
                     │  + Authorization: Bearer <JWT>
                     ▼
                Nginx proxy ─── /api /auth /ai /analytics ───▶ [1] JwtAuthenticationFilter
                                                                   verify signature → who are you?
                                                               [2] SqlGuardrailValidator
                                                                   is this text safe?
                                                               [3] CapabilityRegistry
                                                                   which handler owns this?
                          │
                          ▼
                    SERVICE LAYER
             ┌────────────┼─────────────────┐
             ▼            ▼                 ▼
       MySQL / TiDB   Redis 7 Cache     NVIDIA NIM LLM
       14 tables      ~2 ms for repeat  classify intent +
       source of truth  questions        explain verified rows
             └────────────┴─────────────────┘
                          │
                          ▼
                   JSON response
      ◀── React Query caches it ──┘
User sees ◀── KPI cards / insight panel + chart
the answer
```

---

## 2. Why this project?
**Answer:**
The situation I observed was that almost every AI-analytics demo I saw had the same hole in it: the model was allowed to generate SQL directly, which is really an injection and hallucination waiting to happen, and dashboards were still the only way to get an answer out of a database. My task was to pick a project that would let me prove two things at once — that I can build a realistic, transaction-heavy business application, and that I can integrate an LLM *responsibly*. So I chose e-commerce because it naturally exercises everything: authentication, carts, orders, payments, inventory, reviews, state machines, and analytics, all in one domain. The action was to design it so the AI is deliberately kept away from the database — it classifies the question into an *entity* and an *operation*, a registered handler runs a pre-written, parameterized JPA query, and only then does the model explain the already-verified result. The result is that I got a project that demonstrates real engineering — caching, transactions, RBAC, testing, Docker, CI-style deployment — instead of a thin AI wrapper, and it gives me a strong story to tell about safety-first AI design.

**📐 Diagram for this answer — The risky way vs the ACI way (why I chose this project)**

```
❌ NAIVE "LLM WRAPPER"                          ✅ ACI (what I built)
─────────────────────                          ─────────────────────────

 User question                                  User question
      │                                              │
      ▼                                              ▼
 LLM WRITES SQL       ◀── user input          SqlGuardrailValidator
 "SELECT… ; DROP…"       injected here             │ blocks DDL/DML, ';', --, #, OR 1=1
      │                                              ▼
      ▼                                         LLM CLASSIFIES ONLY:
 raw string handed to DB                            { entity, operation }
      │                                              │
      ▼                                              ▼
 executes whatever                              CapabilityRegistry
 it received                                       │
      │                                              ▼
      ▼                                         pre-written JPA handler
 💥 data loss /                                     │
    hallucinated numbers                           ▼
                                              verified rows from MySQL
                                                       │
                                                       ▼
                                              LLM explains the data
                                                       │
                                                       ▼
                                              ✅ correct + cached + auditable
```

---

## 3. Your contribution / role in the project
**Answer:**
This was an end-to-end individual project, so my role covered the full stack rather than one slice of it. On the **backend** I designed the 14-table normalized schema, wrote the JPA entities, repositories with projections, and the whole service layer — checkout, cart, order lifecycle, inventory, reviews, JWT security, and the analytics engine with its capability registry, SQL guardrail, and both fallback paths for the AI. On the **frontend** I built the dual-persona React shell — a glassmorphism shopper storefront and a cyberpunk executive console — including the checkout wizard, the AI chat with its pipeline animation, and the Three.js data-galaxy visualisation. On the **DevOps** side I wrote the multi-stage Dockerfiles, the four-service Docker Compose with health checks, the Nginx reverse proxy, and I handled the Vercel + Render deployment with environment-driven configuration. My real contribution, though, was the **architecture decision**: separating intent classification from data access so the LLM never touches SQL, and making the analytics layer self-describing so a new domain is just one new handler class. The result is that I can speak about every layer with ownership, not just about the part I found fun.

**📐 Diagram for this answer — Three lanes I owned end-to-end**

```
┌─ LANE 1: BACKEND ─────────────────────────────────────────────────────────┐
│  14-table ERD ─▶ JPA entities ─▶ repositories + interface projections     │
│  CheckoutService (@Transactional)      OrderStatusService (EnumMap FSM)   │
│  JWT security + RBAC                   SqlGuardrailValidator              │
│  Analytics: CapabilityRegistry + 9 handlers + 2 AI fallbacks              │
└───────────────────────────────────────────────────────────────────────────┘
┌─ LANE 2: FRONTEND ────────────────────────────────────────────────────────┐
│  StoreShell (shopper)                  ConsoleShell (admin)              │
│  5-step checkout wizard (Zod)          AI chat + PipelineAnimation        │
│  InsightPanel + EvidenceVisualizer     Three.js data galaxy              │
│  AuthProvider / ProtectedRoute         ApiService + React Query          │
└───────────────────────────────────────────────────────────────────────────┘
┌─ LANE 3: DEVOPS ──────────────────────────────────────────────────────────┐
│  multi-stage Dockerfiles   Docker Compose (4 services + healthchecks)     │
│  Nginx reverse proxy        Vercel (front) + Render (API) deployment      │
└───────────────────────────────────────────────────────────────────────────┘
          │
          └──▶ THE DECISION THAT DEFINES MY CONTRIBUTION:
               AI classifies ──▶ deterministic handler answers ──▶ AI explains
               (never:  AI ──writes──▶ SQL)
```

---

## 4. What tools or software have you used? And why have you used them?
**Answer:**
I'll group them by the job each one does. For the **backend** I used **Java 21 with Spring Boot**, because the domain is transaction-heavy and I wanted mature, opinionated support for Spring Security, Spring Data JPA, transactions, validation, and REST — something a lighter stack would have made me build by hand. I used **Spring Data JPA** so my queries live as readable repository methods with interface projections instead of hand-written SQL strings. For **authentication** I used **JWT with jjwt** rather than server-side sessions, because a stateless token lets me scale to multiple backend instances without sticky sessions. For the **database** I used **MySQL 8 / TiDB Cloud** because the data is genuinely relational — orders, items, payments, shipments, inventory — and I wanted foreign-key integrity and a clean 3NF design. For **caching** I chose **Redis 7 (Upstash)** over an in-process cache like Caffeine, because Redis is shared across instances, survives restarts, and gives me namespace-level eviction after checkout. For **AI** I used **Spring AI with NVIDIA NIM's OpenAI-compatible endpoint**, because Spring AI gives me one abstraction and an easy timeout/fallback story. On the **frontend** I used **React 18 + Vite + TypeScript** for a fast dev loop and type safety, **TanStack React Query** for server-state caching and invalidation instead of Redux, **Tailwind CSS** for styling, **Recharts** for graphs, and **Three.js / React Three Fiber** for the 3D visuals. For **delivery** I used **Docker and Docker Compose** for reproducible multi-container local runs, **Nginx** as the reverse proxy, and **Vercel + Render** as the two managed hosts so the frontend and backend deploy independently. The **result** is that each tool has a one-line justification tied to a real problem, not just a "because it's popular" answer.

**📐 Diagram for this answer — Tool stack, layer by layer, with the reason**

```
LAYER          TOOL                         WHY (not just "it's popular")
─────          ────                         ──────────────────────────────
CLIENT        React 18 + TypeScript        type safety + huge ecosystem
              Vite                         instant HMR, fast cold build
              Tailwind CSS                 styling without leaving markup
              TanStack Query               server-state cache/invalidation
                                           (Redux = boilerplate here)
              Recharts                     charts straight from JSON
              Three.js / React Three Fiber 3D identity (starfield, galaxy)

API           Spring Boot + Java 21        @Transactional, Security, JPA,
                                           Bean Validation come built-in
              JWT (jjwt 0.13)              stateless → any instance serves
              springdoc / Swagger          living API contract

DATA          MySQL 8 / TiDB Cloud         real relations → FK integrity, 3NF
              Redis 7 (Upstash)            shared, TTL, namespace eviction

AI            Spring AI + NVIDIA NIM       one abstraction, easy timeout
                                           + fallback, OpenAI-compatible

DELIVERY      Docker + Compose             "works on my machine" → 1 command
              Nginx                        static files + reverse proxy
              Vercel / Render              front and back deploy independently
```

---

## 5. Explain by drawing a block diagram.

**Spoken version:** The story starts at the **client** — a React SPA hosted on Vercel — which never talks to the database directly; it only calls REST endpoints with a `Bearer` JWT. Those calls go through **Nginx**, which serves the static build and proxies `/api`, `/auth`, `/ai`, and `/analytics` to the **Spring Boot backend**. Inside the backend the request passes a fixed pipeline: the **JWT filter** proves who you are, the **SQL guardrail** proves the text is safe, the **capability registry** decides which handler owns this question, and the **service layer** does the work. The service then splits: **MySQL/TiDB** is the source of truth for the 14 relational tables, **Redis** answers repeat questions in under 5 ms, and **NVIDIA NIM** is only ever handed *classified intent* or *already-verified data* — never a chance to write SQL. The result is a diagram where every arrow has a single responsibility, and you can point at any box and explain why it exists.

**📐 Diagram 2 for this answer — Deployment topology (local vs production)**

```
DEVELOPMENT (one command)                 PRODUCTION (two platforms)
──────────────────────────                ───────────────────────────

docker compose up -d --build              ┌─────────────────────────────┐
     │                                    │ VERCEL — edge CDN           │
     ▼                                    │ React build, SPA rewrites   │
┌──────────────────────────────┐          └──────────────┬──────────────┘
│ frontend:3000  Nginx          │                        │ HTTPS + JWT
│   serves dist + proxies APIs  │                        ▼
│ backend:8081  Spring Boot JAR │          ┌─────────────────────────────┐
│ mysql:8.0     port 3307       │          │ RENDER — web service        │
│ redis:7       port 6379       │          │ Spring Boot container       │
└──────────────────────────────┘          │ /swagger-ui  /ai/ask  …     │
  depends_on: service_healthy             └──────┬──────────────┬───────┘
  (mysql + redis READY before backend)           │              │
                                                 ▼              ▼
                                          ┌────────────┐ ┌──────────────┐
                                          │ TiDB Cloud │ │ Upstash      │
                                          │ MySQL 8.0  │ │ Redis 7 TLS  │
                                          └────────────┘ └──────────────┘
                                                 │
                                                 ▼
                                          NVIDIA NIM (Nemotron 550B)
```

---

## 6. Explain the project.
**Answer:**
Let me walk it the way it actually runs. First, a **shopper** lands on the React app, registers, and `AuthenticationService` creates three linked records in one go — a `APP_USERS` row for credentials and role, a `CUSTOMERS` row for the business profile, and an empty `CART` — then issues a signed JWT that the frontend stores and attaches to every call. Browsing, cart, address and review flows are plain REST through `StoreController` into Spring Data JPA. When the shopper hits **checkout**, `CheckoutService` runs inside one `@Transactional` boundary: it validates the cart, checks stock for every line, creates the order and its order items, deducts inventory, writes an `INVENTORY_LOGS` row with `change_type = SALE`, simulates the payment (COD always succeeds, card/UPI about 90%), transitions the order through the **EnumMap finite state machine**, clears the cart, and evicts the `analytics` and `dashboard` cache namespaces — so if anything throws, everything rolls back together. On the other side, the **admin console** can move an order forward; the same state machine validates the transition, writes an `ORDER_STATUS_HISTORY` audit row, and if the new state is `CANCELLED`, `RETURNED` or `REFUNDED` it automatically restocks inventory and logs `STATUS_RESTOCK`. Then comes the part that makes the project special: the **AI cockpit**. Someone types *"who are my top spending customers?"* into `POST /ai/ask`. The guardrail scans it for DDL/DML, comments, semicolons and tautologies. Redis is checked first — a cache hit returns in ~2 ms. On a miss, the LLM is asked *only* to classify the intent, returning JSON like `{entity: CUSTOMER, operation: TOP_CUSTOMERS}`, built from a prompt the `CapabilityRegistry` generated from the real handler beans at startup. That intent is routed to `CustomerAnalyticsHandler`, which runs a pre-written JPA projection against MySQL. Only now does `AIExplanationService` send the *verified rows* to the model to produce `answer, reason, observations, recommendations`, which is sanitised, cached with a TTL, and rendered by the React `InsightPanel` with a chart chosen by the `EvidenceVisualizer`. The result: the AI can describe what the database says, but it can never invent what the database says.

**📐 Diagram 1 for this answer — Checkout (one @Transactional boundary)**

```
Cart ──▶ POST /api/store/checkout
  │
  ▼
[1] validate cart not empty ───── empty? ──▶ 400 ValidationError
  │
  ▼
[2] check stock for EVERY item ── short? ──▶ 409 ResourceConflict (nothing written)
  │
  ▼
[3] insert ORDERS                 status = PENDING
[4] insert ORDER_ITEMS            price snapshot per line
[5] products.stock -= qty         + INVENTORY_LOGS (change_type = SALE)
[6] simulate payment              COD = 100%   Card/UPI ≈ 90%
        ├─ success ─▶ CONFIRMED ─▶ PROCESSING
        └─ failure ─▶ CANCELLED (+ auto-restock)
[7] clear CART / CART_ITEMS
[8] evictAll("analytics") + evictAll("dashboard")
  │
  ▼
 COMMIT ──────────── any step throws? ──▶ ROLLBACK (whole checkout undone)
```

**📐 Diagram 2 for this answer — AI analytics pipeline (question → insight)**

```
"Who are my top spending customers?"
  │
  ▼
POST /ai/ask   (Bearer JWT)
  │
  ▼
[1] SqlGuardrailValidator ── DROP / ; / -- / OR 1=1 found? ──▶ 403 AiSecurityException
  │
  ▼
[2] Redis GET aci:ai:<hash> ──── HIT ─────────────────────────────────┐  (~2 ms)
  │ MISS                                                              │
  ▼                                                                   │
[3] LLM CLASSIFY ──▶ { entity: CUSTOMER, operation: TOP_CUSTOMERS}    │
  │      timeout / bad JSON? ──▶ heuristicIntent() (regex)            │
  ▼                                                                   │
[4] CapabilityRegistry ──▶ CustomerAnalyticsHandler                   │
  │                                                                   │
  ▼                                                                   │
[5] pre-written JPA projection on MySQL ──▶ verified rows             │
  │                                                                   │
  ▼                                                                   │
[6] LLM EXPLAIN ──▶ answer · reason · observations · recommendations  │
  │      model down? ──▶ deterministic synthesis                      │
  ▼                                                                   │
[7] sanitise (strip markdown code fences) ─▶ parse JSON              │
[8] SETEX aci:ai:<hash>  TTL 1h ◀────────────────────────────────────┘
  │
  ▼
InsightPanel  ──▶ EvidenceVisualizer picks the chart from entity + operation
```

---

## 7. What you learnt from this project?
**Answer:**
The situation was that I started this project thinking the hard part would be the AI integration, but the task that actually taught me the most was keeping the whole system *consistent* while it changed. The action was forcing myself to design the boring parts properly instead of patching them — I learnt that a **transaction boundary** is a design decision, not an annotation, because wrapping checkout in one `@Transactional` method is what makes partial failures impossible; I learnt that a **state machine beats a boolean**, because moving order status into an `EnumMap` of legal transitions removed a whole class of bugs and gave me an audit trail for free; I learnt that **caching is a correctness problem before it is a speed problem**, which is why invalidation is tied to mutation points like checkout and status change rather than to a blind TTL; and I learnt that **LLMs fail in boring, predictable ways** — timeouts, markdown fences, invented fields — so my system now sanitises output, validates JSON, and has heuristic fallbacks so it still works with no API key at all. I also picked up the practical skills: writing Docker multi-stage builds, wiring health checks so containers boot in the right order, and writing parameterised tests so security rules can't silently regress. The result is that I now design for failure first, and the happy path becomes almost an afterthought.

**📐 Diagram for this answer — The loop I now run by (design for failure)**

```
        ┌──────────────────────────────────────────────────────────┐
        ▼                                                          │
  DESIGN FOR FAILURE ──▶ keep state explicit ──▶ tie cache to the mutation
        ▲                                                          │
        │                                                          ▼
  prove it with tests ◀── fall back, never hang ◀── separate intent from data
        │                    (heuristicIntent,        (EnumMap FSM,
        │                     no-AI synthesis)          @Transactional)
        └──────────────────────────────────────────────────────────┘

FOUR LESSONS, ONE LINE EACH
───────────────────────────
@Transactional … a DESIGN decision, not an annotation → no half-written orders
EnumMap FSM …… state machine beats a boolean → invalid moves never hit the DB
Cache …………… correctness FIRST, speed later → invalidate on mutation, not on TTL
LLM …………… fails boringly (timeouts, junk output) → sanitise, validate, fall back
```

---

## 8. Which labs / companies did you visit for this project?
**Answer:**
This was a self-initiated, individual project, so there was no physical lab or company visit attached to it — I built and validated it on my own workstation. What I did instead was use **cloud labs and vendor free tiers as my environment**: **TiDB Cloud** gave me a managed MySQL-compatible database with a realistic distributed backend, **Upstash** gave me a production-style Redis over TLS, **Vercel** and **Render** acted as my deployment environment so I could test real HTTPS, cold starts and environment-variable configuration rather than `localhost`, and **NVIDIA NIM** provided the hosted LLM endpoint behind Spring AI's OpenAI-compatible client. On the study side I worked closely with official documentation and open-source references rather than a mentor — Spring Reference, Spring AI, Baeldung, the React and TanStack Query docs, the Redis documentation, and OWASP's SQL-injection guidance, which is what shaped the guardrail's block list. The result is that even without a lab visit, I can show a live deployed URL, an interactive Swagger page, and a repository where the same stack runs reproducibly through Docker Compose — which I think is stronger evidence than a visit certificate.

**📐 Diagram for this answer — My environment: a distributed cloud lab**

```
   ┌──────────────── MY WORKSTATION ────────────────┐
   │  IntelliJ / VS Code · Git · Docker · mvnw      │
   │  src/main/java · frontend/src · docker · compose│
   └─────────────────────┬──────────────────────────┘
                         │  git push
                         ▼
   ┌──── CLOUD SERVICES I USED AS SANDBOXES ────┐
   │  TiDB Cloud  → managed MySQL-compatible DB │
   │  Upstash     → Redis 7 over TLS            │
   │  NVIDIA NIM  → hosted LLM (via Spring AI)  │
   │  Render      → backend container + Swagger │
   │  Vercel      → frontend edge + SPA rewrites│
   └─────────────────────┬──────────────────────┘
                         │  read as primary sources
                         ▼
   ┌──── KNOWLEDGE SOURCES (instead of a mentor) ────┐
   │  Spring Reference · Spring AI · Baeldung       │
   │  OWASP SQLi Cheat Sheet · RFC 7519 (JWT)       │
   │  react.dev · TanStack Query · MDN · Docker docs│
   └────────────────────────────────────────────────┘
```

---

## 9. Which books / websites did you follow?
**Answer:**
I followed documentation and primary sources far more than books, because this stack moves too fast for printed material to stay accurate. For **Spring** I relied on the official *Spring Framework* and *Spring Boot* reference docs plus **Baeldung**, which is where I worked through `@Transactional` propagation, JPA projections and the security filter chain. For **Spring AI** I used the official Spring AI reference and NVIDIA NIM's API documentation to get the OpenAI-compatible endpoint, request shaping and timeout handling right. For **security** I read **OWASP's SQL Injection Prevention Cheat Sheet** and the **JWT / RFC 7519** material, and that is directly why the guardrail blocks stacked statements, comment evasion and tautologies instead of just filtering a few keywords. For the **frontend** I used **react.dev**, the **TanStack Query** docs (their "stale-while-revalidate" and invalidation guides shaped how I refetch after a mutation), the **React Router** docs for the protected-route pattern, and the **Three.js / React Three Fiber** docs for the starfield and data-galaxy scenes. I also used **MDN** for general browser behaviour, **Docker's** official docs for multi-stage builds and health checks, and **Swagger/OpenAPI** docs for contract-first endpoint design. The result is that every design decision in the project can be traced back to a primary source I can name if I'm asked.

**📐 Diagram for this answer — What I read → where it landed in the code**

```
SOURCE                          WHERE IT LANDED IN THE PROJECT
──────                          ─────────────────────────────────────────────
Spring Reference / Baeldung ───▶ @Transactional boundary in CheckoutService
                             ──▶ JPA projections (TopCustomer, MonthlyRevenue)
                             ──▶ filter chain rules in SecurityConfig

Spring AI + NVIDIA NIM docs ──▶ AIAnalyticsService (classify)
                             ──▶ AIExplanationService (explain)
                             ──▶ timeout + fallback design

OWASP SQLi Cheat Sheet ───────▶ SqlGuardrailValidator block list
                             ──▶ DDL/DML · stacked ';' · comments · tautologies

RFC 7519 / JWT docs ──────────▶ JwtService signing · JwtAuthenticationFilter

react.dev ────────────────────▶ AuthProvider · ProtectedRoute · hooks
TanStack Query docs ──────────▶ stale-while-revalidate + invalidate on mutation
React Router docs ────────────▶ role-aware routing (store vs console)
Three.js / R3F docs ──────────▶ starfield login · DataGalaxy
Docker docs ──────────────────▶ multi-stage Dockerfile · healthchecks
OpenAPI / Swagger ────────────▶ contract-first controllers
```

---

## 10. What are the results or conclusions?
**Answer:**
The result I'm happiest with is that the system does what it promised, measurably. Functionally, the full loop works end to end: a shopper can register, browse a 50-product catalogue across five categories, add to cart, run the five-step checkout, get a payment outcome, and track an order through a validated state machine with a complete audit history — while an admin can transition orders, adjust inventory with a reason code, and manage users, all behind role-based access. On the **performance** side, the Redis layer takes heavy aggregations from roughly **85 ms down to about 2 ms** on a cache hit — better than a 98% reduction — because repeat executive questions never touch the database again within the TTL. On the **safety** side, the conclusions are: the LLM never generates SQL, so the injection surface that kills most text-to-SQL projects simply doesn't exist here; every AI response is built from a verified dataset and then sanitised before parsing; and the system still returns correct, useful answers with **no AI key at all** thanks to the heuristic-intent and deterministic-explanation fallbacks. On **quality**, all **27 automated tests pass** — covering the guardrail's attack vectors, checkout atomicity, order-state transitions, JWT auth, and a JPA integration suite. The overall conclusion I'd draw is that *safety, speed and usefulness are not a trade-off* — with the right separation of concerns you can have all three, and a modular monolith is enough to deliver them without microservice overhead.

**📐 Diagram for this answer — Results, measured**

```
LATENCY — repeat analytics query
  DB only   ████████████████████████████████████████  ~85 ms
  Redis hit ██                                          ~2 ms   ( >98% ↓ )

SAFETY — what the LLM is allowed to do
  naive     LLM ──writes──▶ SQL ──executes──▶ 💥 injection / hallucination
  ACI       LLM ──classifies──▶ handler ──runs──▶ verified rows ──▶ LLM explains ✅

QUALITY
  27 test cases ─▶ guardrail (12 attack payloads) · checkout atomicity
                ─▶ FSM transitions · JWT auth · JPA integration
                ─▶ 0 failures · 100% pass

DEGRADED MODE (no API key at all)
  LLM down ──▶ heuristicIntent() + deterministic synthesis ──▶ still correct & safe
```

---

## 11. How this project can benefit our company or society?
**Answer:**
For a **company**, the first benefit is time-to-answer: today a manager opens a BI tool, filters, waits, and still interprets a grid — here they type a sentence in plain English and get a verified number with a reason, observations and a recommendation attached, so decisions that used to take minutes now take seconds and, more importantly, the number is provably correct because it came from a real JPA query and not from a model's imagination. The second benefit is **risk reduction**: because the AI is cut off from SQL, a company adopting this gets the productivity of an AI copilot without buying a SQL-injection or data-exfiltration liability, which is usually the blocker that stops AI projects reaching production. The third is **cost and speed** — the Redis layer removes repeated aggregation load from the transactional database, so the primary MySQL instance stays free for checkout traffic instead of being consumed by dashboards. For **society and smaller businesses**, the benefit is accessibility: a shop owner or a non-technical operator who has never written a query can still ask *"which products are delayed and hurting my margin"* and get a straight answer, which levels the playing field against large retailers that can afford dedicated data teams. It also reduces the dangerous habit of pasting company data into unsecured chatbots, since the analysis stays inside a system with authentication, role checks and audit logs. The result is a tool that makes good data analysis available to people who don't have analysts — without giving up correctness or security.

**📐 Diagram for this answer — How the value actually reaches people**

```
  Operator types a plain-English question
            │
            ▼
  Verified number (a real JPA query — not model imagination)
            │
            ├──▶ TIME      minutes of dashboard scrolling ──▶ seconds
            ├──▶ RISK      no SQL surface ──▶ no injection / data-leak liability
            ├──▶ COST      Redis absorbs aggregations ──▶ MySQL stays free for checkout
            └──▶ ACCESS    a shop owner with no data team gets analyst-grade answers
            │
            ▼
  Faster, safer decisions — and company data never leaves a system
  with JWT auth + RBAC + audit logs
  (instead of being pasted into an unsecured chatbot)
```

---

## 12. What are the limitations of your project?
**Answer:**
I'd group the limitations into four honest buckets. **First, the AI layer** — intent classification depends on an LLM, so a very unusual phrasing can still be misrouted; the heuristic fallback catches most cases, but it is keyword-based, so it is not a substitute for a trained classifier. **Second, payments and shipping are simulated** — payment success is modelled (COD always succeeds, card/UPI ~90%) and there is no real gateway, no PCI handling, no webhook reconciliation, and shipments have no live carrier integration. **Third, scale and architecture** — it is a single-instance modular monolith with a single database, so beyond a certain load I'd need read replicas, a Redis cluster, and a queue-worker model for the 3+ second LLM calls; there is also no WebSocket layer yet, so order and inventory updates rely on refetching rather than real-time push. **Fourth, data and UX scope** — the catalogue is a seeded 50-SKU demo dataset rather than a real feed, caching is TTL plus explicit eviction rather than event-driven invalidation everywhere, there is no full-text/faceted search, no recommendation engine, no email/notification service, and multi-tenancy, rate limiting and full observability (Micrometer/OpenTelemetry) are designed but not yet shipped. I deliberately present these as **known, scoped decisions rather than oversights**, because each one has a clear upgrade path already sketched in the architecture — and the result is that I can answer "what would you do next?" without having to think on the spot.

**📐 Diagram for this answer — Shipped vs deliberately deferred**

```
SHIPPED ✅                              DEFERRED / KNOWN LIMIT ⏳
──────────                              ─────────────────────────────────
Intent classification + 9 handlers      unusual phrasing can still misroute
                                        (fallback is keyword-based)

Simulated payments (COD / ~90% card)    no real gateway, PCI, webhooks

Single-instance modular monolith        read replicas · Redis cluster ·
                                        queue-worker for 3s LLM calls

TTL + explicit namespace eviction       event-driven invalidation everywhere

Refetch on mutation                     no WebSocket live updates

50-SKU seeded catalogue                 real product feed · search · recos

Single tenant                           tenant_id scoping for multi-tenant

Auth + RBAC + 27 tests                  rate limiting · Micrometer/OTel
          │
          └──▶ every deferred item already has a NAMED SLOT in the architecture,
               so "what's next?" is a plan, not a shrug
```

---

## 13. What was the systematic approach that you followed in your project?
**Answer:**
I followed an **iterative, incremental Agile approach** rather than a big-bang waterfall, because the requirements of an AI feature only become clear once you see the model actually behave. Concretely, the approach had five repeating phases. **Requirement analysis** — I wrote the problem statement first (dashboard overload, LLM hallucination, cache latency) and turned it into a feature list split across storefront, admin, analytics and AI. **Design** — I produced the architecture diagram, the ERD for the 14 tables, the order state diagram, and the AI sequence diagram *before* coding, which is what let me spot that the LLM must never generate SQL while it was still cheap to change. **Implementation in vertical slices** — I built in dependency order: schema and entities → auth/JWT → catalog and cart → checkout with transactions → order state machine and audit → analytics repositories → guardrail and capability registry → AI pipeline → frontend storefront → console → 3D/AI cockpit. **Testing** at every slice — unit tests alongside the code, then integration tests, then a context-load smoke test. **Deployment and iteration** — Docker Compose locally, then Vercel + Render, then reading failures and folding them back into the next iteration. Throughout, I used a **definition of done** that included tests passing and cache invalidation wired up, so "finished" meant more than "it runs". The result is a project where each increment left the system in a working, demo-able state, and no phase was ever allowed to run more than a step ahead of the others.

**📐 Diagram for this answer — The Agile incremental loop I followed**

```
 ┌──────────┐  ┌──────────┐  ┌────────────────┐  ┌────────┐  ┌─────────────┐
 │ 1 REQUIRE│─▶│ 2 DESIGN │─▶│ 3 IMPLEMENT    │─▶│ 4 TEST │─▶│ 5 DEPLOY &  │
 │  ment    │  │ arch·ERD │  │ vertical slice │  │ unit + │  │  ITERATE    │
 │ problem  │  │ state    │  │ schema → auth  │  │ integ  │  │ Docker →    │
 │ statement│  │ diagram· │  │ → cart →       │  │ + ctx  │  │ Render/     │
 │ + feature│  │ sequence │  │ checkout → FSM │  │ smoke  │  │ Vercel →    │
 │ list     │  │ BEFORE   │  │ → analytics →  │  │        │  │ read failure│
 │          │  │ code     │  │ AI → UI        │  │        │  │ → next loop │
 └──────────┘  └──────────┘  └────────────────┘  └────────┘  └──────┬──────┘
      ▲                                                            │
      └──────────────────── new requirement ◀──────────────────────┘

 DEFINITION OF DONE = runs + tests green + cache eviction wired
 (every increment ends in a demo-able state — never a broken main branch)
```

---

## 14. What were the problems faced in completion of this project?
**Answer:**
I'll share the four that taught me the most, because each had a real STAR story. **Problem one — AI hallucination and SQL risk.** My first version let the model draft a query, and it happily invented a column that didn't exist. The action was to redesign the pipeline so the model only returns a structured `{entity, operation}` intent, which a `CapabilityRegistry` maps to a pre-written JPA handler; I added a `SqlGuardrailValidator` as defence-in-depth plus JSON sanitisation that strips markdown fences. The result is zero hallucinated numbers and a 12-case security test suite proving it. **Problem two — stale dashboards after a transaction.** After checkout, revenue and KPI cards showed old values. The action was to replace blind TTLs with mutation-driven, namespace-level eviction in my custom `JsonCacheService`, so `evictAll("analytics")` runs at checkout and at every status transition. The result is consistency without clearing the whole cache. **Problem three — the external LLM being slow or unavailable.** Cold starts and timeouts made the AI cockpit hang. The action was an async call with a fail-fast timeout plus two fallbacks — `heuristicIntent()` for classification and deterministic synthesis for the explanation. The result is that the platform still works with no API key. **Problem four — container boot ordering.** The backend crashed with "connection refused" because MySQL wasn't ready. The action was adding `healthcheck` plus `depends_on: condition: service_healthy` in Docker Compose. The result is a deterministic, one-command startup. The overall result is that every major problem pushed the architecture to be more explicit, and none of them were ever papered over.

**📐 Diagram for this answer — Four problems → four fixes**

```
P1  AI invented a column; SQL injection risk
     └─▶ ACTION  model returns {entity, operation} only
                  CapabilityRegistry → pre-written JPA handler
                  + SqlGuardrailValidator + JSON sanitiser
          └─▶ RESULT  0 hallucinated numbers · 12 attack tests green

P2  Dashboard showed old revenue after checkout
     └─▶ ACTION  custom JsonCacheService, namespace eviction on mutation
                  evictAll("analytics") at checkout + at status change
          └─▶ RESULT  consistent without flushing everything

P3  LLM slow / cold start → AI cockpit hung
     └─▶ ACTION  async call + fail-fast timeout
                  heuristicIntent() + deterministic synthesis fallback
          └─▶ RESULT  platform works with NO API key

P4  Backend crashed with "connection refused"
     └─▶ ACTION  docker healthcheck + depends_on: service_healthy
          └─▶ RESULT  deterministic one-command startup
```

---

## 15. Software model used in your project.
**Answer:**
I used an **iterative and incremental Agile process model**, closest to a lightweight Scrum run by a single developer, and I chose it deliberately over the **Waterfall model**. The reason is that a big part of this project is AI behaviour, and you cannot fully specify how an LLM will phrase or classify a question on paper — you only find out by building a thin slice and observing it, so a linear phase-by-phase model would have forced me to redesign late and expensively. So I worked in short cycles, each ending with a *working, tested, demo-able* increment, and I prioritised a **walking skeleton** first — schema, auth, one end-to-end flow — before adding breadth. In terms of the **architecture model**, the project is a **modular monolith** laid out in a layered **MVC** style — Controller → Service → Repository → Entity, with DTOs at the edges — because it gives me strong domain separation without microservice overhead; the analytics handler package is isolated enough that it could be extracted into a standalone service later without touching the storefront. In terms of **design patterns**, the project uses **Strategy** (each analytics handler is an interchangeable implementation), **Registry** (`CapabilityRegistry` auto-discovers handlers at startup), **State** (the `EnumMap` order lifecycle), **Singleton** (the frontend `ApiService`), and **Facade** (service layer over repositories). The result is that if I'm asked "why not waterfall?" or "what pattern is that?", both answers come from decisions I made on purpose.

**📐 Diagram for this answer — Process model and architecture model side by side**

```
PROCESS MODEL (iterative / Agile)        ARCHITECTURE MODEL (modular monolith)
───────────────────────────────          ─────────────────────────────────────
 loop 1: walking skeleton   ┌─ CONTROLLER ── Auth · Store · Admin · AI · Analytics
 ──▶ working app            ├─ SERVICE ───── Checkout · OrderStatus · JsonCache
 loop 2: storefront                    AIAnalytics · AIExplanation
 ──▶ working app            ├─ HANDLER ───── CapabilityRegistry + 9 handlers
 loop 3: analytics         ├─ REPOSITORY ── JPA methods + projections
 ──▶ working app           ├─ ENTITY ────── 14 tables (3NF)
 loop 4: AI cockpit        └─ DTO/SECURITY ─ request/response + JWT filter
 ──▶ working app

 WATERFALL (rejected):  Requirements ─▶ Design ─▶ Build ─▶ Test ─▶ Deploy
                        one shot · late discovery · expensive rework

 DESIGN PATTERNS IN USE
  Strategy → every AnalyticsCapability is interchangeable
  Registry → CapabilityRegistry auto-discovers handlers at startup
  State    → EnumMap<OrderStatus, Set<OrderStatus>>
  Singleton→ frontend ApiService        Facade → service layer over repos
```

---

## 16. Testing Methodology.
**Answer:**
My testing methodology was **risk-based and layered** — I put the strongest coverage where a bug would hurt most: security, inventory, and order lifecycle. At the **unit level** I used **JUnit 5 with Mockito**, mocking repository and cache dependencies so I could test business rules in isolation: `SqlGuardrailValidatorTest` is parameterised over real attack payloads (`DROP TABLE`, `DELETE FROM`, `UPDATE products`, `INSERT INTO users`, `GRANT ALL`, `TRUNCATE`, comment and tautology evasions) and asserts both the rejections and that legitimate `SELECT`/`WITH` queries still pass; `CheckoutServiceTest` covers the happy path — order created, cart cleared, stock deducted, payment recorded, inventory logged — and the failure path, asserting that insufficient stock throws and nothing is committed; `OrderStatusServiceTest` verifies legal transitions, illegal transitions throwing, history rows being written, and auto-restock on cancel/return. At the **integration level** I used `spring-boot-starter-test` with real Spring Data JPA against the repository layer to prove the JPQL aggregations and interface projections actually return correct rows. At the **application level** an `@ApplicationTests` smoke test verifies the whole Spring context wires up — catching bean-wiring and configuration regressions. On the **frontend** the build is guarded by `tsc -b` in the Vite build, so type errors fail the pipeline. I also ran **manual end-to-end verification** of the deployed app — Swagger for API contracts and click-through of both personas. The result: **27 tests, 0 failures, 100% pass rate**, and more importantly a regression safety net that lets me refactor the analytics engine without fear.

**📐 Diagram for this answer — Risk-based test pyramid**

```
            ╱╲
           ╱  ╲        APPLICATION SMOKE (1)
          ╱    ╲       EcommerceAnalyticsApplicationTests
         ╱      ╲      → does the whole Spring context wire up?
        ╱────────╲
       ╱          ╲    INTEGRATION (5)
      ╱            ╲   OrderRepositoryIntegrationTest
     ╱              ╲  → real JPA, JPQL aggregations, projections return correct rows
    ╱────────────────╲
   ╱                  ╲  UNIT (21) — JUnit 5 + Mockito, repositories mocked
  ╱                    ╲ • SqlGuardrailValidatorTest  12 payloads:
 ╱                      ╲    DROP · DELETE · UPDATE · INSERT · GRANT · TRUNCATE
╱────────────────────────╲   '--' · 'OR 1=1' → REJECTED   SELECT / WITH → ACCEPTED
                            • CheckoutServiceTest   happy path + rollback on short stock
                            • OrderStatusServiceTest legal/illegal moves + auto-restock
                            • AuthServiceTest        JWT issuance & validation
                            • AnalyticsServiceTest   classification + cache behaviour
 ───────────────────────────────────────────────────────────────────────────────
  27 TEST CASES · 0 FAILURES · 100% PASS      frontend guarded by `tsc -b`
```

---

## 17. Questions related to technology used.
**Why Spring Boot and not Node.js?**
The situation was a domain with inventory, checkout, payments, RBAC and analytics all interacting, so my task was to pick a stack where correctness comes from the framework rather than from my own discipline. Spring Boot gave me declarative transactions, a mature security filter chain, JPA with type-safe queries and a huge testing ecosystem, all integrated out of the box; Node would have been fine for a thin service, but here I'd have been assembling and defending that plumbing myself. The result is that the codebase reads as convention-over-configuration and the reliability story is easy to tell.

**📐 Diagram — Spring Boot vs assembling it yourself in Node**

```
Spring Boot (chosen)                       Building the same in Node
───────────────────                        ──────────────────────────
@Transactional   → atomic checkout         manual transaction handling
Spring Security  → filter chain + RBAC     hand-rolled auth middleware
JPA              → typed queries +         query builder / raw SQL
                   interface projections
Bean Validation  → DTO rules               manual sanitising
Swagger/Actuator → contract + health docs  extra libs, extra glue
       │                                          │
       └──▶ code reads as convention     ──▶ plumbing I must defend,
            reliability comes free             document and test myself
```

**Why React and not Next.js?**
This is an authenticated SPA where all data arrives through JWT-protected API calls, so server-side rendering would add a whole rendering layer, hydration rules and route-handling complexity for no real SEO benefit — nobody indexes a private storefront. Vite + React 18 gave me a much faster dev loop, and React Router plus a `ProtectedRoute` component handled role-based navigation cleanly. The result is a simpler app that still feels modern and highly interactive.

**📐 Diagram — React + Vite vs Next.js**

```
React 18 + Vite (chosen)                   Next.js
───────────────────────                    ───────
data arrives via JWT API calls ─────────▶ SSR would render an EMPTY private shell
private storefront → SEO is irrelevant ──▶ no ranking benefit to earn
15+ pages · fast HMR ────────────────────▶ server routing model I don't need
ProtectedRoute + React Router ───────────▶ parallel server-side route handlers
       │                                         │
       └──▶ same interactivity,                  └──▶ extra runtime complexity
            simpler runtime                          for zero product value
```

**Why Redis and not Caffeine or Spring's `@Cacheable`?**
`@Cacheable` and local caches live inside one JVM, which breaks the moment I run more than one backend instance — one instance would hold stale data while another computes fresh values. Redis is a shared store outside the process, so every instance reads the same answer, it survives restarts, and my custom `JsonCacheService` can evict an entire namespace like `analytics` with one call after checkout — something the default annotation can't express. The result is consistent, cross-instance caching with explicit control over keys and TTLs.

**📐 Diagram — One JVM vs many (why Redis, not Caffeine)**

```
@Cacheable / Caffeine                    Redis (chosen)
─────────────────────                    ──────────────────────────
cache lives inside instance A            cache lives OUTSIDE the process
instance B keeps its own copy ──▶ 💥     every instance reads the SAME data
     inconsistent reads                  survives restarts · TTL per key
evict by method name only                evictAll("analytics") = whole namespace
       │                                         │
       └── fine for exactly 1 instance ──────────┘ required the moment you scale out
```

**Why doesn't the AI generate SQL directly?**
Because that is the single most dangerous design in an LLM application — a prompt injection or a hallucinated column turns into data loss or an injection vector, and you can't audit what you can't predict. So I split the job: the model only classifies intent and later explains verified rows, while actual data access goes through deterministic, parameterised JPA handlers chosen from a capability registry. The result is an auditable, testable, injection-free pipeline that still feels conversational to the user.

**📐 Diagram — The two versions side by side (AI writing SQL vs AI classifying)**

```
❌  LLM ──writes──▶ "SELECT … ; DROP TABLE orders" ──▶ DB executes ──▶ 💥
     (unauditable · injection · hallucinated columns)

✅  question ─▶ guardrail ─▶ LLM returns {entity, operation}
                          ─▶ CapabilityRegistry picks the handler
                          ─▶ handler runs a pre-written JPA query
                          ─▶ verified rows ─▶ LLM explains them
     (every row traceable to a query I wrote and tested)
```

**How does JWT authentication work here?**
Login posts credentials to `/auth/login`, `AuthenticationService` validates them and `JwtService` signs a token containing the user id and role; the frontend stores it and every request carries `Authorization: Bearer <token>`, where `JwtAuthenticationFilter` verifies the signature and expiry and populates the `SecurityContext`. Because the server keeps no session, any number of backend instances can serve any request. The result is stateless auth with RBAC enforced in `SecurityConfig` — `/api/admin` is ADMIN-only, `/api/store` is USER + ADMIN, `/auth` is public.

**📐 Diagram — JWT round trip**

```
 LOGIN                         EVERY LATER REQUEST
 ──────                        ───────────────────
 POST /auth/login              fetch('/api/admin/orders')
   │                             + Authorization: Bearer <token>
   ▼                               │
 AuthenticationService             ▼
   │  validate credentials     JwtAuthenticationFilter
   ▼                             ├─ verify signature (secret)
 JwtService signs token           ├─ verify expiry
   │  claims: sub, role           ├─ read role → authority
   ▼                             └─ set SecurityContext principal
 AuthResponse { token, role }        │
   │                                 ▼
   ▼                             SecurityConfig rules
 frontend stores in localStorage  ├ /auth        → public
   │                              ├ /api/store   → ROLE_USER, ROLE_ADMIN
   └──────────────────────────▶  ├ /api/admin   → ROLE_ADMIN only
                                 └ /ai/ask      → authenticated
                                       │
                                       ▼
                              controller runs under that role
  stateless ⇒ no server session ⇒ any backend instance can serve it
```

**Why 14 tables — isn't that over-engineered?**
No — an e-commerce domain is deceptively complex, and collapsing it would create duplication and inconsistent state. I kept `APP_USERS` separate from `CUSTOMERS` because authentication and business profile are different concerns, split `CART` from `CART_ITEMS` and `ORDERS` from `ORDER_ITEMS` so line items stay atomic, added `ORDER_STATUS_HISTORY` and `INVENTORY_LOGS` for auditability, and kept `PAYMENTS`, `SHIPMENTS`, `REVIEWS`, `ADDRESSES` independent. It's a clean **3NF** design, and the result is that analytics can aggregate across real relationships instead of parsing JSON blobs.

**📐 Diagram — Mini ERD (the 5 core tables of 14)**

```
 APP_USERS 1──────1 CUSTOMERS 1───────┬───────∞ ORDERS 1───────∞ ORDER_ITEMS ∞───────1 PRODUCTS
   (auth)        (business profile)   │            │                                   │
   user_id       customer_id, city    │            ├── 1────1 PAYMENTS                 │
   username      signup_date          │            ├── 1────1 SHIPMENTS                │
   password      email                │            ├── 1────∞ ORDER_STATUS_HISTORY     │
   role                                │            └── ∞────∞ REVIEWS ─────────────────┘
                                      │
                                      ├── 1────∞ ADDRESSES
                                      ├── 1────1 CART 1────∞ CART_ITEMS ∞────1 PRODUCTS
                                      └── ∞────∞ REVIEWS

 CATEGORIES 1────∞ PRODUCTS 1────∞ INVENTORY_LOGS (stock_before → stock_after, change_type)

 14 tables · 3NF · FK constraints · APP_USERS and CUSTOMERS split on purpose
 (credentials ≠ commerce profile), ORDERS/ORDER_ITEMS split so line items stay atomic,
 ORDER_STATUS_HISTORY + INVENTORY_LOGS exist purely for auditability.
```

**What is the Self-Describing Capability Pattern?**
Each analytics handler implements an `AnalyticsCapability` interface exposing `supportedEntity()`, `description()` and `supportedOperations()`; `CapabilityRegistry` discovers all those beans at startup and builds the LLM prompt from them dynamically. So adding Shipment analytics means writing one new `@Component` — the prompt, routing and validation all update themselves. The result is the Open-Closed Principle doing real work: the system extends without modification, and code and prompt can never drift apart.

**📐 Diagram — Self-describing capability pattern**

```
 STARTUP                                   RUNTIME
 ───────                                   ───────
 Spring scans @Component beans
   │
   ├── CustomerAnalyticsHandler ──┐
   ├── RevenueAnalyticsHandler ───┤  each implements AnalyticsCapability:
   ├── OrderAnalyticsHandler ─────┤    supportedEntity() · description()
   ├── InventoryAnalyticsHandler ─┤    supportedOperations()
   ├── ShipmentAnalyticsHandler ──┤            │
   ├── PaymentAnalyticsHandler ───┤            ▼
   ├── ProductAnalyticsHandler ───┼──▶  CapabilityRegistry aggregates metadata
   ├── ReviewAnalyticsHandler ────┤            │
   └── CustomerSatisfactionHandler┘            ├─▶ builds the LLM prompt (auto)
                                               ├─▶ validates returned intent (auto)
 ADD ShipmentAnalyticsHandler = 1 new class ──▶└─▶ routes to it (auto)
       no prompt edit · no router edit · no validator edit   ← Open/Closed Principle
```

**How do you cache and then invalidate correctly?**
It's cache-aside: check Redis first, on a miss run the query, store the JSON under a key like `aci:analytics:top-customers` with a 1-hour TTL, and serve it. Invalidation is tied to *mutations*, not to time — checkout and order-status changes call `evictAll("analytics")` and `evictAll("dashboard")` because revenue, order count and inventory KPIs all moved at once. The result is that speed never costs correctness: stale dashboards simply can't outlive the transaction that made them stale.

**📐 Diagram — Cache-aside, then invalidate on mutation**

```
 READ (cache-aside)
   request ─▶ JsonCacheService.get("analytics:top-customers")
                  ├─ HIT  ─▶ deserialise JSON ─▶ ~2 ms ─▶ return
                  └─ MISS ─▶ run JPA projection on MySQL
                              ─▶ serialise ─▶ SETEX aci:analytics:top-customers 3600
                              ─▶ return
 WRITE (evict, never overwrite)
   checkout / order status change / inventory adjust
                  ─▶ evictAll("analytics")
                  ─▶ evictAll("dashboard")
                  └─▶ next read MISSES ─▶ recomputed from the new truth

  TTL 1h  = safety net against forgotten writes
  eviction on mutation = the actual correctness guarantee
```

**How does the Finite State Machine work?**
Statuses are an `OrderStatus` enum and legal moves live in an `EnumMap<OrderStatus, Set<OrderStatus>>` that I wrap as unmodifiable — so `PENDING → CONFIRMED` is allowed, `DELIVERED → PENDING` is not, and an illegal move throws `InvalidOrderTransitionException` before anything touches the database. Terminal states like `CANCELLED`, `RETURNED` and `REFUNDED` additionally trigger restock of every order item plus an `InventoryLog` entry, and every move writes an `ORDER_STATUS_HISTORY` row. The result is that business rules live in code where they're fast, testable and impossible to bypass, instead of in a table that costs a query per transition.

**📐 Diagram — Order lifecycle (EnumMap finite state machine)**

```
                          ┌──────────┐
          checkout ──────▶│ PENDING  │
                          └────┬─────┘
               payment failed  │  payment verified
                 ┌─────────────┴────────────┐
                 ▼                          ▼
          ┌────────────┐            ┌────────────┐
          │ CANCELLED  │◀──────────│ CONFIRMED  │  cancel → AUTO-RESTOCK
          └────────────┘  cancel   └─────┬──────┘
          (terminal)                    │ warehouse picks
                                        ▼
                                  ┌────────────┐
                                  │ PROCESSING │──out of stock──▶ CANCELLED (restock)
                                  └─────┬──────┘
                                        │ carrier dispatch
                                        ▼
                                  ┌────────────┐
                                  │  SHIPPED   │
                                  └─────┬──────┘
                                        ▼
                              ┌───────────────────┐
                              │OUT_FOR_DELIVERY   │
                              └─────┬────────┬────┘
                          delivered │        │ failed / rejected
                                    ▼        ▼
                            ┌───────────┐  ┌──────────┐
                            │ DELIVERED │  │ RETURNED │ → AUTO-RESTOCK
                            └─────┬─────┘  └──────────┘
                                  ▼
                     COMPLETED / REFUNDED  (terminal)

 illegal move (e.g. DELIVERED → PENDING) ─▶ InvalidOrderTransitionException
 every legal move writes an ORDER_STATUS_HISTORY row (from, to, changed_by, at)
```

**How is the frontend structured?**
It's a dual-persona shell: `StoreShell` wraps the shopper routes (home, products, cart, five-step checkout, orders, profile) and `ConsoleShell` wraps the admin routes (dashboard, orders, inventory, reviews, users), with shared services and hooks underneath so nothing is duplicated. `AuthProvider` + `useAuth` centralise token, role and `isAuthenticated/isAdmin`, and `ProtectedRoute` blocks rendering for the wrong role before the request even leaves. A singleton `ApiService` attaches the JWT, TanStack React Query owns server state with invalidation on mutation success, and a replica layer can serve mock data if the backend health check fails. The result is two very different experiences driven by one API and one data model.

**📐 Diagram — Frontend: two personas, one API**

```
                                App.tsx
                                   │
                          AuthProvider (token, role)
                                   │
                          ProtectedRoute (role check)
                                   │
              ┌────────────────────┴────────────────────┐
              ▼                                         ▼
       /store  StoreShell                         /console  ConsoleShell
       glassmorphism · shopper                    cyberpunk · admin
   ┌───────────────────────────┐            ┌───────────────────────────┐
   │ Home · Products · Search  │            │ Dashboard (KPI + charts)  │
   │ ProductDetail + Reviews   │            │ Orders  (status FSM UI)   │
   │ Cart                      │            │ Inventory (ledger adjust) │
   │ Checkout (5-step wizard)  │            │ Reviews · Users           │
   │ Orders · OrderDetail      │            └─────────────┬─────────────┘
   │ Profile + Addresses       │                          │
   └─────────────┬─────────────┘                          │
                 └──────────────────┬─────────────────────┘
                                    ▼
                       singleton ApiService (attaches JWT)
                                    │
                       TanStack Query (cache + invalidate)
                                    │
                       replica.ts → mock fallback if health check fails
```

**How is it deployed and containerised?**
The backend Dockerfile is multi-stage — Maven builds the JAR, then only the JAR goes into a slim JRE image running as a non-root user; the frontend builds in Node and serves static assets from Nginx. `docker-compose.yml` wires four services (MySQL 8, Redis 7, backend, frontend) with `healthcheck` and `depends_on: condition: service_healthy` so the backend never starts before its dependencies, and Nginx proxies `/api`, `/auth`, `/ai` and `/analytics` to it. In production, Vercel hosts the SPA with rewrites and Render hosts the containerised API, both configured purely through environment variables. The result is `docker compose up -d --build` locally and a one-command reproducible stack anywhere else.

**📐 Diagram for this answer — Container startup order**

```
 docker compose up -d --build
          │
          ▼
 ┌─────────────────┐   healthcheck: mysqladmin ping
 │ mysql:8.0       │──────────────────────────────┐
 └─────────────────┘                              │
 ┌─────────────────┐   healthcheck: redis-cli     │
 │ redis:7         │──────── ping ──────────────┐ │
 └─────────────────┘                            │ │
                                                ▼ ▼
                                   depends_on: condition: service_healthy
                                      ┌─────────────────────┐
                                      │ backend:8081        │
                                      │ Spring Boot (JRE)   │
                                      │ non-root user       │
                                      └──────────┬──────────┘
                                                 │ backend healthy
                                                 ▼
                                      ┌─────────────────────┐
                                      │ frontend:3000       │
                                      │ Nginx: dist + proxy │
                                      └─────────────────────┘

 no more "connection refused" on boot — order is declared, not hoped for
```

---
