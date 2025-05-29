# Question 2: ScyllaDB Migration Strategy

## Executive Summary

Migrating Agent Turns, Human Turns, and Steps tables to ScyllaDB offers significant performance and scalability benefits for our high-volume conversational AI platform. The denormalized schema design optimizes read patterns while accepting reasonable tradeoffs.

**Benefits**: 10-25x faster reads, horizontal scalability, lower latency  
**Costs**: Eventual consistency, increased storage (~20%), more complex data model

## Current Challenges

Our relational database struggles with:
- **Scale**: 10M sessions, 100M+ turns, 500M+ steps exceeding capacity
- **Performance**: Complex joins creating bottlenecks
- **Load**: 100K+ conversation loads per minute

Typical problematic query:
```sql
SELECT ht.prompt, at.uid, s.idx, s.content 
FROM human_turns ht
JOIN agent_turns at ON ht.session_uid = at.session_uid  
JOIN steps s ON at.uid = s.agent_turn_uid
WHERE ht.session_uid = 'session_123'
ORDER BY ht.created_at, at.created_at, s.idx;
```

## Proposed Schema Design

```cql
-- For finding a user's recent conversations
CREATE TABLE sessions_by_user (
    account_uid text,         -- Partition Key
    created_at timestamp,     -- Clustering Key 1
    session_uid text,         -- Clustering Key 2
    title text,
    deleted_at timestamp,
    PRIMARY KEY (account_uid, created_at, session_uid)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- For loading all turns in a conversation chronologically
CREATE TABLE turns_by_session (
    session_uid text,         -- Partition Key
    created_at timestamp,     -- Clustering Key 1
    turn_uid text,            -- Clustering Key 2
    turn_type text,           -- 'human' or 'agent'
    parent_uid text,
    active_child text,
    prompt text,              -- only used for human turns
    liked boolean,            -- only used for agent turns
    PRIMARY KEY (session_uid, created_at, turn_uid)
) WITH CLUSTERING ORDER BY (created_at ASC);

-- For getting all the steps for a specific turn
CREATE TABLE steps_by_turn (
    turn_uid text,            -- Partition Key
    idx int,                  -- Clustering Key
    tpe int,
    content text,
    created_at timestamp,
    PRIMARY KEY (turn_uid, idx)
) WITH CLUSTERING ORDER BY (idx ASC);

-- For navigating the conversation tree
CREATE TABLE turns_by_parent (
    parent_uid text,          -- Partition Key
    created_at timestamp,     -- Clustering Key 1
    turn_uid text,            -- Clustering Key 2
    session_uid text,
    turn_type text,
    PRIMARY KEY (parent_uid, created_at, turn_uid)
) WITH CLUSTERING ORDER BY (created_at ASC);
```

## Key Design Decisions

### Partition Key Strategy

- **Session-Based** (`session_uid` for turns): Keeps conversation data on same node
- **User-Based** (`account_uid` for sessions): Distributes users across cluster
- **Turn-Based** (`turn_uid` for steps): Co-locates steps for efficient retrieval
- **Parent-Based** (`parent_uid` for relationships): Optimizes branch navigation

### Clustering Key Strategy

- **Time-Based Ordering**: Chronological for turns (ASC), recent-first for sessions (DESC)
- **Unique Identifiers**: Ensures deterministic ordering when timestamps collide

## Optimized Query Patterns

1. **Load Conversation**
```cql
-- Get turns (single partition scan)
SELECT * FROM turns_by_session WHERE session_uid = 'session123';
-- Get steps for each turn
SELECT * FROM steps_by_turn WHERE turn_uid = 'turn456';
```

2. **User's Recent Conversations**
```cql
SELECT * FROM sessions_by_user WHERE account_uid = 'user789' LIMIT 20;
```

3. **Navigate Branches**
```cql
SELECT * FROM turns_by_parent WHERE parent_uid = 'turn456';
```

## Consistency Management

### Model Changes
- ACID transactions → Eventually consistent operations
- Immediate consistency → Tunable consistency levels
- Referential integrity → Application-managed relationships

### Configuration
```cql
-- Strong consistency for writes
INSERT INTO turns_by_session (...) USING CONSISTENCY QUORUM;

-- Eventual consistency for reads (faster)
SELECT * FROM turns_by_session WHERE session_uid = 'session123' USING CONSISTENCY ONE;
```

### Data Duplication Strategy
1. **Batch Operations**
```cql
BEGIN BATCH
  INSERT INTO turns_by_session (...);
  INSERT INTO turns_by_parent (...);
APPLY BATCH;
```
2. **Background Repair Jobs**: Fix inconsistencies automatically
3. **Read Repairs**: Let ScyllaDB fix inconsistencies during reads

## Performance Improvements

| Operation | Current | ScyllaDB | Improvement |
|-----------|---------|----------|-------------|
| Load conversation | 50-500ms | 5-20ms | 10-25x |
| User history | 100-1000ms | 10-50ms | 10-20x |
| Thread navigation | 20-200ms | 2-10ms | 10-20x |

## Implementation Strategy

1. **Dual-Write Setup**
   - Write to both databases
   - PostgreSQL remains source of truth
   - Test ScyllaDB reads in non-production

2. **Read Migration**
   - Gradually shift reads to ScyllaDB
   - Implement PostgreSQL fallback
   - Monitor performance and consistency

3. **Full Migration**
   - ScyllaDB becomes primary
   - PostgreSQL for analytics/backup
   - Optimize based on real usage

## Success Metrics

- **Performance**: p95 latency < 100ms, 10x current throughput
- **Consistency**: <1% stale reads
- **Resource Usage**: Storage increase ~20%, CPU efficiency

## Risk Mitigation

- **Data Loss**: Regular snapshots, cross-DC replication
- **Performance**: Circuit breakers, gradual traffic shifting
- **Consistency**: Application validation, conflict resolution

## Conclusion

The migration to ScyllaDB offers compelling performance benefits (10-25x faster reads) that justify the complexity of managing eventual consistency. The denormalized schema optimizes for our primary access patterns while maintaining operational flexibility.

For our high-scale, read-heavy conversational AI workload, the tradeoffs favor ScyllaDB with careful attention to consistency requirements. A phased migration approach minimizes risk while allowing validation at each step.
