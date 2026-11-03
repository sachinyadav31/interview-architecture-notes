# Architecture Notes: eProcure and Inclufy

Low-level notes on the big features of the products I worked on, written for interview prep.
Each file covers one feature: **what it does → architecture diagram → the exact flow → key classes/design choices → weak spots and how I'd improve them → interview questions with answers.**

> How to use these in an interview: say "**we** built / in **our** platform…". Explain the flow and the trade-offs. Bring up the weak spots yourself when asked "what would you improve?". That's where senior-sounding answers come from.

## The 60-second project pitch

> "I worked on **eProcure**, a multi-tenant B2B procurement SaaS: buyers and suppliers on one platform. The backend is **30+ Spring Boot microservices** behind a **Spring Cloud Gateway**, talking **synchronously over OpenFeign** and **asynchronously over Kafka** (AWS MSK). Each tenant gets its **own MySQL database per service**; we route connections at runtime with Hibernate multi-tenancy, based on a tenant qualifier the gateway puts in a header. On top of that: a custom approval workflow engine, RFI/RFP/RFQ eSourcing, procure-to-pay (requisition → PO → invoice → Stripe payment), DocuSign contracts, Elasticsearch product search and per-tenant Superset analytics. The frontends are Angular 18 portals. Later at Inclufy I extended it with new inventory/manufacturing/logistics services, a Django SSO layer that signs users into several tenants at once, a LangGraph AI copilot with 500+ tools over our APIs, and IQ Helix, a FastAPI + React learning platform."

## Index

| # | Feature | One-liner |
|---|---|---|
| 00 | [System overview](00-system-overview.md) | All services, tech stack, and how one request travels |
| 01 | [Multi-tenancy (database per tenant)](01-multi-tenancy.md) | ThreadLocal tenant context → Hibernate picks the tenant's Hikari pool |
| 02 | [API Gateway, JWT auth, IAM, ACL, MFA](02-gateway-auth-acl.md) | Gateway validates JWT, injects headers; each service checks API-level ACLs |
| 03 | [Kafka event-driven architecture](03-kafka-events.md) | Tenant-specific topics, dynamic listeners, tenant headers via interceptors |
| 04 | [Procure-to-Pay (POR → PO → Invoice)](04-procure-to-pay.md) | Requisition, approval, PO, goods receipt, invoice, payment status |
| 05 | [Workflow / approval engine](05-workflow-engine.md) | Home-grown serial/parallel approvals; results go out as Kafka events |
| 06 | [eSourcing: RFI / RFP / RFQ](06-esourcing.md) | Sourcing events, supplier responses, clarifications, auto-close by scheduler |
| 07 | [Product search with Elasticsearch](07-elasticsearch-search.md) | Kafka → stored proc builds JSON → bulk index into `{tenant}_{index}` |
| 08 | [Payments with Stripe](08-payments-stripe.md) | Checkout Session, redirect, status events; subscription webhooks |
| 09 | [Contracts and DocuSign e-signing](09-contracts-docusign.md) | Envelope creation, recipient view, completion hook → Kafka |
| 10 | [Notifications: email, SMS, SSE](10-notifications.md) | Kafka-driven templated email; Server-Sent Events for live UI |
| 11 | [Tenant onboarding + Superset analytics](11-tenant-onboarding.md) | Create DBs/users, DNS + nginx, auto-provision dashboards |
| 12 | [Schema migrations, audit, scheduler](12-liquibase-audit-scheduler.md) | Liquibase per tenant DB, Javers audit, DB-polled task scheduler |
| 13 | [CI/CD and deployment](13-cicd-deployment.md) | Jenkins → GitLab CI shared templates, Docker, ECR, compose deploys |
| 14 | [Inclufy SSO across tenants (Django)](14-inclufy-sso.md) | One login → tokens for every tenant you can reach |
| 15 | [AI copilot / chatbot (LangGraph)](15-ai-copilot.md) | Multi-provider LLM, 500+ tools calling our APIs with the user's token |
| 16 | [IQ Helix (FastAPI + React)](16-iq-helix.md) | Assessments, AI course generation, production hardening |
| 17 | [Angular frontend architecture](17-angular-frontend.md) | Portals, shared IAM UI library, interceptors, i18n |
| 18 | [Inventory, manufacturing, logistics, operations](18-supply-chain-services.md) | **Your services**: stock ledger + balances, BOM/production, shipments, risk scoring, generic CQRS base |
| 19 | [Supporting services](19-supporting-services.md) | Textract invoice OCR, UBL e-invoicing, budgets, Kafka Connect analytics, embedded Superset, Eureka |
| 20 | [Inclufy Django platform](20-inclufy-django-platform.md) | Django apps, APScheduler jobs, tenant provisioning step 2, nightly AI agents, automation rules |
| 21 | [Inclufy Ignite (React) + deployment](21-inclufy-ignite-react.md) | Your migration: two Axios clients, refresh queue, permissions, copilot registry, nginx + PM2 |
| 99 | [Generic questions + system design](99-general-questions.md) | Microservices, Kafka, Spring, JPA, and how to answer design rounds |
