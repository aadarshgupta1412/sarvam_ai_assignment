# Question 3: Analytics and Evaluation Query Strategy

## Key Analytics Requirements

For our conversational AI platform, we need focused analytics capabilities for both business intelligence and model evaluation. Taking inspiration from specialized vendors like Langfuse and Arize, I've identified the most critical queries we should prioritize.

## Top 6 Essential Queries

### 1. User Engagement Dashboard

```sql
-- Daily active users and session metrics
SELECT 
  DATE(created_at) as date,
  COUNT(DISTINCT account_uid) as daily_active_users,
  COUNT(*) as total_sessions,
  AVG(EXTRACT(EPOCH FROM (
    SELECT MAX(created_at) - MIN(created_at) 
    FROM human_turns 
    WHERE session_uid = s.uid
  ))) as avg_session_duration_seconds
FROM sessions s
WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
  AND deleted_at IS NULL
GROUP BY DATE(created_at)
ORDER BY date DESC;
```

**PostgreSQL Optimization:**
- Create index: `CREATE INDEX idx_sessions_analytics ON sessions(created_at, account_uid) WHERE deleted_at IS NULL;`
- Create materialized view refreshed daily

**ScyllaDB Approach:**
```cql
CREATE TABLE daily_user_metrics (
  date text,                     -- YYYY-MM-DD format
  account_uid text,
  session_count counter,
  total_duration_seconds counter,
  PRIMARY KEY (date, account_uid)
);
```

### 2. Response Quality Tracking

```sql
-- User satisfaction metrics
SELECT 
  DATE(at.created_at) as date,
  COUNT(*) as total_responses,
  COUNT(CASE WHEN at.liked = true THEN 1 END) as liked_responses,
  COUNT(CASE WHEN at.liked = false THEN 1 END) as disliked_responses,
  ROUND(100.0 * COUNT(CASE WHEN at.liked = true THEN 1 END) / 
    NULLIF(COUNT(CASE WHEN at.liked IS NOT NULL THEN 1 END), 0), 2) as satisfaction_rate
FROM agent_turns at
WHERE at.created_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY DATE(at.created_at)
ORDER BY date DESC;
```

**PostgreSQL Optimization:**
- Create index: `CREATE INDEX idx_agent_turns_satisfaction ON agent_turns(created_at, liked);`

**ScyllaDB Approach:**
```cql
CREATE TABLE response_quality_metrics (
  date text,
  liked_count counter,
  disliked_count counter,
  total_count counter,
  PRIMARY KEY (date)
);
```

### 3. Token Usage and Cost Analysis

```sql
-- Token usage by model
SELECT 
  DATE(st.created_at) as date,
  JSON_EXTRACT_PATH_TEXT(st.content, 'model') as model,
  SUM(CAST(JSON_EXTRACT_PATH_TEXT(st.content, 'input_tokens') AS INTEGER)) as input_tokens,
  SUM(CAST(JSON_EXTRACT_PATH_TEXT(st.content, 'output_tokens') AS INTEGER)) as output_tokens,
  SUM(CAST(JSON_EXTRACT_PATH_TEXT(st.content, 'cost_usd') AS DECIMAL(10,6))) as total_cost_usd
FROM steps st
WHERE st.created_at >= CURRENT_DATE - INTERVAL '30 days'
  AND JSON_EXTRACT_PATH_TEXT(st.content, 'model') IS NOT NULL
GROUP BY date, model
ORDER BY date DESC, total_cost_usd DESC;
```

**PostgreSQL Optimization:**
- Create index: `CREATE INDEX idx_steps_model ON steps((content->>'model')) INCLUDE ((content->>'input_tokens'), (content->>'output_tokens'), (content->>'cost_usd'));`

**ScyllaDB Approach:**
```cql
CREATE TABLE token_metrics (
  date text,
  model text,
  input_tokens counter,
  output_tokens counter,
  cost_usd_cents counter,  -- Store in cents (integer)
  PRIMARY KEY (date, model)
);
```

### 4. Error Rate Monitoring

```sql
-- Error tracking by step type
SELECT 
  DATE(created_at) as date,
  tpe as step_type,
  COUNT(*) as total_steps,
  COUNT(CASE WHEN JSON_EXTRACT_PATH_TEXT(content, 'error') IS NOT NULL THEN 1 END) as error_steps,
  ROUND(100.0 * COUNT(CASE WHEN JSON_EXTRACT_PATH_TEXT(content, 'error') IS NOT NULL THEN 1 END) / 
    COUNT(*), 2) as error_rate_percent
FROM steps
WHERE created_at >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY date, step_type
ORDER BY date DESC, error_rate_percent DESC;
```

**PostgreSQL Optimization:**
- Create index: `CREATE INDEX idx_steps_error ON steps(created_at, tpe) INCLUDE ((content->>'error'));`

**ScyllaDB Approach:**
```cql
CREATE TABLE error_metrics (
  date text,
  step_type int,
  error_count counter,
  total_count counter,
  PRIMARY KEY (date, step_type)
);
```

### 5. Step Performance Analysis

```sql
-- Step type latency percentiles
SELECT 
  tpe as step_type,
  COUNT(*) as step_count,
  PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY CAST(JSON_EXTRACT_PATH_TEXT(content, 'processing_time_ms') AS INTEGER)) as p50_ms,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY CAST(JSON_EXTRACT_PATH_TEXT(content, 'processing_time_ms') AS INTEGER)) as p95_ms,
  MAX(CAST(JSON_EXTRACT_PATH_TEXT(content, 'processing_time_ms') AS INTEGER)) as max_ms
FROM steps
WHERE created_at >= CURRENT_DATE - INTERVAL '1 day'
  AND JSON_EXTRACT_PATH_TEXT(content, 'processing_time_ms') IS NOT NULL
GROUP BY step_type
ORDER BY step_count DESC;
```

**PostgreSQL Optimization:**
- Create index: `CREATE INDEX idx_steps_latency ON steps(created_at, tpe) INCLUDE ((content->>'processing_time_ms'));`

**ScyllaDB Approach:**
```cql
CREATE TABLE latency_buckets (
  date text,
  step_type int,
  latency_bucket text,  -- '0-10ms', '10-50ms', etc.
  count counter,
  PRIMARY KEY ((date, step_type), latency_bucket)
);
```

### 6. Real-Time System Health

```sql
-- Real-time system health (last 15 minutes)
SELECT 
  COUNT(*) as total_steps,
  COUNT(CASE WHEN JSON_EXTRACT_PATH_TEXT(content, 'error') IS NOT NULL THEN 1 END) as error_steps,
  ROUND(100.0 * COUNT(CASE WHEN JSON_EXTRACT_PATH_TEXT(content, 'error') IS NOT NULL THEN 1 END) / 
    NULLIF(COUNT(*), 0), 2) as error_rate_percent,
  COUNT(DISTINCT agent_turn_uid) as turns_processed,
  AVG(CAST(JSON_EXTRACT_PATH_TEXT(content, 'processing_time_ms') AS INTEGER)) as avg_processing_ms
FROM steps
WHERE created_at >= NOW() - INTERVAL '15 minutes';
```

**PostgreSQL Optimization:**
- Create time-based index: `CREATE INDEX idx_steps_recent ON steps(created_at) WHERE created_at > NOW() - INTERVAL '1 hour';`
- Refresh materialized view every minute

**ScyllaDB Approach:**
```cql
CREATE TABLE real_time_metrics (
  timestamp timestamp,
  metric_name text,
  metric_value counter,
  PRIMARY KEY (timestamp, metric_name)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

## Optimization Strategies

### PostgreSQL 

1. **Create targeted indices** for each query pattern
2. **Use materialized views** for expensive aggregations
3. **Implement partitioning** by date for large tables

### ScyllaDB

1. **Denormalize at write time** - calculate metrics as data is inserted
2. **Use counter columns** for aggregations
3. **Design partition keys** to align with query patterns

## Conclusion

These six essential queries provide comprehensive visibility into both user behavior and AI model performance while maintaining system efficiency. The optimization strategies differ between PostgreSQL and ScyllaDB:

- **PostgreSQL** excels at complex analytical queries but requires careful indexing and materialized views.
- **ScyllaDB** requires denormalizing data at write time but delivers exceptional read performance for pre-aggregated data.

By focusing on these core queries, we can build effective analytics dashboards that provide actionable insights without overwhelming our database infrastructure.
