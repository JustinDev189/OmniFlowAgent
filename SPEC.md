# OmniFlowAgent — AI-Powered Omnichain Swap & Bridge Agent

> **Swap and bridge tokens across Ethereum, Solana, Base, and more using natural language. Intent-based execution with secure Wallet-as-a-Service — the agent never sees your keys.**

---

## 📖 Table of Contents

- [Project Goals & Architecture](#project-goals--high-level-architecture)
- [User Personas](#user-personas)
- [Interface & Wallet Options](#interface--wallet-options)
- [Functional Requirements](#functional-requirements)
- [Technical Stack](#technical-stack)
- [Database Schema](#database-schema)
- [Security Protocol](#security-protocol)
- [MVP Roadmap](#mvp-roadmap)
- [Business Model](#business-model)
- [Risk Disclosure](#risk-disclosure)

---

## 🎯 Project Goals & High-Level Architecture

### Project Goals

1. **Intent-based multi-chain execution** — Parse natural language (e.g. “Buy Token X on Solana”) and execute the optimal bridge-then-swap flow even when the user’s funds are on another chain (e.g. USDC on Arbitrum).
2. **Secure Wallet-as-a-Service (WaaS)** — Centralized, encrypted key management so the AI agent never has access to raw private keys; the backend signs and executes on behalf of the user.
3. **Unified chat experience** — Single conversational interface for balances, swaps, and bridges across all supported networks.
4. **Developer-grade execution** — Use industry SDKs (Li.Fi/Socket for bridging, Jupiter/Uniswap for swaps) for reliable routing and execution.

### High-Level Architecture (Text-Based Diagram)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  USER: Option 1 = Browser (Next.js + CopilotKit)  |  Option 2 = Telegram Bot     │
│  • Natural language input  • Balance display  • Transaction status              │
│  Option 2: No web UI; backend generates 12-word seed for user (no import).       │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  AI AGENT LAYER (OpenClaw / LangGraph)                                            │
│  • Intent parsing  • Multi-chain balance aggregation  • Route planning            │
│  • Output: INTENT REQUESTS only (no keys, no raw signing)                        │
│  • e.g. { action: "bridge_then_swap", fromChain: "arbitrum", toChain: "solana",  │
│           tokenIn: "USDC", tokenOut: "X", amount: "100" }                         │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  BACKEND (Node.js/TypeScript + Supabase)                                          │
│  • Auth (Email/Password)  • Intent API  • Signing orchestration                   │
│  • Receives intent → fetches encrypted material from KMS/HSM → signs → executes │
└─────────────────────────────────────────────────────────────────────────────────┘
         │                                    │
         ▼                                    ▼
┌──────────────────────┐          ┌──────────────────────────────────────────────┐
│  WALLET MANAGER      │          │  EXTERNAL SDK LAYER                           │
│  (Circle / WaaS)     │          │  • Li.Fi / Socket (bridge)                     │
│  • Encrypted seeds   │          │  • Jupiter (Solana swap)                       │
│  • HSM/KMS wrap      │          │  • Uniswap / 0x (EVM swap)                    │
│  • Sign & send tx    │          │  • Route resolution & tx building              │
└──────────────────────┘          └──────────────────────────────────────────────┘
         │                                    │
         └────────────────┬───────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  CHAINS: Ethereum │ Arbitrum │ Base │ Solana │ (extensible)                       │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Data flow (no keys to agent):**

1. User chats in the **Web UI (Option 1)** or **Telegram (Option 2)** → Agent reasons over balances (read-only RPC) and produces an **Intent Request**.
2. Backend receives Intent Request, authenticates the user (Email/Password or Telegram user_id), loads wallet material from KMS/HSM. For Option 2, new users get a **generated** 12-word seed (user never inputs seed).
3. Backend uses Li.Fi/Socket + Jupiter/Uniswap to resolve route and build transactions, then signs via Wallet Manager and submits to chains.
4. Results and status are written to DB and surfaced back in the same interface (Web or Telegram).

---

## 👤 User Personas

### Primary: The Multi-Chain Degen

| Attribute | Description |
|-----------|-------------|
| **Profile** | Active crypto user who holds assets on several chains (Ethereum, L2s, Solana). Wants to move and swap quickly without manually bridging and switching UIs. |
| **Goals** | “I want to say ‘buy $100 of Token X on Solana’ and have it happen even if my money is in USDC on Arbitrum.” Minimize steps and cognitive load. |
| **Pain points** | Manually checking balances per chain, choosing bridge, then DEX, then approving multiple txs. High friction and error-prone. |
| **Success criteria** | One chat command → correct bridge + swap executed with acceptable slippage and fees; keys never leave a secure backend. |

### Secondary Personas

- **Cautious multi-chain holder** — Wants to see all balances in one place and execute only after understanding route and cost; values transparency and control.
- **Power user / developer** — May want SDK or API access later; values clear intent semantics and reproducible execution.

---

## 🔀 Interface & Wallet Options

Teams can choose **one** of two paths for the MVP:

| | **Option 1: Web Chat UI** | **Option 2: Telegram** |
|---|---------------------------|------------------------|
| **Interface** | Next.js + Tailwind + CopilotKit in the browser. | Telegram Bot as the chat interface; no web UI to build. |
| **Auth** | Email/Password (Supabase Auth). | Telegram user ID (and optional username); no password. |
| **Wallet creation** | User may **import** a 12-word seed **or** request backend to **generate** a new one. | **Generate only:** backend creates a new 12-word seed for the user; **user does not input or import seed.** |
| **Seed handling** | Import flow: user pastes 12 words; backend encrypts and stores. Generate flow: backend generates, shows once for backup. | Backend generates 12 words (secure RNG), encrypts and stores; optionally show phrase once in private chat for backup, or keep server-side only. |
| **Use case** | Full control, existing wallets can be linked. | Fastest MVP: no UI work; new users get a fresh wallet automatically. |

For **Option 2 (Telegram)**, the backend generates the 12-word BIP-39 phrase for each new user; the user never provides or types a seed. All other requirements (encryption at rest, agent never sees keys, intent-based execution) apply unchanged.

---

## ⚙️ Functional Requirements

### 1a. Option 1: Web Chat UI (Frontend)

| ID | Requirement | Acceptance criteria |
|----|-------------|---------------------|
| FR-C1 | Natural language input | User can type or speak requests like “Swap 50 USDC to ETH on Base” or “Bridge 100 USDC from Arbitrum to Solana and buy BONK.” |
| FR-C2 | Multi-chain balance display | UI shows aggregated balances (by token and by chain) for the connected user, updated after auth. |
| FR-C3 | Intent confirmation | For non-trivial intents (bridge + swap, large amount), show route, estimated fees/slippage, and require explicit confirm before execution. |
| FR-C4 | Transaction history in chat | After execution, show tx hash(es), chain(s), and status (pending/success/failure) in the conversation. |
| FR-C5 | Auth gate | All balance and execution features require Email/Password login; unauthenticated users see only landing and login/signup. |

### 1b. Option 2: Telegram Interface

| ID | Requirement | Acceptance criteria |
|----|-------------|---------------------|
| FR-T1 | Telegram as chat | User talks to a Telegram Bot; natural language commands (balance, swap, bridge) are sent as messages; bot replies with balances, confirmations, and tx status. |
| FR-T2 | Auth by Telegram | User identified by Telegram `user_id` (and optional `username`); first message creates or links account; no Email/Password. |
| FR-T3 | Wallet: generate only | On first use, backend **generates** a new 12-word seed for the user; **user does not input or import a seed.** Seed is encrypted and stored; optionally sent once in a private reply for backup, or kept server-side only. |
| FR-T4 | Balance and execution in chat | Bot shows aggregated balances and executes intents (swap, bridge, bridge_then_swap) with confirmation in thread; tx hash and status returned in chat. |
| FR-T5 | No web UI | No Next.js/CopilotKit frontend required; all interaction is inside Telegram. |

### 2. Logic Engine (Reasoning / Agent)

| ID | Requirement | Acceptance criteria |
|----|-------------|---------------------|
| FR-L1 | Intent parsing | Agent maps natural language to structured intent: `bridge`, `swap`, `bridge_then_swap`, `balance_query`. |
| FR-L2 | Multi-chain balance aggregation | Agent (or backend on its behalf) queries balances for the user’s wallets on all connected chains and exposes a unified view to the reasoning layer. |
| FR-L3 | Route resolution | For “buy X on Chain B with funds on Chain A,” agent (or backend) determines: bridge path (e.g. Li.Fi/Socket) and swap path (Jupiter on Solana, Uniswap on EVM). |
| FR-L4 | Intent output only | Agent outputs only **Intent Requests** (JSON); no private keys, seeds, or raw transaction bytes. |
| FR-L5 | Error and edge handling | Unsupported token/chain or insufficient balance produces a clear message and, where applicable, a suggested alternative. |

### 3. Wallet Manager (Backend WaaS)

| ID | Requirement | Acceptance criteria |
|----|-------------|---------------------|
| FR-W1 | User authentication | **Option 1:** Email/Password (Supabase Auth); session/JWT. **Option 2:** Telegram `user_id`; no password. |
| FR-W2 | Seed import | **Option 1 only.** User can import a 12-word (BIP-39) seed phrase; phrase is encrypted and stored only in encrypted form; plaintext is never persisted. **Option 2 (Telegram):** not used — user does not input seed. |
| FR-W3 | Seed generation | User can request a new wallet; backend generates 12-word phrase (secure RNG), derives keys, stores encrypted form only. **Option 1:** optional show phrase once for backup. **Option 2 (Telegram):** backend always generates for new users; user never provides seed. |
| FR-W4 | Encryption at rest | Seed (or key material) encrypted with AES-256-GCM; key encryption key (KEK) in HSM or KMS; backend never logs or exposes raw seed/private key. |
| FR-W5 | Signing service | Backend exposes an internal “execute intent” API: accepts signed Intent Request + user id, retrieves decrypted key material via KMS/HSM, builds tx(s), signs, and submits. |
| FR-W6 | No key exposure to agent | Agent process has no network or file access to KMS/HSM or to raw keys; it only sends Intent Requests to the backend over authenticated channel. |

---

## 🛠️ Technical Stack

| Layer | Technology | Notes |
|-------|------------|--------|
| **Frontend (Option 1)** | Next.js 14+, Tailwind CSS, CopilotKit | App router; CopilotKit for chat UI and agent integration. |
| **Interface (Option 2)** | Telegram Bot API | Bot as chat interface; no web UI; auth by Telegram user_id. |
| **Agent / Reasoning** | OpenClaw or LangGraph | Orchestration and intent parsing; tools for balance and “request execution” only. |
| **Backend** | Node.js, TypeScript | REST or tRPC APIs for auth, intents, wallet operations; Option 2: webhook or long-polling for Telegram. |
| **Database & Auth** | Supabase | PostgreSQL (users, wallets metadata, transaction history). Option 1: Supabase Auth. Option 2: Telegram user_id as user identifier. |
| **Wallet-as-a-Service** | Circle Programmable Wallets (or Crossmint / Privy / Dynamic) | Custody, key derivation, and signing; align with HSM/KMS where required. |
| **Bridging** | Li.Fi SDK or Socket API | Route and build bridge transactions. |
| **Swaps** | Jupiter (Solana), Uniswap SDK / 0x (EVM) | Swap route and tx building. |
| **Key / Secret Management** | AWS KMS, GCP KMS, or HSM (e.g. AWS CloudHSM) | KEK for AES-256-GCM envelope encryption of seed material. |
| **Infrastructure** | Vercel (frontend), Railway / Fly.io / AWS (backend) | Backend must be able to call KMS/HSM and RPCs. |

---

## 🗄️ Database Schema

### Tables (Supabase/PostgreSQL)

```sql
-- Users: Option 1 = Supabase Auth (email/password). Option 2 = Telegram-only (no email).
-- For Option 2, use telegram_user_id as the unique identifier; email/password nullable.
CREATE TABLE public.users (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email            TEXT UNIQUE,                      -- Option 1: required; Option 2: NULL
  password_hash    TEXT,                             -- Option 1 only
  telegram_user_id BIGINT UNIQUE,                    -- Option 2: Telegram user id
  created_at       TIMESTAMPTZ DEFAULT NOW(),
  updated_at       TIMESTAMPTZ DEFAULT NOW()
);

-- Wallets: only metadata and reference to encrypted material; no raw seed/key.
CREATE TABLE public.wallets (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  chain_type      TEXT NOT NULL,         -- 'evm' | 'solana'
  address         TEXT NOT NULL,        -- derived address (public)
  encrypted_ref   TEXT NOT NULL,         -- KMS key id or HSM reference / ciphertext location
  derivation_path TEXT,                 -- e.g. "m/44'/60'/0'/0/0"
  is_primary      BOOLEAN DEFAULT false,
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, chain_type, derivation_path)
);

CREATE INDEX idx_wallets_user_id ON public.wallets(user_id);

-- TransactionHistory: audit trail of executed intents.
CREATE TABLE public.transaction_history (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id        UUID NOT NULL REFERENCES public.users(id),
  wallet_id      UUID REFERENCES public.wallets(id),
  intent_type    TEXT NOT NULL,         -- 'swap' | 'bridge' | 'bridge_then_swap'
  intent_payload JSONB NOT NULL,       -- sanitized intent (no secrets)
  chain          TEXT NOT NULL,
  tx_hash        TEXT,
  status         TEXT NOT NULL,         -- 'pending' | 'success' | 'failed'
  error_message  TEXT,
  created_at     TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_tx_history_user_created ON public.transaction_history(user_id, created_at DESC);
```

- **users**: 1:1 with auth identity if using Supabase Auth; otherwise store email and password hash.
- **wallets**: One row per derived address; `encrypted_ref` points to KMS/HSM or encrypted blob; never store plaintext seed or private key.
- **transaction_history**: Immutable log of intents and on-chain results for UX and support.

---

## 🔐 Security Protocol

### Encryption Flow (Seed / Key Material)

1. **At onboarding (import or generate)**  
   - User submits 12-word phrase (or requests generation).  
   - Backend generates a **data encryption key (DEK)** and encrypts the seed (or derived keys) with **AES-256-GCM**.  
   - DEK is encrypted with a **key encryption key (KEK)** held in **KMS or HSM** (envelope encryption).  
   - Only `(encrypted_seed, encrypted_DEK)` or KMS/HSM reference is stored; same for any derived private key if stored.  
   - Plaintext seed and DEK are never written to disk or logs and are cleared from memory after use.

2. **At signing (execute intent)**  
   - Backend receives authenticated Intent Request.  
   - Backend requests decryption of DEK from KMS/HSM, then decrypts seed/key material in memory.  
   - Builds transaction(s), signs in process, submits to chain(s).  
   - Clears key material from memory; no key material is sent to the agent or frontend.

### Agent-to-Backend Signing Request Authentication

1. **Session** — User is logged in; frontend holds a session token (e.g. Supabase JWT).
2. **Request** — Every “execute intent” request from the frontend (or from the agent service on behalf of the user) must include:
   - `Authorization: Bearer <session_jwt>`
   - Request body: `{ userId, intentRequest }` where `intentRequest` is the structured intent (e.g. bridge_then_swap with chain/token/amount).
3. **Backend checks**:
   - Verify JWT and extract `userId`.
   - Ensure `userId` matches the resource owner (e.g. wallet owner).
   - Optional: idempotency key to prevent replay.
4. **Agent** — If the agent runs in a separate service, it must call the backend with the user’s session token (obtained via secure handoff from the frontend, e.g. server-side only) so that the backend always authenticates the user, not the agent. The agent never signs; it only forwards intent + user context.

### Security Assumptions and Constraints

- Backend and any signing process run in a trusted environment with access to KMS/HSM.  
- Agent and frontend are untrusted for key material; they never receive seeds or private keys.  
- All sensitive APIs (balance for specific user, execute intent) require valid session JWT.  
- Use HTTPS and secure cookies/headers; consider rate limiting and abuse detection for execute-intent.

---

## 🗺️ MVP Roadmap (5-Day Hackathon Build)

Aligned with competition timeline: **Day 1** — Spec + GitHub + setup | **Day 2–3** — Implement | **Day 4** — Test + polish + demo prep | **Day 5** — Presentation.

**Option 2 (Telegram):** Choose this to skip building a web UI. Day 1 scaffold = backend + Telegram Bot (no Next.js/CopilotKit). Auth = Telegram `user_id`. Day 2 wallet = **generate 12-word seed only** (user does not input seed).

### Day 1 — Spec, GitHub, Scaffold & Auth Foundation

| Task | Owner | Deliverable |
|------|--------|-------------|
| Spec approved | Team | SPEC.md (this doc) submitted and approved by Lucas by 16h (per guidebook). |
| GitHub repo | Team | Public repo created; link shared in Telegram group. |
| Repo scaffold | Dev | **Option 1:** Next.js + Tailwind + CopilotKit (frontend) + Node.js/TS backend + Supabase. **Option 2:** Node.js/TS backend + Telegram Bot + Supabase (no web frontend). |
| Auth | Dev | **Option 1:** Email/Password via Supabase Auth. **Option 2:** Identify user by Telegram user_id; first message = create/link account. |
| DB schema | Dev | `users`, `wallets`, `transaction_history` tables in Supabase (see [Database Schema](#database-schema)). |
| Push Day 1 | Team | Spec + scaffold pushed to GitHub; tag Lucas in Telegram with progress. |

**EOD Day 1:** Repo live, spec approved, auth and DB foundation in place. Ready to build wallet and swap on Day 2.

---

### Day 2 — Wallet (WaaS) & Single-Chain Swap

| Task | Owner | Deliverable |
|------|--------|-------------|
| Wallet creation (MVP) | Dev | **Option 1:** Generate new or import 12 words per user; **Option 2 (Telegram):** generate 12-word seed only (user does not input seed). Encrypt with AES-256-GCM; store only encrypted_ref in DB. |
| Wallet Manager API | Dev | Backend endpoints: create wallet (and for Option 1 only: import wallet), get addresses; signing path uses encrypted material only server-side. |
| Single-chain swap | Dev | One chain (e.g. Base): swap X USDC to ETH via Uniswap/0x; backend builds tx, signs, submits; log in transaction_history. |
| Chat interface | Dev | **Option 1:** CopilotKit chat in browser; user asks balance on Base and swap 50 USDC to ETH; show tx hash. **Option 2:** Telegram bot: same flow in Telegram; bot replies with balance and tx hash. |
| Push Day 2 | Team | Code pushed to GitHub; tag Lucas with progress. |

**EOD Day 2:** **Option 1:** User signs up, creates or imports wallet, sees Base balance in web chat, executes one Base swap. **Option 2:** User opens Telegram bot, gets a generated wallet (no seed input), sees Base balance and executes one Base swap in chat.

---

### Day 3 — Bridge & Second Chain (Solana)

| Task | Owner | Deliverable |
|------|--------|-------------|
| Bridging SDK | Dev | Integrate Li.Fi or Socket: get bridge route and tx payload for e.g. Arbitrum to Base or Arbitrum to Solana. |
| Bridge execution | Dev | Backend builds bridge tx, signs with user wallet, submits; persist in transaction_history. |
| Solana wallet | Dev | Derive Solana address from same seed (or dedicated path); store in wallets; expose Solana balance to agent/backend. |
| Jupiter (Solana) | Dev | Swap on Solana via Jupiter API; backend signs and submits; log in transaction_history. |
| Intent bridge_then_swap | Dev/Agent | Agent parses e.g. Buy X on Solana with USDC on Arbitrum; backend runs bridge then swap in sequence; report status in chat. |
| Push Day 3 | Team | Code pushed to GitHub; tag Lucas with progress. |

**EOD Day 3:** User can say "Bridge 50 USDC from Arbitrum to Solana and swap to BONK"; agent + backend execute and report status in chat.

---

### Day 4 — Polish, Safety & Demo Prep

| Task | Owner | Deliverable |
|------|--------|-------------|
| Balance aggregation | Dev | Chat shows balances for all connected chains (EVM + Solana) in one place. |
| Confirmation step | Dev | For bridge+swap or amount above threshold, show route + estimated fees and require “Confirm” in UI. |
| Error handling | Dev | Clear messages for insufficient balance, unsupported token/chain, and tx failure. |
| Transaction history | Dev | List recent txs in UI (from `transaction_history`) with link to explorer. |
| Readme + env template | Dev | README with setup; `.env.example` for RPC, Supabase, KMS/API keys (no secrets). |
| AI showcase | Team | Screenshot or save prompts used for spec, architecture, and key features; ready for Presentation. |
| Demo run-through | Team | One happy path rehearsed (e.g. Buy Token X on Solana with USDC on Arbitrum); local run stable. |

**EOD Day 4:** MVP feature-complete; product ready for demo; AI showcase and demo flow prepared for Day 5.

---

### Day 5 — Presentation & Optional Final Polish

| Task | Owner | Deliverable |
|------|--------|-------------|
| Final smoke test | Team | Run full flow locally; fix any last-minute regressions. |
| Presentation | Team | Present per guidebook: Problem & User (2 min), Live Demo (5 min), AI Showcase (3 min), Roadmap (2 min), Q&A (5–8 min). |
| Q&A prep | Team | Anticipate BGK questions: security flow, scope cuts, edge cases, next steps. |

**EOD Day 5:** Presentation delivered; feedback captured for post-hackathon roadmap.

---

## 💰 Business Model (Post-MVP)

- **Transaction fee** — Small percentage on swap/bridge volume (e.g. 0.1–0.3%) or flat fee per execution.  
- **Premium / limits** — Free tier: N executions per month; paid tier: higher limits and priority routing.  
- **B2B / SDK** — License “Intent API + WaaS” to wallets or dApps for a white-label omnichain experience.

---

## ⚠️ Risk Disclosure

- **Smart contract and protocol risk** — Bridges and DEXs are third-party; exploits or bugs can lead to loss of funds.  
- **Custodial risk** — WaaS holds encrypted keys; compromise of backend or KMS/HSM could lead to theft. Mitigation: HSM/KMS, minimal key use, and clear key-handling spec.  
- **Slippage and execution** — Cross-chain and swap execution may differ from estimates; users should confirm limits and routes.  
- **Regulatory** — Compliance (e.g. sanctions, KYC) is the responsibility of the operator when moving to production.

---

## 📄 License and Contributing

- **License:** MIT (or as chosen by the team).  
- **Contributing:** Same as guidebook — built for internal competition; forks and feedback welcome.  
- **Repo structure (suggested):** `README.md`, `SPEC.md`, `src/` (frontend + backend), `ai-showcase/` (screenshots of prompts).

---

<div align="center">

**OmniFlowAgent** — *One chat. All chains. Your keys, never in the agent.*

</div>
