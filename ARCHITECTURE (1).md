# Dhaka Tesla Pool: Architecture and ERD

Database: MySQL (XAMPP for local development, MySQL container in `docker compose` for submission).

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
