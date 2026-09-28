# VantageEngine

White-label e-commerce engine in which the site identity, visual theme, content and
optional modules are meant to be configured at runtime instead of hard-coded per client.

> **Status: planning.** This repository does not contain application code yet. The
> architecture and roadmap are being defined; nothing described below is implemented.

## Goals

- Identity, theme (design tokens) and page content loaded from the database, so a new
  client does not require code changes or a redeploy.
- Feature flags per tenant, with dependencies between modules, instead of environment
  variables that require restarts.
- A modular monolith with explicit module boundaries.
- A generic commerce domain (products, bundles, orders, notification channels) that is
  not tied to a single business.

## Planned stack

These are design decisions, not verified dependencies — there is no build file in the
repository yet.

| Area | Planned technology |
|---|---|
| Backend | Java 21, Spring Boot, Spring Modulith, Spring Security, Flyway |
| Database | MySQL 8.4 |
| Frontend | Angular (standalone components, Signals, lazy loading), CSS custom properties |
| Deployment | Docker, Docker Compose |

```mermaid
flowchart LR
    SPA[Angular SPA] -->|config-first bootstrap| API[Spring Boot API]
    API --> PLATFORM[platform module<br/>identity, theme, content, flags]
    API --> MODULES[optional business modules]
    PLATFORM --> DB[(MySQL)]
    MODULES --> DB
```

## Background

VantageEngine is a redesign of an earlier, client-specific storefront built by the same
author. That project is private; a high-level case study is published in the
[portfolio repository](https://github.com/FerS00/portfolio). VantageEngine is intended to
be written from scratch; no code is copied from the earlier project.

## License

Source code is **source-available** under the
[PolyForm Noncommercial License 1.0.0](LICENSE). This is not an OSI-approved open source
license.

> Source code is available under the PolyForm Noncommercial License 1.0.0. Commercial use
> requires a separate license from the copyright holder.

See [COMMERCIAL-LICENSE.md](COMMERCIAL-LICENSE.md) for commercial use.

Required Notice: Copyright (c) 2026 FerS00 (https://github.com/FerS00)

## Author

[FerS00](https://github.com/FerS00)
