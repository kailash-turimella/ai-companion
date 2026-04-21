# AI Companion

A full-stack SaaS platform for creating and chatting with custom AI-powered companions — built to demonstrate end-to-end skills across LLM integration, vector memory, real-time streaming, and subscription infrastructure.

![Next.js](https://img.shields.io/badge/Next.js-14-black?style=flat-square&logo=nextdotjs)
![TypeScript](https://img.shields.io/badge/TypeScript-5-blue?style=flat-square&logo=typescript)
![LangChain](https://img.shields.io/badge/LangChain-latest-green?style=flat-square)
![Prisma](https://img.shields.io/badge/Prisma-ORM-2D3748?style=flat-square&logo=prisma)

---

## What It Does

Users can browse a library of AI characters, each with a distinct personality and backstory, and chat with them in real-time. Pro subscribers can create their own companions — giving them a name, avatar, instructions, and a seed conversation — then share them with others.

The interesting part isn't the chat UI. It's the memory system underneath it.

---

## The Memory Architecture

Most chat apps pass the last N messages as context. This app goes further with a **two-layer memory system**:

```
User Message
     │
     ├─► Redis (Upstash)        →  Last 30 messages, ordered by timestamp
     │                               "What was said recently?"
     │
     └─► Pinecone (Vector DB)   →  OpenAI embeddings + similarity search
                                    "What past messages are semantically relevant?"
```

Both results feed into the prompt alongside the companion's system instructions, giving the LLM both **recency** and **relevance** — so a companion can recall something said 200 messages ago if it's topically related to the current message.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 14 (App Router), TypeScript |
| UI | Tailwind CSS, shadcn/ui, Radix UI |
| LLM Inference | Replicate — Llama 2 13B Chat |
| LLM Orchestration | LangChain |
| Vector Memory | Pinecone + OpenAI Embeddings |
| Session Memory | Upstash Redis |
| Database | PostgreSQL via Prisma ORM |
| Auth | Clerk |
| Payments | Stripe (subscriptions + webhooks) |
| Image Hosting | Cloudinary |
| Rate Limiting | Upstash sliding window |
| State Management | Zustand |
| Forms | React Hook Form + Zod |

---

## Key Technical Decisions

**Streaming responses.** The chat endpoint uses `LangChainStream` and `StreamingTextResponse` from Vercel's `ai` package. The frontend renders tokens as they arrive — no waiting for the full model response.

**Dual-layer memory.** Redis handles ordered history (fast reads, last 30 messages). Pinecone handles semantic retrieval (find relevant context from anywhere in the conversation). Running both gives the model the most useful context window possible.

**Server-side subscription gating.** Companion creation is checked server-side on every request — not just client-side UI hiding. Stripe webhooks (`checkout.session.completed`, `invoice.payment_succeeded`) write subscription state to PostgreSQL, which the gating logic reads.

**Rate limiting at the API boundary.** Upstash sliding window (10 requests / 10 seconds per user) is applied in the chat route before any LLM calls, keeping inference costs predictable.

**Prompt engineering.** Each LLM call assembles context in a specific order: companion personality/instructions → semantically relevant history → recent chat history → current message. This keeps the model in character while being aware of the full conversation.

---

## Architecture Overview

```
Browser (React)
    │
    ├── shadcn/ui components
    ├── Zustand (modal state)
    └── useCompletion (streaming hook)
         │
         ▼
Next.js API Routes
    ├── /api/chat/[chatId]     ← main chat endpoint
    ├── /api/companion         ← create/update companions (Pro only)
    ├── /api/stripe            ← checkout & billing portal
    └── /api/webhook           ← Stripe events
         │
    ┌────┴────────────────────────────────┐
    │            Business Logic           │
    │  MemoryManager (Redis + Pinecone)   │
    │  Subscription check (Prisma)        │
    │  Rate limiter (Upstash)             │
    └────┬────────────────────────────────┘
         │
    ┌────┴────────────────────────────────┐
    │         External Services           │
    │  Replicate  →  Llama 2 inference    │
    │  Pinecone   →  vector memory        │
    │  Upstash    →  Redis history        │
    │  Clerk      →  authentication       │
    │  Stripe     →  billing              │
    │  Cloudinary →  image uploads        │
    └─────────────────────────────────────┘
```

---

## Getting Started

### Prerequisites

- Node.js 18+
- PostgreSQL database
- Accounts for: Clerk, Stripe, Replicate, Pinecone, Upstash, Cloudinary, OpenAI

### Environment Variables

```env
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=
CLERK_SECRET_KEY=
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/
NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL=/

DATABASE_URL=

NEXT_PUBLIC_CLOUDINARY_CLOUD_NAME=

PINECONE_INDEX=
PINECONE_API_KEY=
PINECONE_ENVIRONMENT=

UPSTASH_REDIS_REST_URL=
UPSTASH_REDIS_REST_TOKEN=

OPENAI_API_KEY=
REPLICATE_API_TOKEN=

STRIPE_API_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

### Install & Run

```bash
npm install

npx prisma generate
npx prisma db push

npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

---

## Skills Demonstrated

- **Full-stack Next.js** — App Router, server components, API routes, middleware
- **LLM integration** — Prompt engineering, streaming responses, model orchestration with LangChain
- **Vector databases** — Embeddings, semantic search, hybrid memory retrieval with Pinecone
- **SaaS infrastructure** — Auth (Clerk), payments (Stripe), webhooks, subscription gating
- **Database design** — Relational schema with Prisma + PostgreSQL
- **Production concerns** — Rate limiting, singleton patterns, type safety end-to-end with TypeScript + Zod
