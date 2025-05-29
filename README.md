# Sarvam AI Technical Assignment

## Solutions Overview

This repository contains my solutions to the Sarvam AI technical assignment for the Reasoning AI Team - Back End - Services & API Engineer position.

### Problem Solutions

1. **Secondary Indices Strategy** - [solutions/problem1_secondary_indices.md](solutions/problem1_secondary_indices.md)
   - Developed an optimized indexing strategy for a conversational AI platform's database
   - Focused on balancing query performance with storage overhead and write amplification

2. **ScyllaDB Migration Strategy** - [solutions/problem2_scylladb_migration.md](solutions/problem2_scylladb_migration.md)
   - Created a migration plan from relational database to ScyllaDB
   - Designed denormalized schema optimized for conversational data access patterns

3. **Analytics and Evaluation Query Strategy** - [solutions/problem3_analytics_queries.md](solutions/problem3_analytics_queries.md)
   - Defined key queries for analytics and model evaluation
   - Provided optimization strategies for both relational and NoSQL environments

4. **CQRS Implementation Strategy** - [solutions/problem4_cqrs_implementation.md](solutions/problem4_cqrs_implementation.md)
   - Proposed a hybrid approach combining CDC with strategic dual-writes
   - Balanced latency, consistency, scalability, and fault tolerance requirements

5. **Helm Chart for NoSQL Deployment** - Not completed
   - Due to time constraints and my current technical understanding, I was unable to complete the Helm chart implementation for Question 5

## What I Learned

This assignment helped me develop a deeper understanding of:

- Optimizing database indices for specific query patterns in conversational AI
- Denormalization strategies for NoSQL databases like ScyllaDB
- Tradeoffs between storage overhead and query performance
- Change Data Capture (CDC) mechanisms for data synchronization
- Command Query Responsibility Segregation (CQRS) architectural pattern
- Balancing eventual consistency with user experience requirements
- Analytics query optimization for high-volume conversation data

## Relation to Past Work

This assignment directly relates to my experience at thena.ai, where I worked on conversational agent database management in PostgreSQL. Specifically, I've dealt with the challenges of representing both linear conversations and acyclic graph structures of agent and human turns.

The solutions I've proposed build on this experience while extending my knowledge into NoSQL architectures and more advanced data synchronization patterns.
