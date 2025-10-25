# ADR-0022: Goal Tracking Persistence Strategy

**Status**: ✅ Accepted
**Date**: 2025-10-24
**Scope**: Vertical 4.2 (affects goal storage and querying)

---

## Context

Financial goals (savings targets, income milestones, debt payoff, budget adherence) need persistent storage with flexible schema to accommodate different goal types.

**The challenge:**

```
Goal Creation → Storage → Progress Tracking → Alerts
```

**Goal types have different metadata:**
- **Savings**: `target_amount`, `current_amount`, `monthly_contribution`
- **Debt Payoff**: `principal`, `interest_rate`, `payment_schedule`, `payoff_date`
- **Budget**: `category`, `monthly_limit`, `alert_threshold`
- **Income**: `target_income`, `current_income`, `timeframe`

**Requirements:**
- Query all goals for a user (`GET /users/:id/goals`)
- Query overdue goals across all users (for alert system)
- Query goals by type (`GET /users/:id/goals?type=debt`)
- Query within goal metadata (`WHERE metadata->>'interest_rate' > 5.0`)
- Add new goal types without database migrations
- Maintain relational integrity (user_id foreign key)

**Trade-off space:**
- **Flexibility** vs **Type safety**: Fully normalized schema enforces constraints but requires migrations for new goal types
- **Queryability** vs **Schema-less**: NoSQL provides flexibility but sacrifices relational queries
- **Performance** vs **Flexibility**: JSONB slightly slower than native columns but provides schema evolution

---

## Decision

**We use PostgreSQL table with JSONB metadata column for goal storage.**

```sql
CREATE TABLE goals (
    goal_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    goal_type VARCHAR(50) NOT NULL,  -- 'savings', 'debt', 'budget', 'income'
    target_amount DECIMAL(15,2),     -- NULL for non-monetary goals
    deadline DATE,                    -- NULL for ongoing goals
    status VARCHAR(20) NOT NULL,     -- 'active', 'completed', 'overdue', 'archived'
    metadata JSONB NOT NULL,         -- Goal-type-specific data
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_goals_user_id ON goals(user_id);
CREATE INDEX idx_goals_status ON goals(status);
CREATE INDEX idx_goals_metadata ON goals USING GIN (metadata);
```

- **Structured columns**: Common fields (user_id, name, target_amount, deadline, status)
- **JSONB column**: Goal-type-specific metadata (flexible schema)
- **Validation**: Zod schemas per goal type, enforced in application layer
- **Indexes**: GIN index on JSONB for fast metadata queries

---

## Rationale

### 1. Flexibility for Goal Types
- JSONB column stores goal-type-specific data without schema changes
- Adding new goal type: Just add Zod schema, no database migration
- Example metadata structures:
  ```json
  // Debt goal
  {
    "principal": 50000,
    "interest_rate": 4.5,
    "payment_schedule": "monthly",
    "minimum_payment": 1200
  }

  // Savings goal
  {
    "current_amount": 12000,
    "monthly_contribution": 500,
    "auto_transfer": true
  }
  ```

### 2. Queryability Across Goals
- Query all user's goals: `SELECT * FROM goals WHERE user_id = ?`
- Query overdue goals: `SELECT * FROM goals WHERE status = 'overdue'`
- Query by type: `SELECT * FROM goals WHERE goal_type = 'debt'`
- Query within metadata: `SELECT * FROM goals WHERE metadata->>'interest_rate'::float > 5.0`
- GIN index makes JSONB queries fast (<100ms for user queries)

### 3. Performance with Indexes
- GIN index on JSONB: O(log n) for containment queries (`metadata @> '{"auto_transfer": true}'`)
- Composite index on (user_id, status) for dashboard queries
- Materialized view for aggregations (refreshed hourly):
  ```sql
  CREATE MATERIALIZED VIEW goal_progress_summary AS
  SELECT user_id, goal_type, COUNT(*) as total,
         AVG(progress_percentage) as avg_progress
  FROM goals
  GROUP BY user_id, goal_type;
  ```

### 4. Relational Integrity
- Foreign key to users table: `ON DELETE CASCADE` cleans up orphaned goals
- Can join with transactions for progress tracking:
  ```sql
  SELECT g.*, SUM(t.amount) as contributed
  FROM goals g
  LEFT JOIN transactions t ON t.user_id = g.user_id
      AND t.category = g.metadata->>'linked_category'
  WHERE g.goal_id = ?
  ```

### 5. Schema Evolution Without Downtime
- Add new field to metadata: Just update Zod schema, no migration
- Backward compatible: Existing goals without new field still valid
- Versioning: Can add `metadata_version` field for major schema changes

---

## Consequences

### ✅ Positive

- **Flexible schema**: Add new goal types (e.g., "investment", "tax_savings") without migrations
- **Fast queries**: <100ms for user's goals, <500ms for cross-user aggregations (with indexes)
- **Relational integrity**: Foreign key constraints prevent orphaned goals
- **Type safety in app**: Zod schemas validate metadata before INSERT/UPDATE
- **Analytics-friendly**: Can build dashboards with PostgreSQL aggregations

### ⚠️ Trade-offs

- **JSONB validation in app**: Database can't enforce JSONB structure (need runtime validation)
- **Slightly slower than normalized**: JSONB queries 10-20% slower than native columns (acceptable trade-off)
- **Index bloat**: GIN indexes larger than B-tree (monitored via `pg_relation_size`)

### 🔴 Risks (Mitigated)

- **Risk**: Inconsistent metadata structure (typos, missing fields)
  - **Mitigation**: Strict Zod validation before DB write, unit tests for all goal types
- **Risk**: JSONB query performance degradation with large datasets
  - **Mitigation**: Materialized views for dashboards, partition table by user_id if >10M goals
- **Risk**: Schema evolution breaks old goals
  - **Mitigation**: `metadata_version` field, migration script to backfill missing fields

---

## Alternatives Considered

### Alternative A: Fully Normalized Schema (Separate Tables per Goal Type) (Rejected)

**Approach:**
```sql
CREATE TABLE savings_goals (...);
CREATE TABLE debt_goals (...);
CREATE TABLE budget_goals (...);
```

**Pros:**
- **Type safety**: Database enforces column constraints
- **Slightly faster**: Native columns 10-20% faster than JSONB

**Cons:**
- **Inflexible**: Every new goal type requires migration
- **Hard to query across types**: Can't easily `SELECT * FROM all_goals`
- **Code duplication**: Need separate DAO/repository for each goal type
- **JOIN complexity**: Dashboards require `UNION` across multiple tables

### Alternative B: JSON in User Settings (NoSQL Approach) (Rejected)

**Approach:**
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    settings JSONB  -- Contains {"goals": [...]}
);
```

**Pros:**
- **Simple**: All user data in one table
- **Fast for single user**: No JOIN needed

**Cons:**
- **Not queryable across users**: Can't find "all overdue goals" without full table scan
- **No relational integrity**: Can't foreign key to transactions
- **Scaling issues**: `users` table becomes bottleneck (hot rows)
- **Backup complexity**: Can't selectively backup goals

### Alternative C: Event Sourcing (Goal Events Stored, State Derived) (Rejected)

**Approach:**
```sql
CREATE TABLE goal_events (
    event_id UUID PRIMARY KEY,
    goal_id UUID,
    event_type VARCHAR(50),  -- 'created', 'updated', 'progress', 'completed'
    event_data JSONB
);
```

**Pros:**
- **Full audit trail**: Can replay history
- **Time-travel queries**: Can reconstruct goal state at any point

**Cons:**
- **Overkill for simple CRUD**: Goals don't need complex event sourcing
- **Replay complexity**: Current state requires aggregating all events (slow)
- **Storage overhead**: 10× more storage than current-state table
- **Query complexity**: Every read requires event aggregation

### Alternative D: MongoDB (Document Database) (Rejected)

**Pros:**
- **Flexible schema**: Native JSON documents
- **Fast writes**: No transaction overhead

**Cons:**
- **Infrastructure complexity**: Adds MongoDB cluster to tech stack
- **No relational queries**: Can't JOIN with users or transactions
- **Weaker consistency**: No ACID guarantees (eventual consistency)
- **PostgreSQL JSONB provides same flexibility**: No need for separate database

---

## Implementation Notes

### Table Creation

```sql
CREATE TABLE goals (
    goal_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    goal_type VARCHAR(50) NOT NULL CHECK (goal_type IN ('savings', 'debt', 'budget', 'income')),
    target_amount DECIMAL(15,2),
    deadline DATE,
    status VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'completed', 'overdue', 'archived')),
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_goals_user_id ON goals(user_id);
CREATE INDEX idx_goals_status ON goals(status);
CREATE INDEX idx_goals_type ON goals(goal_type);
CREATE INDEX idx_goals_metadata ON goals USING GIN (metadata);
CREATE INDEX idx_goals_deadline ON goals(deadline) WHERE status = 'active';
```

### Zod Validation (TypeScript)

```typescript
// Base goal schema
const baseGoalSchema = z.object({
  goal_id: z.string().uuid(),
  user_id: z.string().uuid(),
  name: z.string().min(1).max(255),
  goal_type: z.enum(['savings', 'debt', 'budget', 'income']),
  target_amount: z.number().positive().nullable(),
  deadline: z.date().nullable(),
  status: z.enum(['active', 'completed', 'overdue', 'archived']),
});

// Debt goal metadata
const debtMetadataSchema = z.object({
  principal: z.number().positive(),
  interest_rate: z.number().min(0).max(100),
  payment_schedule: z.enum(['monthly', 'biweekly', 'weekly']),
  minimum_payment: z.number().positive(),
});

// Savings goal metadata
const savingsMetadataSchema = z.object({
  current_amount: z.number().min(0),
  monthly_contribution: z.number().positive(),
  auto_transfer: z.boolean(),
  linked_account_id: z.string().uuid().optional(),
});

// Full goal schema with discriminated union
const goalSchema = z.discriminatedUnion('goal_type', [
  baseGoalSchema.extend({
    goal_type: z.literal('debt'),
    metadata: debtMetadataSchema,
  }),
  baseGoalSchema.extend({
    goal_type: z.literal('savings'),
    metadata: savingsMetadataSchema,
  }),
  // ... other goal types
]);
```

### Materialized View for Dashboard

```sql
CREATE MATERIALIZED VIEW goal_progress_summary AS
SELECT
    user_id,
    goal_type,
    COUNT(*) as total_goals,
    COUNT(*) FILTER (WHERE status = 'completed') as completed_goals,
    COUNT(*) FILTER (WHERE status = 'overdue') as overdue_goals,
    AVG(CASE
        WHEN target_amount IS NOT NULL AND target_amount > 0
        THEN ((metadata->>'current_amount')::float / target_amount) * 100
        ELSE NULL
    END) as avg_progress_percentage
FROM goals
WHERE status != 'archived'
GROUP BY user_id, goal_type;

CREATE UNIQUE INDEX ON goal_progress_summary(user_id, goal_type);

-- Refresh hourly via cron job
REFRESH MATERIALIZED VIEW CONCURRENTLY goal_progress_summary;
```

### Progress Tracking Query

```sql
-- Calculate debt goal progress
SELECT
    g.goal_id,
    g.name,
    g.metadata->>'principal' as original_principal,
    COALESCE(SUM(t.amount), 0) as total_paid,
    (g.metadata->>'principal')::float - COALESCE(SUM(t.amount), 0) as remaining
FROM goals g
LEFT JOIN transactions t ON
    t.user_id = g.user_id
    AND t.metadata->>'linked_goal_id' = g.goal_id::text
WHERE g.goal_type = 'debt' AND g.status = 'active'
GROUP BY g.goal_id;
```

---

## Related Decisions

- **ADR-0021**: Projection algorithm (forecasts used to estimate goal completion dates)
- **ADR-0023**: Forecast caching (goal progress cached for 24h)
- **Future**: May add `goal_milestones` table for sub-goals (e.g., "Save $5k by March, $10k by June")

---

**References:**
- Vertical 4.2: [docs/verticals/4.2-forecast.md](../verticals/4.2-forecast.md)
- Primitive: [docs/primitives/ol/GoalStore.md](../primitives/ol/GoalStore.md)
- Schema: [docs/schemas/goal-config.schema.json](../schemas/goal-config.schema.json)
- PostgreSQL JSONB Documentation: https://www.postgresql.org/docs/current/datatype-json.html
