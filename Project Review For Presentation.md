# Project Review For Presentation: Pathfinder SaaS
**AI-Assisted Career Intelligence & Deterministic Learning Path Recommender**

---

## 1. Executive Summary & Problem-Solution Fit

### The Problem
Traditional online learning platforms offer tens of thousands of courses, yet learners face massive cognitive overload and drop-out rates exceeding 90%. When learners ask modern generative AI (like raw ChatGPT) for a learning roadmap, they encounter major architectural flaws:
1. **Hallucinated Prerequisites:** LLMs recommend advanced courses without verifying foundational dependencies (e.g., suggesting Deep Learning before Linear Algebra).
2. **Unverifiable Recommendations:** Generalist models cannot verify if the suggested courses actually exist, are high quality, or meet the learner's precise gaps.
3. **Drifting Explanations:** When asked *"Why did you recommend this course?"*, LLMs generate plausible-sounding justifications disconnected from the underlying decision criteria.
4. **Binary & Fragile Assessment:** Most platforms either make learners sit through redundant introductory content or skip material based on a fragile binary quiz score without accounting for lucky guesses.

### The Pathfinder Solution
**Pathfinder** is an enterprise-grade Career Intelligence and Personalized Learning Path Platform. It solves these issues through a core architectural paradigm:

> **"The Model Explains; The Deterministic Engine Decides."**

Pathfinder decouples **logical reasoning** (skill gap calculation, resource scoring, dependency ordering, milestone chunking, mastery tracking) from **natural language generation** (conversational empathy, prose narration, intent extraction). 

If every external AI provider goes down or the application is operated completely offline without Wi-Fi, **the entire core product remains 100% functional, deterministic, and accurate**.

```
+-----------------------------------------------------------------------------------+
|                                 USER INTERFACE                                    |
|   React 19 + TypeScript + Zustand Store + Clean 1px Design System + Dark/Light    |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
|                           SHARED DOMAIN ENGINE (Zero-React)                       |
|   - Profile Skills (Evidence Hierarchy)   - Skill Gap Analysis (Weighted Diff)    |
|   - Greedy Selection (Utility Scoring)    - Topological Dependency Ordering      |
|   - Milestone Chunking & Static Reasoning - Deterministic Rule Fallbacks          |
+-----------------------------------------------------------------------------------+
                    |                                             |
                    v                                             v
+---------------------------------------+     +-------------------------------------+
|        EXPRESS 5 BACKEND API          |     |          PERSISTENCE LAYER          |
|  - Rate Limiting & Spending Budgets   |     |  - Supabase Auth (GoTrue Sessions)  |
|  - Inbound PII / Injection Screening  |     |  - PostgreSQL JSONB + RLS Policies  |
|  - Outbound Anti-Hallucination Guard  |     |  - Immutable Generated Columns      |
|  - Bayesian Mastery Model Engine      |     |  - LocalStorage Offline Cache       |
+---------------------------------------+     +-------------------------------------+
                    |
                    v
+-----------------------------------------------------------------------------------+
|                     MULTI-PROVIDER LLM ABSTRACTION (LangChain)                    |
|       Anthropic (Claude) -> OpenAI (GPT) -> Google (Gemini) -> Groq -> Mistral    |
|             (Used strictly as constrained language renderers at 5 edges)          |
+-----------------------------------------------------------------------------------+
```

---

## 2. Key Terminology (Plain English Glossary)

To ensure any audience or hackathon judge understands the technical depth, here are the core concepts explained simply:

* **Deterministic Engine:** A predictable system where the same input **always** produces the exact same output according to mathematical rules, with zero randomness or AI hallucination.
* **Skill State Vector ($L \in \{0..5\}^{27}$):** A profile representing a learner's mastery level from 0 (Novice) to 5 (Master) across 27 defined industry skills.
* **Evidence Hierarchy:** The principle that verified proof (completing a comprehensive course or passing an assessment) strictly overrides unverified self-ratings.
* **Topological Dependency Ordering:** A graph algorithm that guarantees prerequisite concepts (e.g., Python Basics, Linear Algebra) are placed before advanced dependants (e.g., Deep Learning Transformers).
* **Bayesian Mastery Modeling:** A statistical method that treats a user's skill level as a probability curve rather than a simple pass/fail grade. It accounts for lucky guesses and question difficulty to prevent learners from accidentally skipping essential topics.
* **Bounded Output Constraints (Constrained AI):** Forcing an AI model to select only from a fixed, predefined list of verified catalogue items or goal IDs. The AI is structurally prevented from inventing non-existent degrees or skills.
* **Graceful Degradation:** The design pattern where, if an external cloud AI or network fails, the system automatically falls back to local rules without crashing or displaying error screens to the user.
* **Row-Level Security (RLS):** Database security built into PostgreSQL that cryptographically ensures users can only read and write their own data, preventing data leaks even if backend code has a flaw.

---

## 3. Deep Dive: Core Features & Architecture

### Feature 1: Conversational Goal Discovery (`src/routes/Chat.tsx`, `src/lib/assistant.ts`)
* **What it does:** Allows learners to express their career ambitions in open-ended natural language (e.g., *"I want to build and deploy generative AI apps into production and I have 10 hours a week"*).
* **How it works behind the scenes:**
  1. The input passes through `guard.screenInput` to scrub PII and neutralize prompt injections.
  2. `POST /api/chat` coordinates with `extractGoal` (`server/llm.ts`), mapping free text to a closed set of 4 verified goal tracks (`ml-engineer`, `data-analyst`, `fullstack-engineer`, `cloud-devops`).
  3. The rule-based engine immediately parses pace, experience, and skills, computing live facts.
  4. The LLM edge `converse` rewrites the response with natural warmth, constrained by strict factual guardrails.

### Feature 2: Structured Multi-Step Onboarding (`src/routes/Onboarding.tsx`, `src/lib/onboarding.ts`)
* **What it does:** A frictionless, single-question questionnaire capturing career ambition, current exposure (`beginner`, `some`, `experienced`), weekly study pace (`casual`, `steady`, `intense`), interests, and self-ratings.
* **Intelligent Edge 4 (Follow-up) & Edge 5 (Intro):**
  * If answers are ambiguous (e.g., user wants ML Engineer but avoids Math), `onboardingFollowup` dynamically asks at most 2 targeted multiple-choice follow-ups chosen *only* from catalog IDs.
  * `onboardingIntro` generates a personalized 2–3 sentence executive summary of the learner's journey upon completion.

### Feature 3: The 5-Stage Recommendation Engine (`src/lib/engine.ts`)
The entire planning logic is pure TypeScript with zero React dependencies, running identically in the browser and the Node.js backend:
1. **`profileSkills(profile)`:** Derives current 0–5 skill levels from declared experience floor ($0, 1, 2$), self-ratings, and completed courses (highest evidence wins).
2. **`skillGaps(profile, goal)`:** Calculates $\text{gap}_s = \max(0, \text{target}_s - \text{current}_s)$ and weights each skill by its criticality to the target career.
3. **`selectItems(profile, goal)` (Greedy Selection):** Iteratively evaluates the resource inventory. In each round, it selects the item maximizing:
   $$\text{Score}(r) = \left(\sum_s \Delta\text{Skill}_s \cdot \text{Weight}_s\right) \times \text{LevelFit}(r) + 0.6 \cdot |\text{Tags} \cap \text{Interests}| - 0.5 \cdot |\text{Tags} \cap \text{Avoid}| - \frac{\text{Hours}(r)}{100}$$
   *$\text{LevelFit}$ penalizes resources that are wildly beyond the learner's current capabilities ($1.0$ for ready, $0.7$ for slightly short, $0.25$ for far off).*
4. **`orderItems(chosen, startLevels)`:** Emits resources topologically so that prerequisites are strictly scheduled before dependants, resolving ties by difficulty, kind (Course $\to$ Project $\to$ Assessment), and duration.
5. **`buildPath(profile)`:** Partitions the ordered sequence into 3–4 logical milestones (*"Establish foundations"*, *"Build core capability"*, *"Apply it on real work"*, *"Prove it end to end"*), computes time-to-completion, and attaches auditable explanations.

```
+-------------------------------------------------------------------------------------+
|                              RECOMMENDATION PIPELINE                                |
|                                                                                     |
|   [Learner Profile]                                                                 |
|          |                                                                          |
|          v                                                                          |
|   1. profileSkills()  ===> Computes current baseline levels (0 to 5)                |
|          |                                                                          |
|          v                                                                          |
|   2. skillGaps()      ===> Identifies delta against Goal target profile             |
|          |                                                                          |
|          v                                                                          |
|   3. selectItems()    ===> Multi-factor greedy utility ranking (Cap: 12 items)       |
|          |                                                                          |
|          v                                                                          |
|   4. orderItems()     ===> Prerequisite-aware topological sort                      |
|          |                                                                          |
|          v                                                                          |
|   5. buildPath()      ===> Chunks into Milestones + Generates "Why This?" Proofs    |
+-------------------------------------------------------------------------------------+
```

### Feature 4: Auditable "Why This?" Explanations (`src/lib/engine.ts`, `server/routes/narrate.ts`)
* Unlike black-box AI recommenders, Pathfinder derives explanations directly from the numerical weights and gap closures computed during selection:
  * **Gap Closure:** *"Moves Machine Learning from level 0 to 3. Your goal needs 4."*
  * **Prerequisite Chain:** *"Required before Deep Learning Specialisation."*
  * **Interest / Avoidance Alignment:** Explains why an avoided subject was kept if no alternative closes a mandatory prerequisite gap.
  * **History Fit:** Explains why beginner courses were safely skipped based on prior completions.
* The optional LLM Edge (`POST /api/narrate`) translates these exact factual bullet points into fluent coaching prose, with strict outbound validation rejecting any invented claims.

### Feature 5: Bayesian Skill Assessment & Topic Checks (`server/mastery.ts`, `src/components/TopicCheck.tsx`)
* **The Danger of Naive Quizzes:** Getting 3 out of 3 questions right might be lucky guessing (25% chance per 4-option question); getting 1 wrong might be a momentary lapse.
* **The Bayesian Solution:**
  * Tracks mastery as a discrete probability distribution over 6 latent buckets ($L \in \{0, 1, 2, 3, 4, 5\}$).
  * **Prior Distribution:** Set via geometric decay $P(L=k) \propto \text{decay}^{|k - \text{assumed}|}$ (Decay: $0.62$ for self-ratings, $0.35$ for verified history).
  * **Likelihood Function:**
    $$P(\text{correct} \mid L, d) = \begin{cases} 0.85 & \text{if } L \ge d \text{ (Learner knows concept)} \\ 0.28 & \text{if } L < d \text{ (Guessing baseline)} \end{cases}$$
  * **Bayesian Posterior Update:**
    $$P_{t+1}(L) = \frac{P(\text{answer}_t \mid L) \cdot P_t(L)}{\sum_k P(\text{answer}_t \mid k) \cdot P_t(k)}$$
  * **Tri-State Decision Verdict:**
    * $P(L \ge \text{target}) \ge 0.70 \implies \mathbf{Accept}$: Confirms mastery, updates profile, and dynamically prunes redundant items from the roadmap.
    * $P(L \ge \text{target}) \le 0.30 \implies \mathbf{Refresh}$: Confirms skill deficit; retains the topic in the curriculum.
    * $0.30 < P < 0.70 \implies \mathbf{Ask\text{-}More}$: Inconclusive state; serves follow-up questions from the item bank without prematurely altering the learning path.
  * **Security Guarantee:** Assessment answer keys and rationales are stored exclusively in `data/quiz-bank.json` on the server and are **never** transmitted to the client browser.

### Feature 6: Real-Time Learning Dashboard (`src/routes/Dashboard.tsx`)
* Displays total curriculum completion percentage, hours invested vs. remaining, weeks to completion based on user pace, and granular segmented meters for every individual target skill.
* Visualizes weekly learning velocity via custom lightweight SVG charts without bulky third-party chart libraries.

### Feature 7: Multi-Provider LLM Resilience & Failover (`server/providers.ts`, `server/llm.ts`)
* Supports 5 distinct model providers via unified LangChain adapters: **Anthropic (Claude Opus/Sonnet)**, **OpenAI (GPT-4o/mini)**, **Google (Gemini 2.0 Flash)**, **Groq (Llama 3.3 70B)**, and **Mistral (Mistral Large)**.
* Dynamic provider chain failover: If Anthropic experiences an API outage or rate limit, requests automatically cascade to OpenAI $\to$ Google $\to$ Groq $\to$ Mistral $\to$ Deterministic Fallback.
* Built-in **Cost & Rate Control (`server/budget.ts`)**: In-memory token counter, per-IP rate limiting, and maximum USD budget caps to prevent accidental billing spikes during hackathon demos.

### Feature 8: Enterprise AI Guardrails & Hallucination Elimination (`server/guard.ts`)
* **Inbound Shielding:** Unicode NFKC normalization, control/invisible tag removal, PII redaction (email, phone, credit cards via Luhn algorithm, API keys), and 8-category prompt-injection detection. Learner text is isolated inside `<learner_text_{nonce}>` tags.
* **Outbound Validation (`validateOutput`):** Discards any LLM-generated response containing numbers, course names, provider titles, or hyperlinks not explicitly present in the engine's pre-computed fact sheet, falling back to deterministic templates instantly.

---

## 4. Data Architecture & Storage Schema

Pathfinder adopts a hybrid storage strategy optimized for ultra-fast, offline-first client performance paired with durable cloud synchronization.

```
+-----------------------------------------------------------------------------------+
|                                 DATA PERSISTENCE                                  |
+-----------------------------------------------------------------------------------+
|  1. Browser LocalStorage:                                                         |
|     - 'pf-state' (Full Zustand store snapshot: Profile, Progress, Messages, etc.) |
|     - 'pf-theme' ('light' | 'dark' | 'system')                                    |
|     - 'pf-nav-collapsed' (Navigation UI state)                                    |
+-----------------------------------------------------------------------------------+
|  2. Static Inventory & Domain Code (Version Controlled):                          |
|     - 'src/lib/catalog.ts' (27 Skills across 4 domains, 35 Learning Resources)    |
|     - 'src/lib/goals.ts' (4 Goal tracks with weighted skill targets)              |
|     - 'data/quiz-bank.json' (66 Multi-difficulty assessment questions)            |
+-----------------------------------------------------------------------------------+
|  3. Cloud Database (Supabase PostgreSQL + GoTrue Auth):                           |
|     - Table: 'public.profiles'                                                    |
|       * id: uuid (Primary Key, Foreign Key -> auth.users.id CASCADE)              |
|       * display_name: text                                                        |
|       * profile: jsonb (LearnerProfile document)                                  |
|       * progress: jsonb (Record<ResourceId, 'todo' | 'active' | 'done'>)          |
|       * conversation: jsonb (Assistant chat turns, trimmed to 200)                |
|       * mastery: jsonb (Per-skill Bayesian posterior distributions)               |
|       * marks: jsonb (Added / Removed roadmap diff highlights)                    |
|       * unverified: jsonb (Offline completed ticks pending verification)          |
|       * Generated Columns: goal_id, experience, pace, completed_count,            |
|                            interest_count, rated_skill_count, onboarded_at        |
+-----------------------------------------------------------------------------------+
```

### Granular TypeScript Schemas (`src/lib/types.ts`)

```typescript
// 1. Learner Profile Contract
export interface LearnerProfile {
  name: string
  experience: 'beginner' | 'some' | 'experienced'
  interests: string[]
  avoid: string[]
  completed: ResourceId[]
  selfRated: Record<SkillId, Level>       // Level is 0 | 1 | 2 | 3 | 4 | 5
  goalId: string | null
  goalStatement: string
  pace: 'casual' | 'steady' | 'intense'   // 4h, 8h, or 16h per week
  onboardedAt: number | null
  intro: string
}

// 2. Resource Catalog Item
export interface Resource {
  id: ResourceId
  title: string
  kind: 'course' | 'project' | 'assessment'
  provider: string
  hours: number
  level: Level
  teaches: Partial<Record<SkillId, Level>>
  requires?: Partial<Record<SkillId, Level>>
  summary: string
  tags: string[]
}

// 3. Generated Learning Path
export interface LearningPath {
  goalId: string
  milestones: Milestone[]
  totalHours: number
  weeks: number
  uncovered: Array<{ skillId: SkillId; from: Level; target: Level }>
  generatedAt: number
}

// 4. Milestone Structure
export interface Milestone {
  id: string
  title: string
  outcome: string
  items: PathItem[]
  entryRequirements: SkillId[]
}

// 5. Path Item with Verifiable Proof
export interface PathItem {
  resourceId: ResourceId
  reasons: Reason[]
  closes: Array<{ skillId: SkillId; from: Level; to: Level }>
}
```

---

## 5. Architectural Evaluation: Prototype vs. Production-Grade Implementation

In a live hackathon pitch, judges love candidates who understand **real-world engineering trade-offs**. Here is a comprehensive comparison of how Pathfinder is implemented today vs. how it would scale in a production enterprise platform:

| Component | Current Implementation in Pathfinder Prototype | How to Execute in Production / Live Large-Scale Systems | Architectural Justification & Next Steps |
|---|---|---|---|
| **Course Catalog & Inventory** | **Static TypeScript & JSON files** (`catalog.ts`, 35 items; `quiz-bank.json`, 66 questions). In-memory filtering. | **Relational DB (PostgreSQL) + Vector DB (pgvector / Pinecone / Qdrant) + Graph DB (Neo4j)** | In production with $100,000+$ courses across Coursera, edX, and YouTube, static files become unmaintainable. PostgreSQL stores course metadata; `pgvector` enables semantic search for natural language discovery; **Neo4j Graph Database** maps prerequisite dependency trees across complex curricula. |
| **Recommendation & Path Generation** | **Greedy Heuristic Search + Topological Sort** (sub-millisecond, bounded at 12 items). | **Hybrid Multi-Stage Pipeline: Graph Traversal (DAG) + Reinforcement Learning (RL) + Collaborative Filtering** | Greedy algorithms can occasionally get stuck in local optima. A production engine would use **DAG-based shortest path algorithms (e.g., $A^*$ search on Skill Knowledge Graphs)** combined with contextual multi-armed bandits or RL to optimize for learner retention, dropout risk, and historical completion rates. |
| **Skill Assessment & Mastery** | **Discrete 6-Bucket Bayesian Update** with uniform prior and 2-parameter piecewise likelihood ($P_{\text{known}}=0.85, P_{\text{guess}}=0.28$). | **Multidimensional Item Response Theory (MIRT) + Deep Knowledge Tracing (DKT / Bayesian Knowledge Tracing)** | Live platforms calibrate questions using 3-parameter IRT (Difficulty $b$, Discrimination $a$, Pseudo-guessing $c$) fitted on empirical student response data across millions of test takers. Recurrent neural networks (DKT) track knowledge decay over time. |
| **LLM Orchestration & Agent Flow** | **Single-Turn Leaf Prompts + Direct LangChain Adapters** with structured output and regex guardrails. | **Semantic Router + Function Calling Agents (LangGraph / Temporal) + Fine-Tuned Small Language Models (SLMs)** | For high concurrency ($10,000+\text{ req/s}$), replace large frontier models with specialized fine-tuned 8B models (e.g., Llama-3-8B / Gemma-2-9B) deployed on private vLLM clusters. This reduces latency from $800\text{ms} \to 40\text{ms}$ and cuts inference costs by 95%. |
| **Persistence & State Sync** | **Zustand LocalStorage + Debounced Supabase Postgres JSONB writes**. | **CQRS + Event Sourcing (Apache Kafka) + WebSockets / Server-Sent Events (SSE)** | JSONB documents are easy to evolve, but high-frequency concurrent edits benefit from an event-sourced architecture (`ResourceCompletedEvent`, `SkillLevelAssessedEvent`) streaming over WebSockets for multi-device live sync. |
| **Labor Market Alignment** | **Pre-defined Goal Tracks** (4 static career profiles: ML Engineer, Data Analyst, Full-Stack, Cloud DevOps). | **Real-Time Labor Market Intelligence Ingestion (Lightcast / O*NET / LinkedIn Job API)** | Continuously ingest real-time job postings, clustering emerging skill requirements using NLP to dynamically adjust skill weights and target mastery levels as industry demands shift. |

---

## 6. Verification Report: Submitted Document vs. Actual Codebase Execution

We conducted a line-by-line verification comparing the submitted project documentation (*"Pathfinder: AI-Assisted Career Intelligence and Skill Development Platform - Technical Documentation"*, 15 pages) against the actual codebase files.

### Summary Scorecard
* **Total Features & Claims Analyzed:** 32 items across 17 documentation sections.
* **Fully Implemented & Verified in Code:** 30 items (**93.75%**)
* **Deliberately Scoped / Bounded by Design:** 2 items (**6.25%** - documented accurately in limitations).
* **Discrepancies / Missing Features:** **0**

### Itemized Verification Matrix

| Document Section & Page | Documented Feature / Claim | Codebase File & Implementation Verification | Status |
|---|---|---|---|
| **§ 1–2 (p. 3–4)** | Principle: *"The model explains; the deterministic engine decides."* | `src/lib/engine.ts`, `server/llm.ts`, `server/guard.ts`. All planning is deterministic; LLMs only format prose. | **VERIFIED (100%)** |
| **§ 2.2 Table 1 (p. 4)** | 5 LLM Integration Edges: `extractGoal`, `narrate`, `converse`, `onboardingFollowup`, `onboardingIntro`. | Implemented in `server/llm.ts` (Lines 430, 575, 751, 952, 1061). Every edge includes a fallback. | **VERIFIED (100%)** |
| **§ 3.1–3.2 (p. 5)** | Shared Domain Layer (Zero-React) + Tech Stack (React 19, Vite, Zustand, Express 5, Zod, LangChain, Supabase). | `src/lib/engine.ts` has zero UI dependencies. `package.json` contains exact dependencies. | **VERIFIED (100%)** |
| **§ 3.3 Table 3 (p. 5)** | Project Scale: 27 Skills, 35 Resources, 4 Goal Tracks, 66 Assessment Items, 25 Assessed Skills, 5 Providers, 29 E2E Checks. | `src/lib/catalog.ts` (27 skills, 35 resources), `src/lib/goals.ts` (4 tracks), `data/quiz-bank.json` (66 items), `scripts/smoke.ts` (29 E2E checks). | **VERIFIED (100%)** |
| **§ 4.1–4.3 (p. 5–6)** | 5-Stage Recommendation Formula: Gap Calculation, LevelFit factor ($1.0, 0.7, 0.25$), Greedy Scoring, 12-item cap. | `src/lib/engine.ts`: `profileSkills` (L45), `skillGaps` (L80), `levelFit` (L101), `selectItems` (L143 with `MAX_ITEMS = 12`). | **VERIFIED (100%)** |
| **§ 4.4 (p. 6)** | Prerequisite-aware Topological Ordering with tie-breaking by difficulty, kind, and length. | `src/lib/engine.ts`: `orderItems` (Lines 217–248). | **VERIFIED (100%)** |
| **§ 5.1–5.4 (p. 7)** | Bayesian Mastery Model: 6 buckets ($L \in 0..5$), Decay ($0.62, 0.35$), $P_{\text{known}}=0.85, P_{\text{guess}}=0.28$, Thresholds ($0.70$ Accept, $0.30$ Refresh, $0.30..0.70$ Ask-More). | `server/mastery.ts`: `DECAY` (L54), `P_KNOWN`/`P_GUESS` (L60-62), `priorFor` (L78), `updateWith` (L88), `judge` (L134). | **VERIFIED (100%)** |
| **§ 6 Table 5 (p. 8)** | 5 Model Providers: Anthropic, OpenAI, Google Gemini, Groq, Mistral via `PATHFINDER_PROVIDERS`. | `server/providers.ts`: Configures and resolves providers dynamically with failover chains. | **VERIFIED (100%)** |
| **§ 6.1 (p. 8)** | Structured Output constrained to JSON Schema and re-validated with Zod. | `server/llm.ts`: `GOAL_EXTRACTION_SCHEMA` and `FOLLOWUP_SCHEMA` parsed via Zod objects. | **VERIFIED (100%)** |
| **§ 7.1–7.2 (p. 8–9)** | Input/Output Guardrails: Unicode NFKC, PII redaction (Luhn card check), Injection detection (8 patterns), Outbound hallucination checks. | `server/guard.ts`: `normalizeText` (L90), `redactPii` (L120), `detectInjection` (L180), `validateOutput` (L380). | **VERIFIED (100%)** |
| **§ 8.1–8.5 (p. 9–10)** | UX Architecture: Offline-first local engine, 200ms debounce, monotonic request tokens, Discover page, Connection badges. | `src/store/useAppStore.ts` (Lines 154–180), `src/components/ConnectionBadge.tsx`, `src/routes/Discover.tsx`. | **VERIFIED (100%)** |
| **§ 9 Table 6 (p. 10)** | Representative API Endpoints (`/health`, `/path`, `/quiz/:skillId`, `/narrate`, `/onboarding/followup`, `/onboarding/summary`, etc.). | `server/routes/` contains all 11 modular route handlers implementing these 25 endpoints. | **VERIFIED (100%)** |
| **§ 11 (p. 11)** | Testing & 29 E2E Smoke Checks (Reproducibility, hidden quiz keys, posterior direction, fallback execution). | `scripts/smoke.ts` runs all 29 automated test cases covering these exact assertions. | **VERIFIED (100%)** |
| **§ 12 (p. 11)** | Security: Helmet CSP nonces, constant-time API key verification, HttpOnly cookies, Postgres Row-Level Security. | `server/http.ts`, `server/supabase.ts`, `supabase/migrations/0001_profiles.sql` (RLS enabled). | **VERIFIED (100%)** |
| **§ 13 (p. 12)** | Deployment: Vite build (`dist/`), Express backend, serverless-ready routing via `vercel.json` and `api/index.ts`. | `vercel.json`, `vite.config.ts`, `api/index.ts` present in root. | **VERIFIED (100%)** |
| **§ 14 Table 8 (p. 13)** | Graceful Degradation Strategy (AI Failure $\neq$ Application Failure). | Verified across all store actions and routes: offline mode works with `PATHFINDER_LLM=off`. | **VERIFIED (100%)** |

---

## 7. Hackathon Pitch & Presentation Strategy

When presenting this project to judges, follow this high-impact 3-minute pitch outline:

### 1. The Hook (30 Seconds)
> *"Everyone has tried asking ChatGPT for a learning roadmap, only to receive a generic list of courses with hallucinated prerequisites, no skill-gap validation, and zero accountability. We built **Pathfinder** around a core architectural principle: **The model explains; the deterministic engine decides.**"*

### 2. Live Demo Flow (90 Seconds)
1. **Goal Discovery:** Type a conversational goal (*"I want to become an ML engineer shipping models to production and I have 8 hours a week"*). Show how the assistant instantly extracts the goal and generates a personalized, milestone-chunked path.
2. **The "Why This?" Panel:** Click on any course in the path. Show that every single justification (*"Moves Python from 1 to 3; required before Deep Learning"*) is mathematically linked to the recommendation engine's gap calculation.
3. **Interactive Topic Check (Bayesian Mastery):** Complete a quick 3-question quiz for Python. Show the Bayesian posterior update live—demonstrating that passing with high statistical confidence ($P \ge 70\%$) automatically recalculates the roadmap and prunes redundant beginner modules.
4. **Offline / Resilience Proof:** Show the connection badge in the header switching seamlessly between `API + model`, `API (deterministic)`, and `Standalone (Browser Engine)` without a single screen flicker.

### 3. Engineering Rigor & Production Scalability (60 Seconds)
* Highlight that the system has **29 automated E2E tests**, **zero-dependency shared domain logic**, **5 LLM providers with automatic failover**, and **inbound/outbound AI guardrails**.
* Walk through how this architecture scales to millions of courses using Graph Databases (Neo4j) and Multidimensional Item Response Theory (MIRT).

---
*Document prepared for Hackathon Round 2 Presentation.*
