# Football Ticket Booking System — Database Design & SQL Queries

A PostgreSQL database project for managing football fans, tournament matches, and ticket booking records.

---

## Table of Contents

- [Database Setup](#database-setup)
- [Schema Overview](#schema-overview)
- [Table Creation](#table-creation)
- [Sample Data](#sample-data)
- [SQL Queries](#sql-queries)

---

## Database Setup

Run this first in Beekeeper Studio to create the database, then switch to it before running anything else:

```sql
CREATE DATABASE Football_ticketing;
```

---

## Schema Overview

| Table    | Description                                              |
|----------|----------------------------------------------------------|
| Users    | Stores all registered users (Football Fans & Ticket Managers) |
| Matches  | Stores all football match details and ticket pricing     |
| Bookings | Transactional table linking users to their match tickets |

---

## Table Creation

> Run these in exact order: drop → Users → Matches → Bookings

### Step 1 — Drop existing tables (prevent conflicts)

```sql
DROP TABLE IF EXISTS Bookings;
DROP TABLE IF EXISTS Matches;
DROP TABLE IF EXISTS Users;
```

### Step 2 — Create Users Table

```sql
CREATE TABLE Users (
    user_id      SERIAL,
    full_name    VARCHAR(100)  NOT NULL,
    email        VARCHAR(150)  NOT NULL,
    role         VARCHAR(50)   NOT NULL,
    phone_number VARCHAR(20),

    CONSTRAINT pk_users       PRIMARY KEY (user_id),
    CONSTRAINT uq_users_email UNIQUE (email),
    CONSTRAINT chk_users_role CHECK (role IN ('Football Fan', 'Ticket Manager'))
);
```

### Step 3 — Create Matches Table

```sql
CREATE TABLE Matches (
    match_id            SERIAL,
    fixture             VARCHAR(200)  NOT NULL,
    tournament_category VARCHAR(100)  NOT NULL,
    base_ticket_price   DECIMAL(10,2) NOT NULL,
    match_status        VARCHAR(50)   NOT NULL,

    CONSTRAINT pk_matches         PRIMARY KEY (match_id),
    CONSTRAINT chk_matches_price  CHECK (base_ticket_price >= 0),
    CONSTRAINT chk_matches_status CHECK (match_status IN ('Available', 'Selling Fast', 'Sold Out', 'Postponed'))
);
```

### Step 4 — Create Bookings Table

```sql
CREATE TABLE Bookings (
    booking_id     SERIAL,
    user_id        INT           NOT NULL,
    match_id       INT           NOT NULL,
    seat_number    VARCHAR(20),
    payment_status VARCHAR(50),
    total_cost     DECIMAL(10,2) NOT NULL,

    CONSTRAINT pk_bookings             PRIMARY KEY (booking_id),
    CONSTRAINT fk_bookings_user        FOREIGN KEY (user_id)  REFERENCES Users(user_id),
    CONSTRAINT fk_bookings_match       FOREIGN KEY (match_id) REFERENCES Matches(match_id),
    CONSTRAINT chk_bookings_total_cost CHECK (total_cost >= 0),
    CONSTRAINT chk_bookings_pay_status CHECK (
        payment_status IN ('Pending', 'Confirmed', 'Cancelled', 'Refunded')
        OR payment_status IS NULL
    )
);
```

---

## Sample Data

### Insert Users

```sql
INSERT INTO Users (user_id, full_name, email, role, phone_number) VALUES
(1, 'Tanvir Rahman',  'tanvir@mail.com', 'Football Fan',   '+8801711111111'),
(2, 'Asif Haque',    'asif@mail.com',   'Football Fan',   '+8801722222222'),
(3, 'Sajjad Rahman', 'sajjad@mail.com', 'Ticket Manager', '+8801733333333'),
(4, 'Jannat Ara',    'jannat@mail.com', 'Football Fan',    NULL);
```

### Insert Matches

```sql
INSERT INTO Matches (match_id, fixture, tournament_category, base_ticket_price, match_status) VALUES
(101, 'Real Madrid vs Barcelona', 'Champions League', 150.00, 'Available'),
(102, 'Man City vs Liverpool',    'Premier League',   120.00, 'Selling Fast'),
(103, 'Bayern Munich vs PSG',     'Champions League', 130.00, 'Available'),
(104, 'AC Milan vs Inter Milan',  'Serie A',           90.00, 'Sold Out'),
(105, 'Juventus vs Roma',         'Serie A',           80.00, 'Available');
```

### Insert Bookings

```sql
INSERT INTO Bookings (booking_id, user_id, match_id, seat_number, payment_status, total_cost) VALUES
(501, 1, 101, 'A-12', 'Confirmed', 150.00),
(502, 1, 102, 'B-04', 'Confirmed', 120.00),
(503, 2, 101, 'A-13', 'Confirmed', 150.00),
(504, 2, 101,  NULL,   NULL,       150.00),
(505, 3, 102, 'C-20', 'Pending',   120.00);
```

---

## SQL Queries

### Query 1 — Retrieve all Champions League matches with status 'Available'

```sql
SELECT match_id, fixture, base_ticket_price
FROM Matches
WHERE tournament_category = 'Champions League'
AND match_status = 'Available';
```

**Expected Output:**

| match_id | fixture                  | base_ticket_price |
|----------|--------------------------|-------------------|
| 101      | Real Madrid vs Barcelona | 150.00            |
| 103      | Bayern Munich vs PSG     | 130.00            |

---

### Query 2 — Search users whose name starts with 'Tanvir' or contains 'Haque' (case-insensitive)

**Concepts used:** `LIKE`, `ILIKE`

```sql
SELECT user_id, full_name, email
FROM Users
WHERE full_name ILIKE 'Tanvir%'
OR full_name ILIKE '%Haque%';
```

**Expected Output:**

| user_id | full_name     | email           |
|---------|---------------|-----------------|
| 1       | Tanvir Rahman | tanvir@mail.com |
| 2       | Asif Haque    | asif@mail.com   |

---

### Query 3 — Retrieve bookings where payment status is NULL, replacing it with 'Action Required'

**Concepts used:** `IS NULL`, `COALESCE`

```sql
SELECT booking_id, user_id, match_id,
       COALESCE(payment_status, 'Action Required') AS systematic_status
FROM Bookings
WHERE payment_status IS NULL;
```

**Expected Output:**

| booking_id | user_id | match_id | systematic_status |
|------------|---------|----------|-------------------|
| 504        | 2       | 101      | Action Required   |

---

### Query 4 — Retrieve booking details with user full name and match fixture

**Concepts used:** `INNER JOIN`

```sql
SELECT b.booking_id, u.full_name, m.fixture, b.total_cost
FROM Bookings b
INNER JOIN Users u ON b.user_id = u.user_id
INNER JOIN Matches m ON b.match_id = m.match_id;
```

**Expected Output:**

| booking_id | full_name     | fixture                  | total_cost |
|------------|---------------|--------------------------|------------|
| 501        | Tanvir Rahman | Real Madrid vs Barcelona | 150.00     |
| 502        | Tanvir Rahman | Man City vs Liverpool    | 120.00     |
| 503        | Asif Haque    | Real Madrid vs Barcelona | 150.00     |
| 504        | Asif Haque    | Real Madrid vs Barcelona | 150.00     |
| 505        | Sajjad Rahman | Man City vs Liverpool    | 120.00     |

---

### Query 5 — List all users and their booking IDs, including users with no bookings

**Concepts used:** `LEFT JOIN`

```sql
SELECT u.user_id, u.full_name, b.booking_id
FROM Users u
LEFT JOIN Bookings b ON u.user_id = b.user_id;
```

**Expected Output:**

| user_id | full_name     | booking_id |
|---------|---------------|------------|
| 1       | Tanvir Rahman | 501        |
| 1       | Tanvir Rahman | 502        |
| 2       | Asif Haque    | 503        |
| 2       | Asif Haque    | 504        |
| 3       | Sajjad Rahman | 505        |
| 4       | Jannat Ara    | NULL       |

---

### Query 6 — Find bookings where total cost is strictly higher than the average

```sql
SELECT booking_id, match_id, total_cost
FROM Bookings
WHERE total_cost > (SELECT AVG(total_cost) FROM Bookings);
```

**Expected Output:**

| booking_id | match_id | total_cost |
|------------|----------|------------|
| 501        | 101      | 150.00     |
| 503        | 101      | 150.00     |
| 504        | 101      | 150.00     |

> Average of all bookings = (150+120+150+150+120) ÷ 5 = **138.00**. Only rows with total_cost > 138 are returned.

---

### Query 7 — Retrieve top 2 most expensive matches, skipping the highest

**Concepts used:** `ORDER BY`, `LIMIT`, `OFFSET`

```sql
SELECT match_id, fixture, base_ticket_price
FROM Matches
ORDER BY base_ticket_price DESC
LIMIT 2 OFFSET 1;
```

**Expected Output:**

| match_id | fixture              | base_ticket_price |
|----------|----------------------|-------------------|
| 103      | Bayern Munich vs PSG | 130.00            |
| 102      | Man City vs Liverpool| 120.00            |

> `OFFSET 1` skips Real Madrid vs Barcelona (150.00) which is the absolute highest. `LIMIT 2` then takes the next 2 rows.

---

## ERD

The ERD was designed using Draw.io and includes:

- **Primary Keys (PK)** and **Foreign Keys (FK)** on all relevant fields
- **One to Many** — One User → Many Bookings
- **Many to One** — Many Bookings → One Match
- **One to One (Logical)** — Each Booking row maps exactly one user to one match for one reserved seat
- Crow's Foot notation for all relationship cardinalities
