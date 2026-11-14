# DevHub

DevHub helps developers manage and monitor their APIs and backend infrastructure from one place.

Think of it as a simplified combination of Postman, an API monitoring service, and a webhook management platform.

## Why DevHub?

Imagine you're building an application with:

- A mobile client
- A backend API
- A payment integration
- An AI service
- Several external integrations

You need to manage access, monitor requests, process events, and track failures. DevHub provides the tools to do that.

## What can a developer do?

### 1. Manage projects and API keys

Create projects, separate development and production environments, generate API keys, and control who can access them.

### 2. Monitor API usage

See how many requests your API receives, which endpoints are slow, how many requests fail, and which users consume the most resources.

### 3. Manage webhooks

Register external URLs that should receive events — for example, notify an ecommerce application when a payment succeeds. Automatically retry failed deliveries.

### 4. Process background jobs

Schedule tasks, execute long-running operations, track their status, and retry failed jobs.

### 5. Monitor activity in real time

Watch incoming requests, errors, job execution, and webhook delivery through live updates.

## Example: How it works

Suppose a developer integrates DevHub into an online store:

1. The store creates a DevHub project.
2. It generates an API key.
3. The store sends events to DevHub when an order is created.
4. DevHub records the event.
5. A background worker processes it.
6. DevHub sends a webhook to the store's payment or delivery integration.
7. If the destination is unavailable, DevHub retries delivery.
8. The developer can inspect the delivery history and errors.

## Docs

- [Architecture](docs/architecture.md) — modular monolith, hexagonal layers, OOP + SOLID
- [Development platform](docs/development-platform.md) — concepts, capabilities, workflow
