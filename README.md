# Zorro AI Voice

**Multi-tenant voice AI platform for restaurants**

A production-grade prototype that lets restaurants deploy their own AI phone agent. Customers call, talk to an LLM-powered host, and everything is logged, metered, and isolated per tenant.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INCOMING CALL                                │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        TWILIO VOICE                                  │
│              Secure webhooks / TLS by default                        │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    GCP CLOUD RUN                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                     FastAPI Backend                            │  │
│  │                                                                │  │
│  │  • Tenant resolution from incoming phone number                │  │
│  │  • Auth + RBAC middleware                                      │  │
│  │  • Circuit breaker for upstream services                       │  │
│  │  • Structured logging + audit trails                           │  │
│  │  • HTTPS endpoints (managed by Cloud Run)                      │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
             ┌──────────┐  ┌──────────┐  ┌──────────┐
             │   LLM    │  │   TTS    │  │ Postgres │
             │ (Agent)  │  │(Eleven   │  │(Cloud SQL)│
             │          │  │  Labs)   │  │ encrypted │
             └──────────┘  └──────────┘  └──────────┘
                    │             │             │
                    └─────────────┼─────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     CALLER HEARS RESPONSE                            │
│              Natural language, realistic voice                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology | Why This Choice |
|-------|------------|-----------------|
| **Telephony** | Twilio Programmable Voice | Industry standard, secure webhooks, TLS by default |
| **Backend** | FastAPI (Python) | Async-native, type hints, fast iteration |
| **Compute** | GCP Cloud Run | Scales to zero, HTTPS managed, no infra overhead |
| **Database** | Cloud SQL (PostgreSQL) | Managed Postgres, encryption-at-rest, multi-tenant ready |
| **LLM** | OpenAI / Anthropic | Swappable provider layer for agent logic |
| **TTS** | ElevenLabs | Realistic voice output, low latency streaming |
| **Frontend** | Next.js 15 (React + TypeScript) | Modern stack, good auth patterns |
| **Billing** | Stripe (scaffolded) | Industry standard for SaaS metering |

---

## What's Running Today

### Multi-Tenant Data Model

The system is built tenant-first. Every restaurant group is isolated at the data layer.

```
accounts
    └── organizations (tenant boundary)
            └── locations (individual restaurants)
                    └── phone_numbers (Twilio numbers)
                            └── call_logs (per-call records)
```

Each API request resolves the tenant from the authenticated user or incoming phone number. No cross-tenant data leakage by design.

### Role-Based Access Control

Basic RBAC is implemented at the organization level:

**Owner**: Full access, billing, user management  
**Admin**: Location management, view all calls  
**Staff**: View calls for assigned locations only

Permissions are checked in middleware before any route handler executes.

### Call Handling Flow

1. Twilio receives incoming call to restaurant's number
2. Webhook hits Cloud Run with caller info + Twilio session
3. Backend resolves which tenant owns that phone number
4. LLM agent is initialized with restaurant-specific context (hours, menu, FAQs)
5. Caller speaks → transcribed → LLM generates response → TTS → caller hears reply
6. Full call record saved: duration, transcript, outcome, metadata

### Ops, Safety & Early Security Foundations

This is "sensible foundations for a prototype" — not a full enterprise security program yet, but the bones are right.

**Reliability**  
Circuit breaker pattern for upstream services (LLM, TTS, Twilio). Graceful fallback responses when services timeout or fail. Structured logging to trace each step of call handling.

**Tenant Isolation**  
Enforced at both DB and API layers. Org-scoped data access on every request. No cross-tenant queries possible by design.

**Audit & Observability**  
Basic audit-style logging (who did what, when). Call records with full metadata for debugging and analytics.

**Managed Security (via stack)**  
Cloud Run provides HTTPS endpoints with managed TLS. Cloud SQL handles encryption-at-rest. Twilio delivers secure webhooks with TLS by default.

### Dashboard (Next.js)

Restaurant users can log in with tenant-aware sessions, view their organizations and locations, see phone numbers assigned to each location, and browse call history with metadata including duration, time, and outcome.

---

## Designed But Not Production-Ready

These exist in code and architecture but need polish before launch:

| Feature | Status | What's Left |
|---------|--------|-------------|
| **Stripe billing** | Schema + test mode | Self-serve UI, pricing page, webhooks |
| **Analytics dashboard** | Queries written | Frontend visualization, ROI metrics |
| **A/B agent testing** | Architecture ready | UI for managing experiments |
| **OpenTable integration** | Flow designed | End-to-end wiring |
| **SMS follow-ups** | Twilio ready | Trigger logic, templates |

### Not Yet Implemented

Being upfront about what's not here yet:

No SOC 2, ISO, HIPAA, or PCI certifications. No formal GDPR-style data rights or deletion flows. PII redaction and retention policies are on the roadmap, not live.

This is a prototype with real infrastructure, not a compliance-certified enterprise product. That's the next phase.

---

## Design Decisions

### Why multi-tenant from day one?

I've run a multi-tenant SaaS (killfeed.dev) for 6 years. Retrofitting tenant isolation is painful. Building it in from the start means clear data boundaries, straightforward per-tenant billing, easier security reasoning, and the ability to onboard new restaurants without code changes.

### Why circuit breakers?

Voice is unforgiving. A 3-second hang during a web request is annoying. A 3-second silence during a phone call feels broken. The system needs to fail gracefully or not fail at all.

### Why FastAPI over Node?

For this use case, Python's async ecosystem and the existing LLM tooling made iteration faster. Type hints plus Pydantic give me confidence when refactoring.

### Why be upfront about security gaps?

Because that's what a senior engineer does. Knowing what you haven't built yet is as important as knowing what you have. The foundations are solid; the enterprise hardening comes when there's revenue to justify it.

---

## What This Demonstrates

This isn't a weekend hack. It's a production-minded system built by someone who understands the difference between "works on my laptop" and "ready for customers."

**Tenant isolation**: Data model designed for many customers from the start  
**Failure handling**: Circuit breakers, structured logging, graceful degradation  
**Real infrastructure**: Cloud Run, Cloud SQL, Twilio — not localhost demos  
**Ops thinking**: Logging, RBAC, audit trails for call records  
**Honest scoping**: Clear about what's built vs. what's roadmap  
**Security awareness**: Sensible foundations now, compliance when it matters

---

## About the Builder

I'm a self-taught builder who's been shipping production systems for over a decade. I built this system in roughly 15 days, working a few hours each evening after my day job. I use AI-assisted development to move fast without sacrificing architecture quality.

**11 years as Network Specialist at TELUS** — infrastructure at scale  
**6 years running killfeed.dev** — profitable B2B SaaS, 4,100+ servers, 3.3M player profiles, 300K log rows/day  
**Current role**: Technical Account Manager doing everything from customer success to sensor encapsulation in a lab

I build things that work, then make them better.

---

*Last updated: November 2025*
