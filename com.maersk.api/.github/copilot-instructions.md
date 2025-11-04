# Copilot / AI agent instructions — maersk-booking-api

This repository is a small Spring Boot (WebFlux) service for booking checks and creating bookings.
Follow these concise, actionable rules when making edits or generating code.

Key architecture (read before changing behavior)
- Framework: Spring Boot 3.x, reactive WebFlux (see `pom.xml`). Java 17 is the target.
- Data: Reactive MongoDB via Spring Data Reactive Mongo (models under `src/main/java/com/booking/model`, repository interfaces under `.../repo`). Documents use `@Document` and `@Field` to map Mongo field names (example: `BookingDocument`).
- API layer: REST controllers under `src/main/java/com/booking/controller` (example: `BookingController`). They return Reactor types (`Mono<...>`).
- Services: Business logic lives under `src/main/java/com/booking/service` (examples: `AvailabilityService`, `BookingService`, `BookingRefGenerator`). Services return Reactor types and avoid blocking calls.
- External I/O: HTTP calls use `WebClient` configured in `com.booking.config.WebClientConfig` (bean: `maerskWebClient`). When adding new external calls, reuse that bean or add a `@Qualifier` if multiple `WebClient` beans exist.

Important patterns & conventions (do not break these)
- Reactive-first: Use Reactor types (Mono/Flux) in controllers and services. Do not convert to blocking calls (no .block()) in production code.
- Error handling: Services often return `Mono.error(...)` for validation problems and map persistence errors to simple runtime exceptions (see `BookingService#createBooking`). Preserve the existing pattern of mapping upstream errors to domain-friendly codes/messages.
- External calls resilience: `AvailabilityService` uses `retryWhen(...)` and `onErrorResume(...)` to fall back to safe defaults. Follow similar resilience patterns when calling external systems.
- Tests: Controller tests use `@WebFluxTest` with `@MockBean` for services and `WebTestClient` to exercise endpoints (see `src/test/java/com/booking/controller/BookingControllerTest.java`). For repository/integration tests, the project uses `de.flapdoodle.embed.mongo` (embedded Mongo) in test scope.

Build / run / test (developer workflows)
- Build and run tests: `mvn clean test` or `mvn -DskipTests=false test`.
- Package: `mvn -DskipTests=true package` (or omit skip to run tests during package).
- Run locally: ensure a MongoDB is available or set `MONGODB_URI`. Default in `application.yml`: `mongodb://localhost:27017/maersk_domain`.
- Run the app: `mvn spring-boot:run` or run the generated JAR from `target/`.
- Run only controller tests: `mvn -Dtest=com.booking.controller.BookingControllerTest test`.

Integration points & environment notes
- External availability API: `AvailabilityService` POSTs to `maerskWebClient` at `/api/bookings/checkAvailable`. The base URL is set in `WebClientConfig` (`https://maersk.com`) — treat this as a placeholder for the real upstream in dev/staging.
- Environment variable: `MONGODB_URI` controls Mongo connection (see `application.yml`).

Files to read for context when changing behavior
- `pom.xml` — dependencies and Java/Spring Boot versions.
- `src/main/java/com/booking/controller/BookingController.java` — API surface and validation annotations.
- `src/main/java/com/booking/service/*` — business rules and inter-service flows.
- `src/main/java/com/booking/config/WebClientConfig.java` — external HTTP client wiring.
- `src/test/java/com/booking/controller/BookingControllerTest.java` — unit test patterns (WebTestClient + MockBean).

Quick examples for common changes
- Add a new endpoint: put REST class under `controller`, return Reactor types (Mono/Flux), validate with `@Valid` and request DTOs in `dto` package.
- Call a new external HTTP endpoint: reuse `maerskWebClient` bean; follow `AvailabilityService` pattern (bodyValue -> retrieve -> bodyToMono -> retryWhen -> onErrorResume).
- Persist a new document: create a `@Document` in `model`, add repository interface in `repo` extending `ReactiveMongoRepository`, use `repo.save(...)` from services and map persistence exceptions similarly to `BookingService`.

If you make database or API contract changes
- Update `BookingDocument`/DTOs and data mapping annotations (`@Field`) consistently.
- Update tests: controller tests use mocked services; repository changes require integration tests (embedded Mongo).

What not to change without confirmation
- Global error mapping and simple domain error strings (e.g. `DB_SAVE_ERROR`) — these may be relied on by clients/tests.
- The WebClient base URL in `WebClientConfig` without coordinating upstream test stubs.

When unsure, ask the human reviewer for:
- Real upstream API contracts (payload shapes and URLs) before changing `AvailabilityService`.
- Any changes to persistence field names/naming conventions.

If you need to add or update files, keep diffs small and run `mvn test` before opening a PR.

---
If anything here is unclear or you want a different level of detail (examples, code snippets, or assumptions about upstreams), tell me which area to expand. 
