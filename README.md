# Driver Communication System

This repository contains a comprehensive system design created for a technical challenge to architect a WhatsApp-based communication system that allows delivery drivers to interact with order management systems through natural conversation.

**Note:** This is a **design repository** containing architecture documentation, visual diagrams. No code implementation is included.

---

## Challenge Overview

Design and architect a WhatsApp-based communication system that allows delivery drivers to:
- Get daily order summaries
- Ask questions about deliveries
- Confirm order details
- Interact naturally using conversational AI

---

## Two Architecture Approaches

This project provides **two distinct architectural approaches** to suit different needs:

### MVP Architecture (Simple & Fast)
**Perfect for proof of concept, rapid validation**


**[See MVP Architecture →](./README_MVP.md)**

### Production Architecture (Scalable & Resilient)
**Perfect for high availability, enterprise needs**



**[See Production Architecture →](./README_PRODUCTION.md)**

---

## Architecture Comparison

| Factor | MVP | Production |
|--------|-----|------------|
| **Queuing** | ❌ Direct | ✅ RabbitMQ |
| **Caching** | ❌ None | ✅ Redis |
| **Workers** | ❌ Single process | ✅ 3-5 workers |
| **Load Balancer** | ❌ No | ✅ NGINX/ALB |
| **Circuit Breaker** | ❌ No | ✅ Yes |
| **Auto-scaling** | ❌ Manual | ✅ Automatic |
| **Best For** | POC, Testing | Production, Scale |

---

## Visual Diagrams

Open these Excalidraw files at [excalidraw.com](https://excalidraw.com) to view the interactive architecture diagrams:

- **[whatsapp-driver-architecture-MVP.excalidraw](./whatsapp-driver-architecture-MVP.excalidraw)** - Simplified MVP flow
- **[whatsapp-driver-architecture.excalidraw](./whatsapp-driver-architecture.excalidraw)** - Complete production architecture (26-step flow)

---

## Decision Guide

### Choose MVP if:
- You're validating the concept
- Budget is tight (< $50/month)
- You have < 50 drivers
- You can tolerate occasional downtime
- You want to launch fast

### Choose Production if:
- You have 50+ drivers (or will soon)
- You need high uptime
- You're handling critical operations
- You have compliance requirements (GDPR, SOC 2)
- You have budget
- You have DevOps expertise

---

## Features

### Authentication & Security
- Phone number-based driver verification
- Status validation (active/inactive/suspended)
- Permission-based access control
- Complete audit logging
- Rate limiting (per-driver and global)

### AI-Powered Conversations
- OpenRouter with GPT-5.1-mini for natural language understanding
- Prompt caching for cost optimization (40-64% savings)
- Intent classification (get_orders, order_details, help, etc.)
- Function calling for structured API interactions
- Context-aware follow-up questions

### Message Persistence
- All messages saved to PostgreSQL for compliance
- Audit trail for all driver interactions

### Resilience (Production)
- Circuit breaker pattern for external APIs
- Message queuing with RabbitMQ
- Dead letter queues for failed messages
- Redis caching for performance
- Horizontal scaling of workers

---

## Example Conversation

**Driver:** "What are my orders today?"

**System Flow:**
1. Message received via WhatsApp
2. Driver authenticated against database
3. Message saved for audit trail
4. OpenRouter (gpt-5-mini) classifies intent
5. Orders fetched from APP API
6. Natural response generated
7. Sent back via WhatsApp

**Response:**
```
📦 You have 3 orders today, Johan!

🚚 Orders:
1. #12345 - Tech Corp Inc (12:00 PM)
   📍 123 Main St, Suite 400

2. #12346 - Westside Restaurant (2:30 PM)
   📍 456 Oak Ave

3. #12347 - Airport Terminal B (4:00 PM)
   📍 Terminal B, Gate 5

Would you like details on any order? Type from 1 to 3 to choose any order.
```

---

## AI Integration: OpenRouter with gpt-5-mini

### Why OpenRouter
- **Unified API:** Single API key for multiple LLM providers
- **Prompt Caching:** Automatic caching
- **Provider Routing:** Intelligent load balancing for uptime
- **Cost Dashboard:** Centralized spending tracking
- **Flexibility:** Easy model switching without code changes

### Why gpt-5-mini
- **Adaptive Reasoning:** Dynamic computation for complex tasks
- **Function Calling:** Best-in-class for structured API interactions
- **Multilingual:** Excellent performance across languages
- **Cost:** $0.25/M input tokens $2/M output tokens via OpenRouter

### Prompt Caching Strategy
OpenRouter automatically caches system prompts and function definitions, reducing input costs zero infrastructure. MVP uses single provider routing (OpenAI only) for consistent caching. Production adds Redis response caching for LLM cost reduction.

---

## Guardrails for AI Safety

### Purpose
Security layer that filters AI inputs and outputs to prevent malicious exploitation, data leaks, and unauthorized access.

### Key Protections

**Input Guardrails:**
- Prompt injection detection
- NSFW content blocking
- Business scope enforcement

**Output Guardrails:**
- PII sanitization
- Hallucination prevention
- Sensitive data masking

### Implementation
- **MVP:** n8n built-in Guardrails node, zero additional cost
- **Production:** Dedicated service with custom policies

---

## License

MIT License

---