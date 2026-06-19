# Enterprise Airline Booking & Operations Management System

An enterprise-grade, highly scalable database architecture designed to manage the complex lifecycle of domestic and international aviation operations. This project provides a production-ready MySQL 8.0 schema that balances strict data integrity with sub-millisecond read performance using modern architectural patterns.

## 📖 Overview

At its core, this system manages everything from passenger booking lifecycles and dynamic seat inventory to ground logistics (check-in, baggage, special requests) and fleet/crew telemetry.

To solve the classic database trade-off between strict normalization (which degrades read performance due to deep SQL joins) and fast user experiences, this architecture implements a **Command Query Responsibility Segregation (CQRS)** pattern.

* **The Write Path (MySQL):** Engineered in strict **Boyce-Codd Normal Form (BCNF)** to guarantee absolute data integrity, eliminate update anomalies, and provide a bulletproof financial source of truth.
* **The Read Path (Redis / Materialized Views):** An asynchronous event bus (RabbitMQ) listens to transactional commits and flattens the relational data into pre-joined JSON documents, providing zero-join, sub-millisecond read speeds for API gateways and external clients.

## ✨ Key Business Capabilities

* **Complex Itinerary Management:** Supports multi-passenger group bookings and multi-leg connecting flights grouped seamlessly under a unified PNR (Passenger Name Record) envelope.
* **Granular Operational Logistics:** Real-time tracking of gate check-ins, baggage routing, special service requests (SSR), and operational flight delays at the individual flight-segment level.
* **Dynamic Fleet & Crew Mappings:** Manages aircraft model configurations, physical hull tracking, dynamic seat availability matrices, and overlapping crew duty rosters.
* **Decoupled Financial Integrity:** Payment and refund ledgers are completely decoupled from individual segments to natively support group billing, partial cancellations, and clean accounting audits.

## 🏗️ Architectural Highlights

1. **The PNR Aggregation Layer:** A parent `pnr_group` table sits atop the transactional core, allowing multiple travelers and connection legs to be booked as a single transaction while maintaining segment-level granularity for ground ops.
2. **Append-Only State Ledgers:** Instead of overwriting historical booking statuses, the `booking_status` table acts as a time-series ledger to preserve complete system auditability.
3. **Controlled Denormalization:** Strategic snapshots (such as locking in `excess_fee` amounts at the moment of check-in) insulate historical accounting ledgers against future corporate policy changes.
4. **Strict BCNF Geography:** Geopolitical locations are strictly hierarchical (`country` → `city` → `airport`) to prevent transitive dependencies and geographic data drift.

## 🛠️ Tech Stack Concept

* **Primary Database (Write Store):** MySQL 8.0
* **Caching / Read Store:** Redis Materialized Views
* **Message Broker:** RabbitMQ / Apache Kafka (for async denormalization)
* **Design Pattern:** CQRS + Event-Driven Architecture

## 🗄️ Core Database Indexing Strategy

To support high-frequency transactions and the CQRS read-model build process, the following custom B-Tree indexes are deployed. All indexes are MySQL 8.0 InnoDB-compatible. InnoDB additionally builds an adaptive hash index in memory on hot B-Tree pages, approximating O(1) lookup for repeated exact-match access patterns without requiring explicit hash index declarations.

> **MySQL compatibility notes:**
> - `INCLUDE` columns (PostgreSQL syntax) are not supported in InnoDB. Covering index behavior is achieved by adding all required columns directly to the index key.
> - Partial/filtered indexes with `WHERE` clauses (PostgreSQL syntax) are not supported in MySQL. Full indexes are used instead; the application layer applies the equivalent filter at query time.
> - Explicit `HASH` indexes are not supported in InnoDB. A `UNIQUE` B-Tree index on `pnr_code` provides the same point-lookup performance via InnoDB's adaptive hash mechanism.

```sql
-- Index 1: Route + date composite for flight search
-- Equality match on route_id, then range scan on scheduled_departure within that corridor.
-- Eliminates full table scans on the flight schedule for the most frequent search query.
CREATE INDEX idx_flight_route_date
    ON flight (route_id, scheduled_departure);

-- Index 2: Covering index for seat inventory checkout
-- All four columns are in the index key (InnoDB does not support INCLUDE columns).
-- Query can be answered entirely from the index B-Tree without a row heap lookup.
-- Prevents row-locking contention during high-traffic checkout windows.
CREATE INDEX idx_seat_inventory_lookup
    ON seat (flight_id, availability_status, fare_class_id, seat_number);

-- Index 3: Gate boarding status for checked-in passenger dashboard
-- MySQL does not support partial indexes with WHERE clauses; this is a full B-Tree index.
-- Application layer adds WHERE boarding_status = 'checked_in' at query time.
-- booking_id suffix allows gate agents to retrieve booking details without a secondary lookup.
CREATE INDEX idx_checkin_boarding_alert
    ON checkin (boarding_status, booking_id);

-- Index 4: PNR code exact-match lookup
-- InnoDB does not support explicit hash indexes; B-Tree UNIQUE is used instead.
-- InnoDB adaptive hash index provides near O(1) lookup on hot pages automatically.
-- UNIQUE constraint also enforces the business rule that PNR codes are globally non-repeating.
-- Note: if the UNIQUE constraint on pnr_group(pnr_code) was declared at table creation,
-- MySQL auto-generates this index and explicit creation is not required.
CREATE UNIQUE INDEX idx_pnr_code
    ON pnr_group (pnr_code);

-- Index 5: Delay minutes range scan for operations dashboard
-- MySQL does not support WHERE-clause filtered indexes; this is a full B-Tree index.
-- Application query adds WHERE delay_minutes > 120; B-Tree enables efficient range scans.
-- flight_id suffix allows dashboard queries to retrieve affected flight IDs from the index directly.
CREATE INDEX idx_flight_delay_tracking
    ON delay_record (delay_minutes, flight_id);

-- Index 6: Crew scheduling conflict detection
-- Equality match on crew_id, then range scan on assignment_datetime for overlapping duty windows.
-- Hit on every new crew assignment write; most critical index on the write path.
CREATE INDEX idx_crew_assignment_schedule
    ON crew_assignment (crew_id, assignment_datetime);
```

This schema architecture was designed to bridge the gap between rigorous academic database normalization and the real-world performance needs of a high-concurrency commercial aviation platform. It is intended for use by enterprise software engineers, database architects, and aviation operations specialists seeking to implement a robust, scalable, and maintainable airline booking and operations management system.