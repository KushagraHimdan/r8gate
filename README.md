# R8Gate

A distributed rate limiter and API gateway, built on the MERN stack with Redis-backed atomic counters.

R8Gate sits in front of your APIs and controls how many requests a client can make in a given time window — protecting backend services from traffic spikes, abuse, and noisy-neighbor problems. It ships as both an importable Express middleware and a standalone gateway that proxies to a backend service.

## Why

Most rate limiters are either single-instance (break under horizontal scaling) or a managed black box (Kong, AWS API Gateway) you don't fully understand. R8Gate is a small, self-hosted, horizontally-correct rate limiter — built to be run, read, and reasoned about.

## Features

- Fixed window and token bucket rate-limiting algorithms
- Atomic, Redis-backed counters — correct even across multiple gateway instances
- Use as Express middleware (`app.use(r8gate(config))`) or as a standalone gateway proxy
- Per-client (API key) and per-route limit configuration, via MongoDB-backed tiers
- Standard `429` responses with `Retry-After` and rate-limit headers
- Live React dashboard showing allowed/denied traffic in real time

## Stack

React · Node.js/Express · MongoDB · Redis · Docker

## Status

🚧 In development. Docs for the full design and build plan live in [`docs/`](./docs).

## Quick Start

```bash
docker-compose up
```

_(Full setup instructions coming as the project is built out.)_

## Docs

- [Product Requirements](./docs/PRD.md)
- [Software Requirements Specification](./docs/SRS.md)
- [Architecture](./docs/architecture.md)
- [UI/UX Design](./docs/design.md)
- [Development Phases](./docs/phases.md)