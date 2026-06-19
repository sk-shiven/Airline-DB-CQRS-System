# Enterprise Airline Booking & Operations Management System

An enterprise-grade, highly scalable database architecture designed to manage the complex lifecycle of domestic and international aviation operations. This project provides a production-ready PostgreSQL schema that balances strict data integrity with sub-millisecond read performance using modern architectural patterns.

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

1. **The PNR Aggregation Layer:** A parent `PNR_Group` table sits atop the transactional core, allowing multiple travelers and connection legs to be booked as a single transaction while maintaining segment-level granularity for ground ops.
2. **Append-Only State Ledgers:** Instead of overwriting historical booking statuses, the `BookingStatus` table acts as a time-series ledger to preserve complete system auditability.
3. **Controlled Denormalization:** Strategic snapshots (such as locking in `excess_fee` amounts at the moment of check-in) insulate historical accounting ledgers against future corporate policy changes.
4. **Strict BCNF Geography:** Geopolitical locations are strictly hierarchical (`Country` → `City` → `Airport`) to prevent transitive dependencies and geographic data drift.

## 🛠️ Tech Stack Concept

* **Primary Database (Write Store):** MySQL
* **Caching / Read Store:** Redis Materialized Views
* **Message Broker:** RabbitMQ / Apache Kafka (for async denormalization)
* **Design Pattern:** CQRS + Event-Driven Architecture

## 🗄️ Core Database Indexing Strategy

To support high-frequency transactions and the CQRS read-model build process, the following custom B-Tree and Hash indexes are deployed:

```sql
-- Query 1: Accelerates structural route flight selection loops
CREATE INDEX idx_flight_route_date ON Flight (route_id, scheduled_departure);

-- Query 2: Drives fast seat lock validations for checkout workflows
CREATE INDEX idx_seat_inventory_lookup ON Seat (flight_id, availability_status) INCLUDE (fare_class_id, seat_number);

-- Query 3: Powers real-time terminal gate operational dashboard counts
CREATE INDEX idx_checkin_boarding_alert ON CheckIn (boarding_status) WHERE boarding_status = 'checked_in';

-- Query 4: High-speed point lookup hash indexing for PNR references
CREATE UNIQUE INDEX idx_pnr_code_hash ON PNR_Group USING hash (pnr_code);

-- Query 5: Filters and isolates major disruption metrics across schedules
CREATE INDEX idx_flight_delay_tracking ON DelayRecord (delay_minutes) WHERE delay_minutes > 120;

-- Query 6: Optimizes clustering evaluations over crew member rosters
CREATE INDEX idx_crew_assignment_schedule ON CrewAssignment (crew_id, assignment_datetime);
```

This schema architecture was designed to bridge the gap between rigorous academic database normalization and the real-world performance needs of a high-concurrency commercial aviation platform. It is intended for use by enterprise software engineers, database architects, and aviation operations specialists seeking to implement a robust, scalable, and maintainable airline booking and operations management system.