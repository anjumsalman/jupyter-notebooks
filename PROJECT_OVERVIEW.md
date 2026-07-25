# Decision Compass - Complete Technical Reference

## Product Summary

**Decision Compass** is a workspace-based product decision intelligence platform for PM teams. It enables evidence-driven product decisions through a structured workflow: **Feedback → Features → Problems → Decisions → Evidence → Confidence Scoring → PRD Generation**, with AI-assisted workflows and a governed multi-agent analysis pipeline.

---

## Architecture

### Tech Stack
- **Frontend**: React 18 + TypeScript + Vite + React Router 6
- **State**: React Query (TanStack Query) for server state, Zustand for local workspace, React Context for auth/workspace/product contexts
- **UI**: shadcn/ui (Radix primitives) + Tailwind CSS
- **Backend**: Supabase (PostgreSQL + RLS + Edge Functions)
- **AI**: Lovable Gateway → Google Gemini models
- **Auth**: Supabase Auth (email/password)

### Runtime Layers
```
React App (Routes + Pages + Hooks) → Supabase Edge Functions
     ↓                              ↓
Supabase Postgres + RLS        LLM Gateway
     ↓                              ↓
                              Observability (traces, agent outputs, logs)
```

### Multi-Tenancy Model
- **Workspace** = primary tenant boundary (all entities workspace-scoped)
- **Product** = secondary boundary (routes `/p/:slug/*` activate product scope)
- **Roles**: `owner` | `full_access` | `view_access` (enforced at UI + RLS)

---

## Core Data Model (ERD)

```
workspaces ──┬── workspace_memberships ── profiles
             ├─ products
             ├─ feedback_items
             ├─ product_features
             ├─ problems
             ├─ decisions ── decision_options
             │           └─ evidence_artifacts
             │           └─ confidence_scores
             ├─ product_documents ── product_document_versions
             ├─ workflow_runs
             ├─ pmos_artifacts
             ├─ ai_conversations ── ai_messages
             ├─ activity_log
             ├─ workspace_invitations
             └─ ai_rules
```

---

## Entity Definitions

### 1. Workspace (`workspaces`)
**Purpose**: Primary tenant container. All operational data is workspace-scoped.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | Auto-generated |
| `user_id` | UUID (FK→profiles) | Owner of workspace |
| `name` | TEXT | Workspace display name |
| `description` | TEXT | Optional description |
| `created_at` | TIMESTAMPTZ | Auto-set |
| `updated_at` | TIMESTAMPTZ | Auto-updated via trigger |

**RLS**: `user_owns_workspace(ws_id)` - only owner can CRUD workspace itself

---

### 2. Workspace Membership (`workspace_memberships`)
**Purpose**: Links users to workspaces with roles.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | Auto-generated |
| `workspace_id` | UUID (FK→workspaces) | |
| `user_id` | UUID (FK→profiles) | |
| `role` | `workspace_role` ENUM | `owner` \| `full_access` \| `view_access` |
| `joined_at` | TIMESTAMPTZ | Auto-set |

**Note**: Created via invitation acceptance or owner creation

---

### 3. Profile (`profiles`)
**Purpose**: Extends Supabase auth.users with display info.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK, FK→auth.users) | |
| `email` | TEXT | From auth |
| `full_name` | TEXT | Optional |
| `company_name` | TEXT | Optional |
| `location` | TEXT | Optional |
| `avatar_url` | TEXT | Optional |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

**Auto-created**: Trigger `handle_new_user()` on auth.user insert

---

### 4. Product (`products`)
**Purpose**: Product scope within workspace. Enables `/p/:slug/*` routing.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `name` | TEXT | Display name |
| `slug` | TEXT | URL-safe identifier (unique per workspace) |
| `created_by` | UUID (FK→profiles) | |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

**RLS**: `has_workspace_access(workspace_id)` for SELECT, `has_workspace_edit_access(workspace_id)` for mutations

---

### 5. Feedback Item (`feedback_items`)
**Purpose**: Raw customer signals. Input to problem discovery.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `product_id` | UUID (FK→products, NULLABLE) | Set when created in product scope |
| `type` | `feedback_type` ENUM | `interview` \| `note` \| `quote` \| `insight` |
| `title` | TEXT | Brief summary |
| `content` | TEXT | Full feedback text |
| `source` | TEXT | Origin (customer name, ticket ID, etc.) |
| `customer_segment` | TEXT | Segment tag |
| `tags` | TEXT[] | Array of tags |
| `created_at` | TIMESTAMPTZ | |

**Import**: CSV import supported via `addBulkFeedback` hook

---

### 6. Product Feature (`product_features`)
**Purpose**: Catalog of implemented/planned features per product.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `product_id` | UUID (FK→products, NULLABLE) | |
| `name` | TEXT | Feature name |
| `description` | TEXT | What it does |
| `purpose` | TEXT | Why it exists / user value |
| `limitations` | TEXT | Known constraints |
| `module` | TEXT | Logical grouping (e.g., "Onboarding", "Dashboard") |
| `user_journey` | TEXT | Journey stage (e.g., "Activation", "Daily Use") |
| `customer_segments` | TEXT[] | Target segments |
| `created_at` | TIMESTAMPTZ | |

**UI**: Grouped by `module` in Features page. CSV import supported.

---

### 7. Problem (`problems`)
**Purpose**: Synthesized problem statements derived from feedback/features.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `product_id` | UUID (FK→products, NULLABLE) | |
| `title` | TEXT | Problem statement |
| `description` | TEXT | Detailed description |
| `customer_impact` | TEXT | Impact narrative |
| `evidence_strength` | `evidence_strength` ENUM | `strong` \| `moderate` \| `weak` |
| `frequency` | INTEGER | Occurrence count (optional) |
| `status` | `problem_status` ENUM | `identified` \| `validated` \| `addressed` |
| `linked_feedback` | UUID[] | Array of `feedback_items.id` |
| `related_features` | UUID[] | Array of `product_features.id` |
| `created_at` | TIMESTAMPTZ | |

**Mapping Recalculation**: RPC `recalculate_problem_mappings` (similarity threshold) updates `linked_feedback` and `related_features` via `problem_evidence_map` and `problem_feature_map` tables

---

### 8. Decision (`decisions`)
**Purpose**: Core decision record tying problem → options → evidence → confidence → PRD.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `product_id` | UUID (FK→products, NULLABLE) | |
| `problem_id` | UUID (FK→problems, NULLABLE) | Source problem |
| `selected_option_id` | UUID (FK→decision_options, NULLABLE) | Chosen option |
| `decision_type` | `decision_type` ENUM | `enhancement` \| `fix` \| `replacement` \| `net-new` \| `defer` |
| `confidence` | JSONB | Structured confidence metadata (see below) |
| `rationale` | TEXT | Decision rationale |
| `owner` | TEXT | Owner name |
| `status` | `decision_status` ENUM | `draft` \| `pending` \| `decided` \| `implemented` |
| `prd_generated` | BOOLEAN | Whether PRD exists |
| `prd_content` | TEXT | Generated PRD markdown |
| `confidence_score` | INTEGER (0-100) | Evidence-weighted score for UI |
| `strategic_alignment` | NUMERIC (0-1) | PM input for scoring |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

**Confidence JSONB Structure**:
```json
{
  "evidenceStrength": "low|medium|high",
  "dataCoverageGaps": ["gap1", "gap2"],
  "pmConfidence": "low|medium|high",
  "overallConfidence": "low|medium|high",
  "notes": "optional"
}
```

---

### 9. Decision Option (`decision_options`)
**Purpose**: Discrete options for a decision with trade-offs.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `decision_id` | UUID (FK→decisions) | |
| `title` | TEXT | Option title |
| `description` | TEXT | Detailed description |
| `type` | `decision_type` ENUM | Same as decision.type |
| `trade_offs` | JSONB | `{ "pros": [], "cons": [] }` |
| `effort` | `effort_level` ENUM | `low` \| `medium` \| `high` |
| `impact` | `impact_level` ENUM | `low` \| `medium` \| `high` |
| `related_features` | UUID[] | Array of `product_features.id` |
| `created_at` | TIMESTAMPTZ | |

**Generation**: AI via `generate-decision-options` edge function (enforces 1 defer option)

---

### 10. Evidence Artifact (`evidence_artifacts`)
**Purpose**: Typed, structured evidence attached to decisions for scoring.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `decision_id` | UUID (FK→decisions) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `product_id` | UUID (FK→products, NULLABLE) | |
| `source_feedback_id` | UUID (FK→feedback_items, NULLABLE) | Auto-linked from problem mappings |
| `type` | TEXT | `interview` \| `usage_data` \| `experiment` \| `support_ticket` \| `meeting_note` \| `survey` \| `analytics` \| `community` \| `insight` \| `quote` \| `note` |
| `segment` | TEXT | Customer segment |
| `recency_days` | INTEGER | Days since evidence created |
| `sample_size` | INTEGER | Sample count (default 1) |
| `severity` | INTEGER (1-5) | Problem severity rating |
| `frequency` | INTEGER (1-5) | Occurrence frequency |
| `source_reliability` | NUMERIC (0-1) | Source credibility |
| `linked_metrics` | BOOLEAN | Whether metrics attached |
| `content` | TEXT | Evidence summary (max 2000 chars) |

**Auto-generation**: `calculate-decision-score` backfills from `problem_evidence_map` + `feedback_items` if empty (idempotent via `source_feedback_id`)

---

### 11. Confidence Score (`confidence_scores`)
**Purpose**: Persisted scoring output for PRD gate and UI.

| Field | Type | Description |
|-------|------|-------------|
| `decision_id` | UUID (PK, FK→decisions) | Unique per decision |
| `evidence_strength` | NUMERIC (0-1) | Weighted evidence quality |
| `pm_confidence` | NUMERIC (0-1) | PM judgment input |
| `overall_confidence` | NUMERIC (0-1) | Combined score (gate threshold = 0.65) |
| `label` | TEXT | `High` \| `Medium` \| `Low` |
| `data_gaps` | TEXT[] | Identified gaps |
| `data_gap_penalty` | NUMERIC | Penalty applied |
| `scoring_version` | TEXT | `v1.0` (deterministic) \| `agentic_v1` (agentic) |
| `calculated_at` | TIMESTAMPTZ | |

**Two Scoring Paths** (Gap-04 to reconcile):
1. **Deterministic** (`calculate-decision-score`): Evidence artifact weighting + feature coverage + PM inputs
2. **Agentic** (`decision-analyze` → `scoreEngine`): Evidence strength + agent confidence avg + alignment + decision quality
3. **Unified write** (`writeUnifiedScore`): Mirrors agentic 0-1 → `decisions.confidence_score` (0-100) + this table

---

### 12. Product Document (`product_documents`)
**Purpose**: Strategic documentation per product with version history.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `product_id` | UUID (FK→products) | |
| `doc_type` | TEXT | `brief` \| `roadmap` \| `metrics` \| `updates` \| `experiments` \| `releases` |
| `content` | TEXT | Markdown content |
| `created_by` | UUID (FK→profiles) | |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

**Categories**:
- `brief`: Problem, hypothesis, success metrics
- `roadmap`: Now/Next/Later + milestones
- `metrics`: Key metric snapshots
- `updates`: Status updates (On Track/At Risk/Blocked)
- `experiments`: Experiment designs/results
- `releases`: Release notes

---

### 13. Product Document Version (`product_document_versions`)
**Purpose**: Immutable history of document changes.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `document_id` | UUID (FK→product_documents) | |
| `content` | TEXT | Markdown snapshot |
| `created_by` | UUID (FK→profiles) | |
| `created_at` | TIMESTAMPTZ | |

**UI**: Version inspector with revert capability

---

### 14. Workflow Run (`workflow_runs`)
**Purpose**: Execution trace for PMOS workflow runs.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `product_slug` | TEXT | Product slug at time of run |
| `decision_id` | UUID (FK→decisions, NULLABLE) | For PRD workflow |
| `goal` | TEXT | `prd` \| `one-pager` \| `experiment` \| `launch` \| `update` \| `decision-log` \| `meeting-actions` \| `what-if` |
| `skill_name` | TEXT | Matched skill (e.g., `prd-writer`) |
| `status` | TEXT | `pending` \| `running` \| `completed` \| `failed` \| `cancelled` |
| `steps` | JSONB | Array of `{ label, status }` |
| `artifact_id` | UUID (FK→pmos_artifacts, NULLABLE) | Output artifact |
| `error_message` | TEXT | If failed |
| `user_brief` | TEXT | User input |
| `created_by` | UUID (FK→profiles) | |
| `reviewed_at` | TIMESTAMPTZ | When marked reviewed |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

---

### 15. PMOS Artifact (`pmos_artifacts`)
**Purpose**: Generated artifacts from workflows with traceability.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `product_slug` | TEXT | |
| `skill_name` | TEXT | Skill that generated it |
| `artifact_type` | TEXT | Matches goal |
| `title` | TEXT | |
| `content` | TEXT | Markdown output |
| `citations` | JSONB | Source file paths / DB refs |
| `trace` | JSONB | Execution trace (skill, context files, validation result) |
| `created_by` | UUID (FK→profiles) | |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

**Validation**: `validateArtifact(type, content)` checks required sections per artifact type

---

### 16. AI Conversation (`ai_conversations`)
**Purpose**: Chat sessions (global or decision-scoped).

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `decision_id` | UUID (FK→decisions, NULLABLE) | NULL = global chat |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

---

### 17. AI Message (`ai_messages`)
**Purpose**: Individual messages in a conversation.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `conversation_id` | UUID (FK→ai_conversations) | |
| `role` | `message_role` ENUM | `user` \| `assistant` |
| `content` | TEXT | |
| `created_at` | TIMESTAMPTZ | |

**UI**: Streaming SSE from `ai-assistant` edge function, persisted via `useConversations` hook

---

### 18. Activity Log (`activity_log`)
**Purpose**: Audit trail of workspace actions.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `user_id` | UUID (FK→profiles) | |
| `action` | TEXT | e.g., `created_decision`, `updated_evidence` |
| `entity_type` | TEXT | `decision` \| `feedback` \| etc. |
| `entity_id` | UUID | |
| `metadata` | JSONB | Additional context |
| `created_at` | TIMESTAMPTZ | |

---

### 19. Workspace Invitation (`workspace_invitations`)
**Purpose**: Email-based member invitation flow.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `email` | TEXT | Invitee email |
| `role` | `workspace_role` ENUM | `full_access` \| `view_access` (NOT owner) |
| `invited_by` | UUID (FK→profiles) | |
| `status` | `invitation_status` ENUM | `pending` \| `accepted` \| `declined` |
| `created_at` | TIMESTAMPTZ | |
| `expires_at` | TIMESTAMPTZ | |

**Acceptance**: Creates `workspace_memberships` row, marks invitation `accepted`

---

### 20. AI Rule (`ai_rules`)
**Purpose**: Workspace/product-scoped rules injected into agent prompts.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `product_id` | UUID (FK→products, NULLABLE) | NULL = workspace-wide |
| `name` | TEXT | Rule name |
| `body` | TEXT | Rule content (injected into prompt) |
| `activation` | TEXT | `always` \| `agent_requested` \| `manual` |
| `created_by` | UUID (FK→profiles) | |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

**Injection**: `loadRules()` in pipeline - `always` + `agent_requested` auto-included (1.5KB budget)

---

### 21. Customer Insight (`customer_insights`)
**Purpose**: Synthesized insights from feedback analysis.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `product_id` | UUID (FK→products, NULLABLE) | |
| `theme` | TEXT | Insight theme |
| `summary` | TEXT | Description |
| `sentiment` | `insight_sentiment` ENUM | `positive` \| `neutral` \| `negative` |
| `evidence_ids` | UUID[] | Supporting `feedback_items` |
| `segment` | TEXT | Customer segment |
| `period_start` | TIMESTAMPTZ | |
| `period_end` | TIMESTAMPTZ | |
| `supporting_metrics` | JSONB | Metric snapshots |
| `created_by` | UUID (FK→profiles) | |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

---

### 22. Feature Recommendation (`feature_recommendations`)
**Purpose**: AI-suggested features with evidence backing.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `product_id` | UUID (FK→products, NULLABLE) | |
| `title` | TEXT | |
| `rationale` | TEXT | Why this feature |
| `evidence_ids` | UUID[] | Supporting feedback |
| `problem_ids` | UUID[] | Related problems |
| `effort` | `rec_effort` ENUM | `low` \| `medium` \| `high` |
| `expected_impact` | `rec_impact` ENUM | `low` \| `medium` \| `high` |
| `status` | `recommendation_status` ENUM | `suggested` \| `accepted` \| `dismissed` |
| `decision_id` | UUID (FK→decisions, NULLABLE) | If promoted to decision |
| `created_by` | UUID (FK→profiles) | |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

---

### 23. Product Metric (`product_metrics`)
**Purpose**: Time-series metric data for products/features.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `product_id` | UUID (FK→products) | |
| `feature_id` | UUID (FK→product_features, NULLABLE) | |
| `metric_key` | TEXT | e.g., `activation_rate`, `retention_d7` |
| `metric_value` | NUMERIC | |
| `unit` | TEXT | `%`, `count`, `seconds`, etc. |
| `period_start` | DATE | |
| `period_end` | DATE | |
| `segment` | TEXT | Optional segment |
| `source` | `metric_source` ENUM | `manual` \| `csv` \| `mixpanel` \| `amplitude` |
| `notes` | TEXT | |
| `created_by` | UUID (FK→profiles) | |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

**Index**: `(workspace_id, product_id, metric_key, period_end DESC)`

---

### 24. Decision Outcome (`decision_outcomes`)
**Purpose**: Post-implementation outcome tracking for learning.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `decision_id` | UUID (FK→decisions, UNIQUE) | One per decision |
| `workspace_id` | UUID (FK→workspaces) | |
| `product_id` | UUID (FK→products, NULLABLE) | |
| `outcome_status` | `decision_outcome_status` ENUM | `shipped` \| `cancelled` \| `reversed` \| `partial` |
| `shipped_at` | TIMESTAMPTZ | |
| `actual_impact` | `actual_impact_level` ENUM | `low` \| `medium` \| `high` \| `unknown` |
| `actual_metrics` | JSONB | Realized metric changes |
| `lessons` | TEXT | Learnings |
| `predicted_confidence` | NUMERIC | From confidence_scores |
| `predicted_impact` | TEXT | From decision analysis |
| `accuracy_score` | NUMERIC | Calibration metric |
| `created_by` | UUID (FK→profiles) | |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

---

### 25. Roadmap Initiative (`roadmap_initiatives`)
**Purpose**: Structured roadmap items linked to decisions.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `product_id` | UUID (FK→products) | |
| `title` | TEXT | |
| `description` | TEXT | |
| `status` | `initiative_status` ENUM | `proposed` \| `planned` \| `in_progress` \| `shipped` \| `paused` \| `cancelled` |
| `priority` | `initiative_priority` ENUM | `low` \| `medium` \| `high` \| `critical` |
| `target_quarter` | TEXT | e.g., `2026-Q3` |
| `linked_decisions` | UUID[] | Array of `decisions.id` |
| `created_by` | UUID (FK→profiles) | |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

**Index**: GIN on `linked_decisions` for reverse lookup

---

### 26. Rate Limit (`rate_limits`)
**Purpose**: Per-user/endpoint rate limiting for edge functions.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `user_id` | UUID | |
| `ip_address` | TEXT | Fallback |
| `endpoint` | TEXT | Function name |
| `created_at` | TIMESTAMPTZ | |

**Cleanup**: Queries use 1-minute window (`gte created_at`)

---

### 27. Decision Trace (`decision_traces`)
**Purpose**: Agentic pipeline run metadata.

| Field | Type | Description |
|-------|------|-------------|
| `trace_id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `decision_id` | UUID (FK→decisions) | |
| `user_id` | UUID (FK→profiles) | |
| `status` | TEXT | `ok` \| `insufficient_data` \| `error` \| `started` |
| `degraded_mode` | BOOLEAN | Memory/impact degradation |
| `request` | JSONB | Full request payload |
| `response_payload` | JSONB | Full response |
| `error` | JSONB | Error details if failed |
| `duration_ms` | INTEGER | |
| `created_at` | TIMESTAMPTZ | |

---

### 28. Decision Agent Output (`decision_agent_outputs`)
**Purpose**: Per-agent outputs for replay/debug.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `trace_id` | UUID (FK→decision_traces) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `agent` | TEXT | `critic` \| `evidence` \| `impact` \| `feature` \| `planner` \| `risk` \| `documentation` |
| `stage` | TEXT | `parallel` \| `sequential` |
| `status` | TEXT | `ok` \| `insufficient_data` \| `error` |
| `confidence_score` | NUMERIC | |
| `output` | JSONB | Full agent output |
| `created_at` | TIMESTAMPTZ | |

---

### 29. Decision Embedding (`decision_embeddings`)
**Purpose**: Vector embeddings for similar-decision retrieval.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `decision_id` | UUID (FK→decisions) | |
| `product_id` | UUID (FK→products, NULLABLE) | |
| `content` | TEXT | Title + description |
| `embedding` | VECTOR(1536) | OpenAI-compatible |
| `created_at` | TIMESTAMPTZ | |

**Function**: `match_decision_embeddings(workspace_id, embedding, k)` for retrieval

---

### 30. Product Document Section (`product_doc_sections`)
**Purpose**: Section-level embeddings for targeted retrieval.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `product_id` | UUID (FK→products) | |
| `document_id` | UUID (FK→product_documents) | |
| `doc_type` | TEXT | `roadmap` \| `metrics` \| `updates` |
| `heading` | TEXT | Section heading |
| `content` | TEXT | Section body |
| `embedding` | VECTOR(1536) | |
| `token_count` | INTEGER | |
| `created_at` | TIMESTAMPTZ | |

**Function**: `match_product_doc_sections(workspace_id, product_id, doc_types, query_embedding, k)`

---

### 31. Decision Session Memory (`decision_session_memory`)
**Purpose**: 24hr TTL memory for re-analysis continuity.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `decision_id` | UUID (FK→decisions) | |
| `trace_id` | UUID (FK→decision_traces) | |
| `payload` | JSONB | `{ summary, features, risks, alignment, degraded? }` |
| `version` | INTEGER | Incremented per run |
| `created_at` | TIMESTAMPTZ | |

**Read**: `readSessionMemory()` enforces 24hr TTL

---

### 32. Decision Edge (`decision_edges`)
**Purpose**: Knowledge graph edges between decisions.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID (PK) | |
| `workspace_id` | UUID (FK→workspaces) | |
| `source_decision_id` | UUID (FK→decisions) | |
| `target_decision_id` | UUID (FK→decisions) | |
| `edge_type` | TEXT | `similar` \| `supersedes` \| `related` |
| `weight` | NUMERIC | Similarity score |
| `created_at` | TIMESTAMPTZ | |

---

## Enums Reference

| Enum | Values |
|------|--------|
| `workspace_role` | `owner`, `full_access`, `view_access` |
| `invitation_status` | `pending`, `accepted`, `declined` |
| `feedback_type` | `interview`, `note`, `quote`, `insight` |
| `evidence_strength` | `strong`, `moderate`, `weak` |
| `confidence_level` | `high`, `medium`, `low` |
| `decision_type` | `enhancement`, `fix`, `replacement`, `net-new`, `defer` |
| `problem_status` | `identified`, `validated`, `addressed` |
| `decision_status` | `draft`, `pending`, `decided`, `implemented` |
| `effort_level` | `low`, `medium`, `high` |
| `impact_level` | `low`, `medium`, `high` |
| `message_role` | `user`, `assistant` |
| `initiative_status` | `proposed`, `planned`, `in_progress`, `shipped`, `paused`, `cancelled` |
| `initiative_priority` | `low`, `medium`, `high`, `critical` |
| `insight_sentiment` | `positive`, `neutral`, `negative` |
| `recommendation_status` | `suggested`, `accepted`, `dismissed` |
| `rec_effort` | `low`, `medium`, `high` |
| `rec_impact` | `low`, `medium`, `high` |
| `metric_source` | `manual`, `csv`, `mixpanel`, `amplitude` |
| `decision_outcome_status` | `shipped`, `cancelled`, `reversed`, `partial` |
| `actual_impact_level` | `low`, `medium`, `high`, `unknown` |
| `agent_status` | `ok`, `insufficient_data`, `error` |
| `risk_level` | `low`, `medium`, `high`, `critical` |

---

## Key Flows Detail

### Feedback → Problem → Decision Flow
1. **Capture Feedback** (`/feedback` or `/p/:slug/feedback`)
   - Manual create or CSV import
   - Product-scoped routes auto-set `product_id`
2. **Create/Import Features** (`/features` or `/p/:slug/features`)
   - Module grouping for organization
   - CSV import supported
3. **Create Problem** (`/problems`)
   - Links feedback (`linked_feedback`) and features (`related_features`)
   - Evidence strength assessment
4. **Recalculate Mappings** (Problem card action)
   - Calls RPC `recalculate_problem_mappings(problem_id, threshold)`
   - Updates `problem_evidence_map` and `problem_feature_map`
5. **Create Decision** (`/decisions` → New)
   - Selects problem, sets owner, status=draft
6. **Generate Options** (Decision detail → Generate Options)
   - Calls `generate-decision-options` edge function
   - Returns 3 structured options (1 must be `defer`)
7. **Add Evidence** (Decision detail → Evidence tab)
   - Manual or auto-seeded from problem mappings
   - Typed artifacts with reliability dimensions
8. **Calculate Confidence** (Decision detail → Score)
   - Calls `calculate-decision-score` (deterministic) OR
   - Runs `decision-analyze` (agentic) → `writeUnifiedScore`
9. **PRD Gate** (Decision detail → Generate PRD)
   - Reads `confidence_scores.overall_confidence`
   - Blocks if ≤ 0.65
   - Idempotent: returns existing PRD
10. **Generate PRD** (if gate passed)
    - Calls `generate-prd` edge function
    - Saves to `decisions.prd_content` + `prd_generated=true`

---

### Agentic Pipeline (`decision-analyze`)
```
1. Normalize & Validate Request (Zod schema)
2. Rate Limit Check (10/min/user)
3. Workspace Authorization
4. Memory Retrieval:
   - Similar past decisions (vector search)
   - Patterns & failures
   - Product knowledge (brief + top-K sections)
   - Session memory (24hr TTL)
   - Prior agent outputs (last 7)
   - AI rules (always + agent_requested)
5. Build Envelope (trace_id + all context)
6. PARALLEL Stage:
   - Critic Agent → clarity, assumptions, contradictions
   - Evidence Agent → confidence, gaps, signals
   - Impact Agent → conversion/retention deltas
7. Cross-Agent Guard:
   - Alignment score (critic clarity vs evidence confidence)
   - If critic/evidence insufficient → SAFE INSUFFICIENT RESPONSE
8. SEQUENTIAL Stage:
   - Feature Agent → feature variants + experiments
   - Planner Agent → epics, stories, dependencies, execution order
   - Risk Agent → categorized risks with likelihood×severity
   - Documentation Agent → PRD payload, decision log, stakeholder summary
9. Score Engine:
   - Evidence strength (input + agent)
   - Agent confidence average
   - Alignment score
   - Decision quality (clarity + impact + feasibility)
   - Risk level (max likelihood×severity)
10. Final Synthesis → FinalResponse schema
11. Persist:
    - Trace + all agent outputs
    - Session memory (v+1)
    - Unified score mirror (decisions.confidence_score + confidence_scores)
12. Return response
```

---

## RBAC Matrix

| Action | owner | full_access | view_access |
|--------|-------|-------------|-------------|
| View workspace data | ✅ | ✅ | ✅ |
| Create/edit feedback | ✅ | ✅ | ❌ |
| Create/edit features | ✅ | ✅ | ❌ |
| Create/edit problems | ✅ | ✅ | ❌ |
| Create/edit decisions | ✅ | ✅ | ❌ |
| Add/edit evidence | ✅ | ✅ | ❌ |
| Calculate confidence | ✅ | ✅ | ❌ |
| Generate PRD | ✅ | ✅ | ❌ |
| Run workflows | ✅ | ✅ | ❌ |
| Manage members/invites | ✅ | ✅ | ❌ |
| Workspace settings | ✅ | ✅ | ❌ |
| Delete workspace | ✅ | ❌ | ❌ |
| Transfer ownership | ✅ | ❌ | ❌ |

**Enforcement**:
- UI: `canEdit(role)`, `canDelete(role)`, `canInvite(role)`, `canManageMembers(role)` from `src/types/workspace.ts`
- RLS: `has_workspace_edit_access(workspace_id)` for mutations, `has_workspace_access(workspace_id)` for reads
- Edge functions: Validate JWT → workspace membership → role check

---

## Edge Functions Reference

| Function | Method | Auth | Rate Limit | Key Behavior |
|----------|--------|------|------------|--------------|
| `decision-analyze` | POST | Bearer JWT | 10/min/user | 7-agent pipeline, trace persistence |
| `calculate-decision-score` | POST | Bearer JWT | 20/min/user+IP | Deterministic scoring, backfills evidence |
| `generate-prd` | POST | Bearer JWT | 5/min | Gate: confidence>0.65, idempotent |
| `generate-decision-options` | POST | Bearer JWT | 10/min | 3 options via tool calling |
| `detect-data-gaps` | POST | Bearer JWT | - | Gap analysis from artifacts |
| `ai-assistant` | POST | Bearer JWT | 20/min | Streaming SSE, context injection |
| `recommend-features` | POST | Bearer JWT | - | AI feature suggestions |
| `prioritize-roadmap` | POST | Bearer JWT | - | Roadmap prioritization |
| `ingest-metrics` | POST | Bearer JWT | - | CSV/manual metrics import |
| `draft-decision-update` | POST | Bearer JWT | - | Stakeholder update draft |
| `embed-product-sections` | POST | Bearer JWT | - | Section embeddings for retrieval |
| `upsert-decision-embedding` | POST | Bearer JWT | - | Decision vector for similarity |
| `record-decision-outcome` | POST | Bearer JWT | - | Outcome tracking |
| `synthesize-insights` | POST | Bearer JWT | - | Cross-product insights |

All functions:
- Validate Bearer JWT via `supabase.auth.getClaims()`
- Check workspace access via RLS-respecting client
- Use `service_role` for writes bypassing RLS
- Structured error responses with trace_id

---

## Skills System

### Skill Definition (`docs/skills/{name}/SKILL.md`)
```markdown
---
name: skill-name
description: "When to use trigger phrases"
---

# Skill Title

## When to Use
- Trigger 1
- Trigger 2

## Process
### Step 1: ...
### Step 2: ...

## Output
Expected output format
```

### Registered Skills (18)
| Skill | Purpose |
|-------|---------|
| `prd-writer` | Generate PRDs |
| `one-pager` | Executive summaries |
| `experiment-designer` | Experiment design |
| `launch-readiness` | Launch planning |
| `stakeholder-update` | Status updates |
| `decision-logger` | Decision records |
| `meeting-to-actions` | Meeting → action items |
| `what-if` | Impact simulation |
| `strategy-connector` | Strategy mapping |
| `writing-clearly` | Prose editing |
| `working-backwards` | PR/FAQ from future state |
| ... | (8 more) |

### Router (`src/lib/pmos/skillRouter.ts`)
- Loads all skills at build via `import.meta.glob`
- Tokenizes query + skill metadata (name, description, whenToUse)
- Jaccard similarity + name match boost
- Thresholds: SELECT ≥ 0.75, FALLBACK < 0.5
- Returns: `{ selectedSkill, confidence, top3, fallbackPlan? }`

---

## Product Documentation & Knowledge Injection

### Document Types
| Type | Purpose | Injected in Pipeline |
|------|---------|---------------------|
| `brief` | Problem, hypothesis, success metrics | **Always** (whole doc, anchor) |
| `roadmap` | Now/Next/Later, milestones | Top-K sections by similarity |
| `metrics` | Key metric snapshots | Top-K sections + latest 25 `product_metrics` rows |
| `updates` | Status updates | Top-K sections |
| `experiments` | Experiment designs | On-demand |
| `releases` | Release notes | On-demand |

### Retrieval Budget
- Total: 6KB
- Brief: whole document (priority)
- Others: Top-3 sections per type (max 6 sections total)
- Section embeddings created lazily on first retrieval

---

## Observability & Debugging

### Trace Replay UI (`/decisions/trace/:traceId`)
- Run summary (status, degraded, duration)
- Parallel stage outputs (critic, evidence, impact)
- Sequential stage outputs (feature, planner, risk, documentation)
- Final response payload
- Outcome vs prediction (if `decision_outcomes` exists)

### Agent Output Persistence
Every agent output stored in `decision_agent_outputs` with:
- Agent name, stage, status
- Confidence score
- Full JSON output
- Data sources cited
- Duration (parallel stage)

### Logging
`log(level, event, metadata)` → Supabase logs with correlation IDs

---

## Known Gaps (from requirements)

| ID | Gap | Impact |
|----|-----|--------|
| GAP-01 | Problem→Decision navigation state not fully consumed in prefill | UX friction |
| GAP-02 | Role-gated CTAs appear before server rejection | Inconsistent UX |
| GAP-03 | Dual AI invocation patterns (raw URL vs `supabase.functions.invoke`) | Maintenance burden |
| GAP-04 | Two scoring paths (agentic vs deterministic) not reconciled | Conflicting confidence values |
| GAP-05 | Memory tables may not be fully wired in active pipeline | Incomplete context |
| GAP-06 | Export excludes PMOS artifacts, AI conversations | Incomplete data export |

---

## Environment Variables

```env
# Frontend (Vite)
VITE_SUPABASE_URL=https://xxx.supabase.co
VITE_SUPABASE_PUBLISHABLE_KEY=eyJ...
VITE_AGENTIC_ANALYSIS_ENABLED=true

# Edge Functions (Supabase secrets)
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...
LOVABLE_API_KEY=sk-...
```

---

## Test Structure

```
src/test/
├── setup.ts                    # Vitest config
├── pmos/
│   ├── validators.test.ts      # Artifact validation
│   ├── skillRouter.test.ts     # Routing logic
│   └── pathResolver.test.ts    # Knowledge paths
└── agentic/
    ├── requestSchema.test.ts   # Zod validation
    ├── normalizeRequest.test.ts
    ├── featureDedup.test.ts
    └── evidenceMap.test.ts
```

Run: `npm run test` (vitest)

---

## Implementation Sequence (from requirements)

1. Tenant model + auth + workspace membership + RBAC baseline
2. Product context routing and shell/navigation
3. Core CRUD entities: feedback, features, problems, decisions/options
4. Evidence model and scoring pipeline
5. PRD and decision option generation
6. Product docs/versioning
7. Invitations, admin, export, activity logging
8. Global AI assistant
9. Agentic multi-agent orchestration with strict schemas and observability
10. Insights and polish

---

## Definition of Done (System Level)

- All functional requirements F1-F16 implemented
- API contracts and schema validation enforced
- Role and tenant isolation verified by test suite
- Core workflows testable end-to-end
- Observability traces available for analysis pipeline
- Documentation and runbooks updated for operations