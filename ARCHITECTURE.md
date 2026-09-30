# Dhaka Tesla Pool: Architecture and ERD

This document is the design-first step from the PRD (Section 9). If the implementation changes, update it here.

**Database:** MySQL (MariaDB via XAMPP during local development, a MySQL container in `docker compose` for submission). All tables use the **InnoDB** engine, which is required for transactions and row locks.

## 1. System architecture

```mermaid
flowchart LR
    subgraph Client
        B["Browser"]
    end

    subgraph Docker_Compose["docker compose"]
        F["Frontend<br/>React + Vite<br/>served by nginx"]
        subgraph API["Node.js API (Express)"]
            R["Routes / Controllers<br/>HTTP + validation"]
            S["Services<br/>ride, pool, fare, auth"]
            D["Data access<br/>transactions + row locks"]
            R --> S --> D
        end
        DB[("MySQL<br/>InnoDB")]
    end

    X["XAMPP MySQL<br/>local dev + phpMyAdmin"] -.->|"same schema"| DB

    B -->|"HTTPS"| F
    F -->|"REST + JSON<br/>Bearer JWT"| R
    D -->|"SQL"| DB
    M["Migrations + seed<br/>Jashim, Bullet, Nusrat, Rafiq, Shirin"] -.->|"runs on startup"| DB
```

**Why this shape**
- One API and one database. No queues, cache or microservices, because the MVP has no load that needs them.
- Business rules (matching, fare, capacity, state transitions) live in the **Services** layer, not in routes or the frontend.
- Capacity safety is enforced twice: by a row lock in the service and by a `CHECK` constraint in the database.
- XAMPP is for fast local development. The submitted version must still run with `docker compose up`, so the same migrations run against a MySQL container.

## 2. Ride lifecycle

```mermaid
stateDiagram-v2
    [*] --> REQUESTED
    REQUESTED --> MATCHED: driver accepts / pool joined
    REQUESTED --> CANCELLED: passenger cancels
    MATCHED --> DRIVER_ARRIVED: driver marks arrival
    MATCHED --> CANCELLED: passenger or driver cancels
    DRIVER_ARRIVED --> STARTED: driver starts trip
    DRIVER_ARRIVED --> CANCELLED: passenger cancels before start
    STARTED --> COMPLETED: driver completes trip
    COMPLETED --> [*]
    CANCELLED --> [*]
```

Rules:
- Any transition not shown above is rejected with `409 Conflict`.
- Once a ride is `STARTED`, the passenger can no longer cancel.
- Every transition writes one row to `ride_status_history`.

## 3. ERD

```mermaid
erDiagram
    USERS ||--o| VEHICLES : "driver owns"
    USERS ||--o{ RIDE_REQUESTS : "passenger requests"
    USERS ||--o{ PAYMENTS : "pays"
    VEHICLES ||--o{ POOLS : "runs"
    ZONES ||--o{ RIDE_REQUESTS : "pickup"
    ZONES ||--o{ RIDE_REQUESTS : "destination"
    POOLS ||--o{ POOL_MEMBERS : "has"
    RIDE_REQUESTS ||--o| POOL_MEMBERS : "joins"
    RIDE_REQUESTS ||--o{ RIDE_STATUS_HISTORY : "audited by"
    RIDE_REQUESTS ||--o| PAYMENTS : "settled by"

    USERS {
        int id PK "AUTO_INCREMENT"
        varchar name
        varchar email UK
        varchar password_hash
        varchar role "PASSENGER or DRIVER"
        int wallet_balance_paisa "TeslaPay simulated"
        tinyint is_online "drivers only, 0 or 1"
        datetime created_at
    }

    VEHICLES {
        int id PK "AUTO_INCREMENT"
        int driver_id FK, UK
        varchar name "e.g. Bullet"
        int capacity "fixed, CHECK > 0"
        datetime created_at
    }

    ZONES {
        int id PK "AUTO_INCREMENT"
        varchar name UK "Banani, Gulshan 1, ..."
        int zone_index "position along the corridor"
        decimal lat
        decimal lng
    }

    POOLS {
        int id PK "AUTO_INCREMENT"
        int vehicle_id FK
        varchar status "OPEN, FULL, IN_PROGRESS, COMPLETED, CANCELLED"
        int capacity "snapshot of vehicle capacity"
        int seats_taken "CHECK seats_taken <= capacity"
        int pickup_zone_id FK
        datetime created_at
        datetime completed_at
    }

    POOL_MEMBERS {
        int id PK "AUTO_INCREMENT"
        int pool_id FK
        int ride_request_id FK, UK
        int seats
        datetime joined_at
        datetime left_at "set on cancel"
    }

    RIDE_REQUESTS {
        int id PK "AUTO_INCREMENT"
        int passenger_id FK
        int pickup_zone_id FK
        int destination_zone_id FK
        int seats "CHECK between 1 and 3"
        varchar status "see lifecycle"
        int base_fare_paisa
        int distance_charge_paisa
        int pool_discount_paisa
        int total_fare_paisa "base + distance - discount"
        varchar payment_method "CASH or TESLAPAY"
        datetime created_at
        datetime cancelled_at
    }

    RIDE_STATUS_HISTORY {
        int id PK "AUTO_INCREMENT"
        int ride_request_id FK
        varchar from_status
        varchar to_status
        int changed_by FK "user id"
        datetime created_at
    }

    PAYMENTS {
        int id PK "AUTO_INCREMENT"
        int ride_request_id FK, UK
        int user_id FK
        varchar method "CASH or TESLAPAY"
        int amount_paisa
        varchar status "PENDING, PAID, REFUNDED"
        datetime created_at
    }
```

## 4. Table notes (be ready to explain each)

| Table | Purpose | Key decision |
|-------|---------|--------------|
| `users` | Passengers and drivers in one table, split by `role`. | One login system. Wallet balance lives here because the wallet is simulated. |
| `vehicles` | A Tesla and its fixed seat capacity. | `driver_id` is unique: one driver owns one Tesla in the MVP. |
| `zones` | Predefined Dhaka areas with a `zone_index`. | Replaces real routing. The matching rule and distance charge both use `zone_index`. |
| `pools` | One shared trip of one Tesla. | `seats_taken` with `CHECK (seats_taken <= capacity)` is the last line of defence against overbooking. |
| `pool_members` | Which ride request belongs to which pool. | `ride_request_id` is unique, so a request can be in only one pool. `left_at` keeps history when someone cancels. |
| `ride_requests` | One passenger's request and their own fare. | Each passenger has an individual fare breakdown. Passengers only read rows where `passenger_id` is their own id. |
| `ride_status_history` | Append-only audit log of every state change. | Lets us explain exactly what happened after a ride ends. |
| `payments` | Cash or TeslaPay record per ride. | Separate from `ride_requests` so payment state can change without touching ride state. |

**MySQL-specific notes**
- Use `INT AUTO_INCREMENT` ids: simpler to read in phpMyAdmin than UUIDs.
- Use `DATETIME` (store UTC) for timestamps and `TINYINT(1)` for booleans.
- `CHECK` constraints are enforced in MariaDB 10.2+ and MySQL 8.0.16+. Older versions silently ignore them, so verify your XAMPP version.
- Status columns can be `VARCHAR` with a `CHECK (status IN (...))`, or `ENUM`. `VARCHAR` + `CHECK` is easier to change later.

## 5. Concurrency: Bullet has 1 seat left

Nusrat and Shirin both claim the last seat at the same instant.

```mermaid
sequenceDiagram
    participant N as Nusrat
    participant S as Shirin
    participant API as API
    participant DB as MySQL InnoDB

    N->>API: POST /rides/:id/join
    S->>API: POST /rides/:id/join
    API->>DB: START TRANSACTION, SELECT pool FOR UPDATE (Nusrat)
    API->>DB: START TRANSACTION, SELECT pool FOR UPDATE (Shirin)
    Note over DB: Shirin waits for the row lock
    API->>DB: seats_taken 2 to 3, COMMIT
    DB-->>API: Nusrat gets the seat
    Note over DB: Lock released, Shirin reads seats_taken = 3
    API-->>S: 409 Pool is full
```

**Now (MVP):** an InnoDB transaction with `SELECT ... FOR UPDATE` on the pool row, plus the `CHECK` constraint as a safety net. MyISAM would not work here because it has no transactions or row locks.

**At larger scale:** move to optimistic locking with a version column and retries, partition matching by geo-cell so contention is spread out, and add idempotency keys so retried requests never double-book.

## 6. Assumptions baked into this design

- One driver owns one Tesla; capacity is 3 seats.
- A passenger can book 1 to 3 seats in one request.
- Money is stored as **integer paisa** to avoid floating-point rounding errors.
- Geography is a predefined zone list; no map API or real routing.
- Payment is cash or simulated TeslaPay wallet; no real gateway.
- Local development uses XAMPP MySQL; the submitted setup uses a MySQL container.
