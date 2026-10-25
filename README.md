# Gatekeeper: SLO-Aware Multi-Tenant Gateway

**An API gateway designed to explore adaptive traffic management, tenant fairness, and latency-aware request control.**

Gatekeeper is a Python-based API gateway built with FastAPI and Redis. It provides API-key authentication, request rate limiting, and request forwarding to backend services. Its planned adaptive extension will explore how a gateway can adjust traffic policies in response to backend latency, errors, and competing tenant demand while working toward defined Service Level Objectives (SLOs).

The project focuses on backend engineering, distributed systems concepts, reliability, and performance evaluation.

## Table of Contents

- [Overview](#overview)
- [Motivation](#motivation)
- [Architecture](#architecture)
- [Current Features](#current-features)
- [Planned Adaptive Features](#planned-adaptive-features)
- [Technology Stack](#technology-stack)
- [Getting Started](#getting-started)
- [API Usage](#api-usage)
- [Rate Limiting](#rate-limiting)
- [SLOs and Performance Evaluation](#slos-and-performance-evaluation)
- [Project Structure](#project-structure)
- [Development Roadmap](#development-roadmap)
- [Engineering Concepts](#engineering-concepts)
- [Future Improvements](#future-improvements)

## Overview

Modern backend systems often expose multiple services to clients. Without a centralized gateway, each service may need to implement its own authentication, traffic control, and request-handling policies.

Gatekeeper introduces a gateway layer between clients and backend services.

The gateway is responsible for:

- Authenticating incoming requests using API keys.
- Enforcing request rate limits using Redis.
- Forwarding authorized requests to backend services.
- Handling request failures and upstream timeouts.
- Providing a foundation for multi-tenant traffic management.
- Exploring latency-aware traffic policies and SLO-driven adaptation.

The long-term goal is to evolve Gatekeeper from a conventional API gateway into an adaptive traffic-management system that balances service reliability with fair resource allocation across tenants.

## Motivation

A fixed rate limit can protect a backend from excessive traffic, but it does not necessarily account for changing service conditions.

For example, a backend that responds quickly under normal traffic may become overloaded during a burst of requests. A fixed policy may continue admitting traffic even when latency and error rates begin increasing.

Gatekeeper aims to explore the following question:

**Can a gateway adapt its traffic policies to changing backend conditions while maintaining predictable latency and fair access for multiple tenants?**

The project will investigate this through controlled experiments, explicit performance targets, and comparisons between fixed and adaptive traffic policies.

## Architecture

### Current foundation

```text
                  Client
                    |
                    v
          +---------------------+
          | Gatekeeper Gateway  |
          |      FastAPI        |
          +---------------------+
             |       |       |
             |       |       |
             v       v       v
        API-Key    Redis   Request
        Auth       Rate    Forwarding
                   Limit       |
                               v
                    +-------------------+
                    | Products Service  |
                    +-------------------+
```

### Target adaptive architecture

```text
                  Multiple Clients
                 /       |       \
                v        v        v
             Tenant A Tenant B Tenant C
                 \       |       /
                  v      v      v
          +---------------------------+
          |     Gatekeeper Gateway    |
          |---------------------------|
          | Authentication            |
          | Tenant-Aware Rate Limits  |
          | Admission Control         |
          | Load Shedding             |
          | Circuit Breaker           |
          +---------------------------+
                 |              |
                 v              v
             +--------+   +-------------+
             | Redis  |   | Backend Pool|
             +--------+   +-------------+
                                |
                                v
                       Latency and Errors
                                |
                                v
                     +--------------------+
                     | Adaptive Controller|
                     | SLO Feedback Loop  |
                     +--------------------+
                                |
                                v
                       Updated Traffic Policy

                 Prometheus --> Grafana
                    Metrics and Dashboards
```

The second diagram represents the target architecture. Features shown there will be marked as implemented only after development and testing.

## Current Features

### 1. API Gateway

- Built using FastAPI.
- Provides a centralized entry point for backend requests.
- Forwards requests to a separate Products Service.
- Separates gateway responsibilities from business logic.

### 2. API-Key Authentication

- Protects configured endpoints using an API-key header.
- Rejects requests that do not provide the expected credentials.
- Separates access control from backend business logic.

Example header:

```http
x-api-key: YOUR_API_KEY
```

The development key used for local testing should be replaced with environment-based secrets before deployment.

### 3. Redis-Backed Rate Limiting

- Uses Redis to track request activity.
- Enforces a configured request limit over a time window.
- Returns HTTP `429 Too Many Requests` when the configured limit is exceeded.
- Uses shared Redis state as the foundation for coordinating limits across gateway instances.

The current local test configuration allows five requests within the configured window and rejects the next request once the limit is reached.

### 4. Backend Request Forwarding

- Routes authorized requests to the Products Service.
- Separates gateway logic from backend service logic.
- Provides a foundation for adding additional services and routing policies.

### 5. Error Handling

The gateway implementation includes handling for relevant request failures, such as rate-limit violations and upstream service errors or timeouts.

## Planned Adaptive Features

The following capabilities form the next development phase.

### 1. SLO-Aware Adaptive Rate Limiting

Develop a feedback controller that adjusts tenant request limits using observed backend latency and error signals.

The controller will operate within configurable minimum and maximum limits, with cooldown periods and hysteresis to reduce unstable policy changes.

**Initial engineering target:** p99 latency below 200 ms under a clearly specified test workload.

This is a target, not a measured result.

### 2. Multi-Tenant Admission Control

Introduce tenant-specific traffic policies so that one high-volume tenant cannot consume all available backend capacity.

Planned capabilities:

- Tenant identification and configurable quotas.
- Per-tenant request limits and priority policies.
- Admission control during overload.
- Load shedding when the backend cannot safely accept more traffic.
- Fairness evaluation under competing workloads.

### 3. Circuit Breaker and Resilience

Implement a circuit breaker to temporarily stop sending requests to an unhealthy backend.

The design will explore the standard circuit-breaker states:

- **Closed:** Requests are allowed through.
- **Open:** Requests are rejected or failed fast while the backend is unhealthy.
- **Half-open:** A limited number of test requests determine whether recovery has occurred.

Timeouts and bounded retries will be designed to avoid amplifying backend overload.

### 4. Observability with Prometheus and Grafana

Instrument the gateway to expose operational metrics, including:

- Request throughput.
- Request latency and p95/p99 latency.
- HTTP status-code distribution.
- Rate-limited requests.
- Backend errors and timeouts.
- Per-tenant traffic and policy decisions.
- Adaptive controller decisions.

Prometheus will collect metrics, while Grafana will be used to visualize traffic, reliability, and latency trends.

### 5. Reproducible Benchmarking

Build a repeatable test workload to compare traffic-control strategies under normal traffic, bursts, competing tenants, and backend failures.

The comparison will include:

- Fixed-window rate limiting.
- Token-bucket rate limiting.
- Adaptive rate limiting.

The experiments will measure throughput, p99 latency, error rates, and tenant fairness. Performance improvements will be reported only when supported by measured baseline and experimental results.

## Technology Stack

| Technology | Purpose |
|---|---|
| Python | Core implementation |
| FastAPI | Gateway API and request handling |
| Redis | Shared rate-limit state |
| HTTPX | HTTP communication with backend services |
| Docker | Running infrastructure services consistently |
| Prometheus | Metrics collection (planned integration) |
| Grafana | Monitoring dashboards (planned integration) |
| Pytest | Automated testing |
| Locust or k6 | Load testing and benchmarking (planned) |
| GitHub Actions | Continuous integration (planned) |

## Getting Started

### Prerequisites

Install the following:

- Python 3.10 or compatible version.
- Docker Desktop.
- Git.
- A running Redis instance.
- The Gatekeeper source code and configured Products Service.

### 1. Clone the repository

```bash
git clone https://github.com/Irasri007/Gatekeeper.git
cd Gatekeeper
```

Replace the repository URL if the project is hosted under a different URL.

### 2. Create a virtual environment

On Windows:

```powershell
py -3.10 -m venv .venv
.venv\Scripts\activate
```

Install the dependencies:

```bash
python -m pip install --upgrade pip
pip install fastapi "uvicorn[standard]" redis httpx pytest
```

If the repository contains a `requirements.txt`, prefer installing the project's pinned dependencies:

```bash
pip install -r requirements.txt
```

### 3. Start Redis with Docker

If your existing Redis container is already running, reuse it.

Otherwise, create a Redis container:

```bash
docker run -d --name gatekeeper-redis -p 6379:6379 redis:7-alpine
```

Verify that it is running:

```bash
docker ps
```

Check Redis connectivity:

```bash
docker exec gatekeeper-redis redis-cli ping
```

Expected response:

```text
PONG
```

If the container already exists but is stopped, start it using:

```bash
docker start gatekeeper-redis
```

### 4. Configure environment variables

Configure the gateway's Redis connection, API key, and backend service URL according to the variable names used in your implementation.

For example, the intended configuration may look like:

```text
REDIS_URL=redis://localhost:6379/0
API_KEY=replace-with-a-local-development-key
PRODUCTS_SERVICE_URL=http://127.0.0.1:8001
```

These are example configuration values. Match the variable names to the actual code before using this file as `.env`.

Do not commit real secrets to version control.

### 5. Start the Products Service

Start the existing Products Service using its configured development command. Ensure it listens on the address configured in the gateway, currently expected to be:

```text
http://127.0.0.1:8001
```

### 6. Start the gateway

From the project directory, run:

```bash
python -m uvicorn gateway:app --reload --port 8000
```

The gateway will be available at:

```text
http://127.0.0.1:8000
```

If the application exposes FastAPI's interactive documentation, visit:

```text
http://127.0.0.1:8000/docs
```

## API Usage

The examples below use the existing local development endpoint and API-key configuration.

### 1. Access a protected product endpoint

On Windows Command Prompt:

```cmd
curl -i -H "x-api-key: gatekeeper-dev-key-123" http://127.0.0.1:8000/products/1
```

Example response:

```json
{
  "id": 1,
  "name": "Laptop",
  "price": 55000
}
```

The example response assumes the Products Service contains this sample product.

### 2. Test authentication

Send a request without an API key:

```cmd
curl -i http://127.0.0.1:8000/products/1
```

The gateway should reject the request if authentication is enforced on this endpoint.

### 3. Test rate limiting

Run the following commands repeatedly:

```cmd
curl -i -H "x-api-key: gatekeeper-dev-key-123" http://127.0.0.1:8000/products/1
```

With the current five-request limit, requests within the configured window should be allowed until the limit is reached. Further requests should receive HTTP `429` until the applicable window or counter resets.

The exact behavior depends on the configured rate-limiting algorithm and window.

## Rate Limiting

Rate limiting controls how many requests a client can make within a specified period.

For example, a policy configured for five requests per minute permits up to five requests during the applicable window. Requests exceeding the limit receive HTTP `429 Too Many Requests`.

Redis provides shared state for request counters or timestamps. This is important when multiple gateway instances must enforce a coordinated limit.

The adaptive extension will build on this foundation by adjusting tenant-specific limits in response to observed system conditions.

## SLOs and Performance Evaluation

A Service Level Objective (SLO) defines a measurable reliability or performance goal.

For Gatekeeper, the initial performance target is:

| Metric | Initial target or evaluation |
|---|---|
| p99 latency | Target below 200 ms under a defined workload |
| Tenant fairness | Target Jain's fairness index of at least 0.90 |
| Throughput | Measure requests processed per second |
| Error rate | Measure failed and rejected requests |
| Recovery time | Measure recovery after backend faults |
| Policy stability | Measure unnecessary or oscillating limit changes |

The p99 latency target means that 99% of measured requests should complete within the specified threshold under the defined measurement conditions.

Jain's fairness index can be calculated as:

\[
J(x_1,\ldots,x_n)=
\frac{(\sum_{i=1}^{n}x_i)^2}
{n\sum_{i=1}^{n}x_i^2}
\]

Here, \(x_i\) represents the measured service allocation or throughput for tenant \(i\). The index ranges from \(1/n\) to \(1\) for nonnegative allocations with a positive total: a value closer to 1 indicates more equal allocation.

Fairness must be evaluated against the chosen policy, tenant priorities, and workload. Equal throughput is not always the correct outcome when tenants have different entitlements.

### Benchmarking methodology

The planned experiments will:

1. Establish a baseline using fixed rate limits.
2. Generate workloads with different tenant request rates.
3. Introduce burst traffic and backend latency.
4. Inject controlled backend failures.
5. Compare fixed-window, token-bucket, and adaptive policies.
6. Record p95/p99 latency, throughput, errors, and fairness.
7. Repeat experiments under consistent conditions.
8. Document the environment, configuration, and results.

No performance improvement or achieved SLO should be reported until the relevant experiments have been completed.

## Project Structure

The repository's current core includes the gateway implementation and its supporting configuration. A possible target structure as the project grows is:

```text
Gatekeeper/
├── gateway.py
├── auth.py
├── rate_limiter.py
├── adaptive_controller.py
├── circuit_breaker.py
├── routing.py
├── config.py
├── tests/
│   ├── test_auth.py
│   ├── test_rate_limiter.py
│   ├── test_gateway.py
│   ├── test_adaptive_controller.py
│   └── test_circuit_breaker.py
├── benchmarks/
│   ├── workloads/
│   └── results/
├── monitoring/
│   ├── prometheus.yml
│   └── grafana/
├── docker-compose.yml
├── requirements.txt
├── .env.example
├── .gitignore
└── README.md
```

This is a proposed organization, not a claim that every listed file already exists.

## Development Roadmap

- [x] Build the FastAPI gateway foundation.
- [x] Add API-key authentication.
- [x] Integrate Redis-backed request rate limiting.
- [x] Validate rate-limit rejection behavior.
- [x] Integrate the Products Service endpoint.
- [ ] Add automated tests for authentication and rate limiting.
- [ ] Introduce explicit tenant identity and tenant-specific policies.
- [ ] Implement the SLO-aware adaptive controller.
- [ ] Add fairness-aware admission control and load shedding.
- [ ] Implement circuit breaking and resilience tests.
- [ ] Integrate Prometheus metrics and Grafana dashboards.
- [ ] Benchmark fixed-window, token-bucket, and adaptive strategies.
- [ ] Evaluate latency, fairness, and recovery under fault injection.
- [ ] Add reproducible deployment and continuous integration.

## Engineering Concepts

Gatekeeper provides a practical environment for exploring:

- API gateway and reverse-proxy design.
- Authentication and request authorization.
- Distributed rate limiting with Redis.
- Multi-tenant resource allocation.
- Feedback control and SLO-driven system behavior.
- Tail-latency measurement and performance testing.
- Circuit breakers and overload protection.
- Observability and fault injection.
- Reproducible experiments and evidence-based optimization.

## Future Improvements

Potential extensions include:

- Health-aware load balancing across backend replicas.
- Version-aware routing and canary releases.
- Tenant key rotation and configurable quota policies.
- Distributed tracing and request correlation IDs.
- Concurrency and failure testing across multiple gateway instances.
- Automated performance regression checks in CI.

## Project Status

**Status:** Active development.

Gatekeeper currently provides the foundational gateway, authentication, Redis-backed rate limiting, and backend integration. Adaptive rate control, tenant fairness, the complete monitoring stack, and comparative performance benchmarks are part of the planned evolution and will be documented as implemented after verification.

The objective is to develop a reproducible backend systems project that demonstrates not only how to build a gateway, but also how to evaluate its reliability, fairness, and behavior under changing traffic conditions.
