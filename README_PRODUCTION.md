# Production Architecture: Scalable & Resilient


A distributed, queue-based architecture that keeps the WhatsApp driver assistant independent from the logistics SaaS while guaranteeing fast acknowledgements, async processing, and operational guardrails. This mirrors the structure of the MVP README but covers every production-only concern.

---

## Architecture Overview

**Visual Diagram:** `whatsapp-driver-architecture.excalidraw`

![MVP Flow](./PRODUCTION-FLOW.jpg)

**Flow (26 steps grouped into 7 phases):**
1. **Message reception & authentication (1-8):** Driver sends WhatsApp message to Meta Business API. Webhook forwards to load balancer (ALB/NGINX) that routes to Fastify instances. Verify signatures, authenticate driver, record message, return 200 OK immediately.
2. **Message queuing (9-10):** Webhook enqueues to RabbitMQ with durable queues and DLQs, decoupling ingress from processing.
3. **Context & preparation (11-14):** Workers pull from queue, retrieve context from Redis, authenticate against PostgreSQL, enforce rate limits, run input guardrails.
4. **AI processing (15-16):** OpenRouter (GPT-5.1 mini) performs intent classification via function calling.
5. **Data retrieval (17-19):** Worker fetches data from SaaS API through circuit breaker with fallback to cached data.
6. **Response generation (20-23):** Format for WhatsApp, cache in Redis, store in PostgreSQL, run output guardrails.
7. **Message delivery (24-26):** Delivery worker sends via Meta API, records status, updates monitoring.

---

## Design Decisions & Rationale

### 1. Meta WhatsApp Business API
**Decision:** Use official Meta WhatsApp Business API directly.
**Why:** Direct access to Meta's official platform, full API control, production-grade reliability, and compliance with WhatsApp business policies.
**Trade-offs:** Requires Meta Business verification, but provides official support and prevents ban risk. Evolution API is MVP-only for testing.

### 2. Managed Cloud Hosting
**Decision:** Deploy Fastify webhooks, workers, and delivery services on AWS ECS/Fargate or GCP Cloud Run behind ALB/Cloud Load Balancing.  
**Why:** Autoscaling, managed TLS, health checks, blue/green deployments, and built-in observability. Railway/Render are solid alternatives for teams that prefer an all-in-one PaaS.  
**Trade-offs:** Higher monthly spend than a single VPS but critical for uptime targets and compliance audits.

### 3. Fastify + TypeScript Service Layer
**Decision:** Keep Fastify from the MVP but move to a plugin-based TypeScript codebase.  
**Why:** Fastify offers top-tier performance, schema validation, and simple extension points. Staying with the same framework minimizes migration risk. TypeScript makes the growing codebase maintainable and catches integration bugs early.  
**Trade-offs:** Slightly more upfront effort than plain JavaScript but pays off in reliability and developer ergonomics.

### 4. RabbitMQ Message Backbone
**Decision:** Route every inbound and outbound message through RabbitMQ (Amazon MQ).  
**Why:** Guarantees message durability, supports consumer acknowledgements, retries with exponential backoff, priority queues, and DLQs for manual inspection. Twilio receives a fast ACK because processing continues asynchronously.  
**Alternatives:** AWS SQS (less flexible fan-out patterns) or Kafka (overkill, heavier ops). RabbitMQ keeps latency low and offers the exact semantics we need.

### 5. Redis for Context, Rate Limits, and Caching
**Decision:** Use Redis (ElastiCache) as the shared state layer.  
**Why:** Sub-millisecond lookups for conversation context, sliding-window rate limits, and a response cache that cuts OpenRouter usage by ~30% on top of prompt caching. TTLs make conversation expiry automatic.  
**Trade-offs:** Another managed service to operate, but the LLM cost savings and rate-limiting precision offset the overhead.

### 6. Circuit Breaker & Resilience Patterns
**Decision:** Wrap APP API calls with `@fastify/circuit-breaker` plugin.
**Why:** The logistics SaaS can exhibit slowdowns or outages. The breaker isolates that risk, fails fast, and allows cached fallbacks so queues do not pile up. This is the main difference between MVP and production.
**Configuration:** timeout threshold, error percentage threshold, reset timeout, fallback to Redis cache when open.

### 7. PostgreSQL (Audit & Compliance)
**Decision:** Keep PostgreSQL (RDS) as the source of truth for drivers, messages, responses, guardrail events, and audit logs.  
**Why:** Relational integrity, JSONB columns for flexible metadata, and built-in tooling for retention policies (active vs archive). Required for GDPR/SOC 2 evidence.  
**Trade-offs:** Needs indexing and partition strategy at higher volumes, but the operational story is well understood.

### 8. Guardrails Microservice
**Decision:** Move from the MVP’s n8n node to a dedicated FastAPI guardrails service.  
**Why:** Centralized policy updates, ability to scale separately from workers, richer NLP libraries, detailed logging, and fail-open vs fail-closed decisions per environment.  
**Responsibilities:** Input validation (prompt injection, authorization, NSFW) and output sanitization (PII masking, hallucination detection, scope enforcement).

### 9. Load Balancer & Worker Pool
**Decision:** Use ALB/NGINX for ingress and run separate worker pools (message workers, delivery workers).  
**Why:** Load balancer enforces health checks and zero-downtime deploys; worker pools let us scale compute-intensive tasks independently. Queue depth metrics trigger auto-scaling policies (e.g., add worker when backlog >100 messages).  
**Trade-offs:** Slightly more infrastructure to monitor but enables predictable throughput as driver counts grow.

---

## AI Integration Strategy

**See [Main README - AI Integration](./README.md#ai-integration-openrouter-with-gpt-51) for shared details.**

**Production-Specific:**
- **Model:** OpenRouter GPT-5.1 for intent classification and response generation
- **Prompt caching:** Automatic caching reduces input costs (system prompt + functions)
- **Provider routing:** Environment flag toggles single-provider (caching) vs multi-provider (failover)
- **Redis response cache:** Caches repeated queries (intent + parameters hash)
- **Worker flow:** Assemble context → guardrails → LLM → API fetch (with breaker) → format → deliver

---

## Guardrails for AI Safety

**See [Main README - Guardrails](./README.md#guardrails-for-ai-safety) for purpose and protections.**

**Production Implementation:**
- **Dedicated FastAPI service** (2 instances with auto-scaling)
- **Input:** Prompt injection, authorization, NSFW, business scope
- **Output:** PII stripping, hallucination detection, data leak prevention
- **Operations:** Severity ladder, audit logging, alerting on violations

---

## Automation & Prototyping

- **n8n workflow:** 39 nodes mirroring production flow for testing new intents and policies
- **Why n8n:** Self-hosted, native Redis/RabbitMQ support, built-in guardrails, no per-step billing
- **Path:** Prototype in n8n → translate to Fastify plugins → deploy with same infrastructure


## Summary

**Production Architecture Prioritizes:**
1. **Reliability:** Official Meta WhatsApp API, load balancer, queues, circuit breaker, guardrails
2. **Scalability:** Horizontal scaling based on queue depth
3. **Compliance:** Audit trail, GDPR retention, guardrail logging
4. **Cost Efficiency:** Prompt + Redis caching reduces LLM costs
5. **Maintainability:** Fastify + TypeScript for team productivity

**Best For:** 50+ drivers, regulated industries, SLA requirements, high uptime needs.

---

## Related Documentation

- **[README.md](./README.md)** – Overview and architecture comparison
- **[README_MVP.md](./README_MVP.md)** – MVP architecture
- **[whatsapp-driver-architecture.excalidraw](./whatsapp-driver-architecture.excalidraw)** – Production diagram

