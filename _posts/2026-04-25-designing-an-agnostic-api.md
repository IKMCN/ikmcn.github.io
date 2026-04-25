---
layout: post
title: "Building a car management API from the ground up"
description: "A series of learning MVPs building toward a production-quality ASP.NET Core API with agnostic architecture, full testing, and CI/CD."
date: 2026-04-25
category: Architecture
series: Car Management API
series_part: 1
---

This is the first post in a series. Not a tutorial series where I already know all the answers — a learning series where I figure it out in public, build it in stages, and write about it as I go.

The goal is a car management API. Not because the world needs another car API, but because it is a domain simple enough to understand quickly and complex enough to be worth building properly. It is also, as it happens, the exact domain used in [Nick Chapsas'](https://nickchapsas.com) REST API course and the Packt book *Programming APIs with C# and .NET* — both of which have directly influenced the design here.

The honest starting point: six months ago I did not know what a constructor was. I am building this to understand how real production systems are designed, tested, and deployed. Every decision in this series is one I reasoned my way to, not one I was handed.

## What I am building

A REST API for managing cars. Users can read and filter. Admins can create, update, and delete. The system needs to be:

- **Agnostic at every layer** — the frontend, the authentication provider, and the database are all swappable without touching the core logic
- **Properly tested** — unit tests, integration tests, API contract tests, end-to-end tests, all automated
- **Production-quality** — versioning, health checks, pagination, rate limiting, security headers, structured logging, real-time notifications
- **Deployable anywhere** — Docker for development, Jenkins for CI/CD, and a three-machine homelab running bare metal Ubuntu for the target environment

The reference point for the final state is Nick Chapsas' course. His finished project is where this series ends up. But his final project is not where it starts — that would be learning nothing.

## The architecture

The key insight that shaped everything else: every dependency that might change should be behind an interface.

The frontend will probably start as vanilla JavaScript. It might become React. It does not matter — the API returns JSON and anything that can make an HTTP request can consume it.

Authentication starts as JWT. It might become Auth0. It does not matter — the controllers depend on `IAuthService`, not a specific provider.

The database starts as PostgreSQL. It might change. It does not matter — the application layer depends on `ICarRepository`, not a specific database technology.

Here is what the full architecture looks like:

```
Vanilla JS Frontend  (HTTP + WebSocket)
        ↓
ASP.NET Core Web API
        ├── IAuthService          → JwtAuthService / Auth0AuthService
        ├── ICarService           → CarService
        ├── ICarRepository        → PostgresCarRepository / FakeCarRepository
        └── ICarNotificationService → SignalRNotificationService / FakeNotificationService

Projects:
  Cars.Api          — controllers, hubs, middleware, auth, Swagger
  Cars.Application  — models, interfaces, repositories, services, validators
  Cars.Contracts    — request and response DTOs (the versioned surface)

Infrastructure:
  PostgreSQL        — running in Docker (dev) or bare metal Ubuntu (homelab)
  Nginx             — reverse proxy, HTTPS termination
  Jenkins           — CI/CD pipeline
  Seq               — structured log aggregation
  Prometheus        — metrics
  Grafana           — dashboards
```

The Contracts project is worth explaining. Your domain model and your API response object are not the same thing. `Car` is your internal model. `CarResponse` is what the client receives. Separating them means you can change one without breaking the other — and it is the Contracts project that gets versioned, not your domain.

## The testing strategy

Testing is not added at the end. It is built alongside every MVP from day one. By the final MVP the test suite has six layers:

**Unit tests** — individual classes in isolation, no database, no HTTP, fast. xUnit and NSubstitute throughout.

**Integration tests** — the API running against a real PostgreSQL instance in a container, using TestContainers. These verify the full request/response cycle including database.

**API contract tests** — Postman collection covering happy paths and unhappy paths for every endpoint. Newman runs these automatically in CI.

**End-to-end tests** — Playwright driving the JavaScript frontend through real user journeys. Login, browse, filter, admin CRUD, verify real-time updates.

**Performance baseline** — response time benchmarks captured in CI so regressions are caught early.

**Post-deployment smoke tests** — a lightweight suite that runs after every deployment to verify the live system is responding correctly.

## The MVP roadmap

Each MVP is a working system. Not a prototype that gets thrown away — each one builds directly on the last.

---

**MVP 1 — The bones**

Single project. `Car` class, one controller, in-memory list as the data store. GET all, GET by id, POST, PUT, DELETE. Swagger included from day one so every endpoint is immediately testable.

*Testing:* xUnit project set up alongside the API. Unit tests on the controller with a fake repository injected. The test habit starts here.

---

**MVP 2 — Real data**

PostgreSQL via Docker. Dapper for data access. The project splits into `Cars.Api` and `Cars.Application`. `ICarRepository` is introduced here — one implementation now, but the pattern is established.

*Testing:* Integration tests using TestContainers — a real PostgreSQL container spins up, the API runs against it, tests verify data is written and read correctly. GitHub Actions runs the full suite on every push.

---

**MVP 3 — Validation and errors**

FluentValidation on all request objects. `ValidationMappingMiddleware` so errors are consistent RFC 7807 ProblemDetails responses. Global error handling middleware catches anything that slips through.

*Testing:* Unit tests on validators in isolation. Integration tests covering unhappy paths — missing required fields, invalid data types, boundary conditions. Postman collection started: happy paths and unhappy paths for every endpoint.

---

**MVP 4 — Authentication and authorisation**

JWT with two roles: read-only user and admin. Admin gets full CRUD. Users get GET with filtering. `IAuthService` introduced so the provider is swappable. Refresh token strategy included — short-lived access tokens, long-lived refresh tokens.

*Testing:* Unit tests on the auth service. Integration tests verifying unauthenticated requests are rejected, role boundaries are enforced, token expiry is handled correctly. Postman collection extended with auth flows. Newman added to GitHub Actions.

---

**MVP 5 — Production features**

API versioning. Health checks endpoint. Output caching with cache tag invalidation. Pagination on list endpoints. Rate limiting. Security headers middleware. CORS configuration. The `Cars.Contracts` project splits out here.

*Testing:* Integration tests covering pagination — correct page returned, correct total in response headers. Versioning tests — v1 and v2 endpoints behave correctly. Cache invalidation verified. Performance baseline captured.

---

**MVP 6 — Frontend and end-to-end**

Vanilla JS frontend. The full journey: browse cars, filter by make and model, admin login, add a car, see it appear. Playwright introduced for frontend automation.

*Testing:* E2E tests covering the complete user journey. Admin journey. These run in GitHub Actions against a fully Dockerised stack — API, database, and frontend all in containers.

---

**MVP 7 — CI/CD and deployment**

Jenkins pipeline. Docker Compose for the full stack. Deploy to homelab: PostgreSQL on machine one, API on machine two, frontend on machine three. Nginx as the reverse proxy. Database migration strategy in place.

*Testing:* The pipeline runs unit tests, integration tests, Newman, and Playwright in order. Any failure blocks deployment. Test reports published as pipeline artefacts. Post-deployment smoke tests run automatically after a successful deploy.

---

**MVP 8 — Real-time**

SignalR. A `CarHub` with `ICarNotificationService` behind it — so the hub is also swappable. When admin adds, updates, or deletes a car, all connected clients see the change without refreshing.

*Testing:* SignalR integration tests verify clients receive push notifications when CRUD operations happen. Playwright tests extended to verify updates appear in the UI in real time.

---

**MVP 9 — Observability**

Serilog with Seq for structured logging. Prometheus metrics. Grafana dashboards. OpenTelemetry distributed tracing so a request can be followed from the frontend through the API to the database. Secrets management — nothing sensitive in config files.

*Testing:* Log assertions in integration tests verify structured entries are written for key operations. Health check endpoint tested in CI. Grafana dashboards verified post-deployment.

---

## Why build it this way

The architecture is more complicated than necessary for a car API. That is the point. The car API is a vehicle — the real goal is understanding how production systems are put together: why they are structured the way they are, why testing is non-negotiable, why every dependency should be behind an interface.

Nick Chapsas' course shows what the destination looks like. This series is the journey from knowing what a constructor is to being able to read, understand, and build from that destination.

Each post in the series will cover one MVP: the decisions made, the code written, the tests that prove it works, and what I learned that I did not expect.

The code will be on GitHub. The mistakes will be in the posts.

---

*Next: MVP 1 — getting the bones in place. A working API, a test project, and nothing else.*
