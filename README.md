# ER Diagram – Airbnb-like Database (requirements.md)

## Overview
This document lists entities, their attributes (PK = primary key, FK = foreign key), and relationships for an Airbnb-style application.

## Entities and Attributes

### users
- user_id (PK)
- first_name
- last_name
- email (unique, not null)
- phone
- password_hash
- created_at (timestamp)

### properties
- property_id (PK)
- owner_id (FK → users.user_id)
- title
- description
- address
- city
- country
- price_per_night (decimal)
- created_at (timestamp)

### bookings
- booking_id (PK)
- user_id (FK → users.user_id)      -- the guest who booked
- property_id (FK → properties.property_id)
- start_date (date)
- end_date (date)
- total_price (decimal)             -- derived but stored for convenience
- status (enum: pending, confirmed, cancelled)
- created_at (timestamp)

### payments
- payment_id (PK)
- booking_id (FK → bookings.booking_id) UNIQUE
- amount (decimal)
- payment_date (timestamp)
- method (e.g., credit_card, cash, etc.)
- status (e.g., pending, paid, refunded)

### reviews
- review_id (PK)
- booking_id (FK → bookings.booking_id) UNIQUE
- rating (INT 1-5)
- comment (text)
- created_at (timestamp)

## Relationships (cardinality)
- users (1) — (M) properties  
  *One user can own many properties.*

- users (1) — (M) bookings  
  *One user (guest) can make many bookings.*

- properties (1) — (M) bookings  
  *One property can have many bookings over time.*

- bookings (1) — (1) payments  
  *Each booking has one payment record (enforced by UNIQUE constraint on payments.booking_id).*

- bookings (1) — (1) reviews  
  *Each completed booking may produce a single review (enforced by UNIQUE constraint on reviews.booking_id).*

## Simple ASCII ER sketch
# Database Normalization (normalization.md)

Goal: Bring the schema to Third Normal Form (3NF).

## 1NF (First Normal Form)
- All attributes are atomic and single-valued.
  - Example: `address`, `city`, `country` stored separately (no comma-separated blobs).
- No repeating groups inside rows.

## 2NF (Second Normal Form)
- Ensure every non-key attribute depends on the whole primary key.
  - All tables use single-column primary keys (serial IDs), so partial dependency issues avoided.
  - No tables with composite keys and non-key attributes depending on only part of a composite key.

## 3NF (Third Normal Form)
- Remove transitive dependencies: non-key attributes must not depend on other non-key attributes.
  - Example: `payments` does not store `user` contact — it references `bookings`, which references `users`.
  - `properties` stores `price_per_night` but not derived totals; `total_price` is stored on `bookings` to preserve historical values, not recalculated from `properties` at query-time.

## Specific normalization decisions
1. **Separate payments & reviews from bookings**  
   - Avoids duplicate payment/review data repeated per booking row; allows independent lifecycle and audit fields.

2. **Atomic address fields**  
   - `address`, `city`, `country` avoid storing a single `location` blob.

3. **No derived attribute duplication**  
   - `total_price` is the only stored derived value (to preserve historical price). This is a documented exception. Otherwise calculations performed at query time.

4. **Constraints**  
   - Use foreign keys and unique constraints to enforce intended one-to-one relationships (payments and reviews to bookings).

## Result
Schema adheres to 3NF: minimal redundancy, clear relationships, and referential integrity via foreign keys.
-- schema.sql
-- DROP existing tables (safe for reruns)
DROP TABLE IF EXISTS reviews CASCADE;
DROP TABLE IF EXISTS payments CASCADE;
DROP TABLE IF EXISTS bookings CASCADE;
DROP TABLE IF EXISTS properties CASCADE;
DROP TABLE IF EXISTS users CASCADE;

-- Users table
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100) NOT NULL UNIQUE,
    phone VARCHAR(30),
    password_hash TEXT NOT NULL,
    created_at TIMESTAMP WITHOUT TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Properties table
CREATE TABLE properties (
    property_id SERIAL PRIMARY KEY,
    owner_id INTEGER NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    title VARCHAR(150) NOT NULL,
    description TEXT,
    address VARCHAR(255),
    city VARCHAR(100),
    country VARCHAR(100),
    price_per_night DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    created_at TIMESTAMP WITHOUT TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Bookings table
CREATE TABLE bookings (
    booking_id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    property_id INTEGER NOT NULL REFERENCES properties(property_id) ON DELETE CASCADE,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    total_price DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    status VARCHAR(20) NOT NULL CHECK (status IN ('pending','confirmed','cancelled')),
    created_at TIMESTAMP WITHOUT TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_dates CHECK (end_date > start_date)
);

-- Payments table (one payment per booking enforced by UNIQUE)
CREATE TABLE payments (
    payment_id SERIAL PRIMARY KEY,
    booking_id INTEGER NOT NULL UNIQUE REFERENCES bookings(booking_id) ON DELETE CASCADE,
    amount DECIMAL(12,2) NOT NULL,
    payment_date TIMESTAMP WITHOUT TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    method VARCHAR(50),
    status VARCHAR(30)
);

-- Reviews table (one review per booking)
CREATE TABLE reviews (
    review_id SERIAL PRIMARY KEY,
    booking_id INTEGER NOT NULL UNIQUE REFERENCES bookings(booking_id) ON DELETE CASCADE,
    rating INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    comment TEXT,
    created_at TIMESTAMP WITHOUT TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Suggested indexes for performance
CREATE INDEX idx_properties_owner ON properties(owner_id);
CREATE INDEX idx_properties_city ON properties(city);
CREATE INDEX idx_bookings_user ON bookings(user_id);
CREATE INDEX idx_bookings_property ON bookings(property_id);
-- seed.sql
-- Sample users
INSERT INTO users (first_name, last_name, email, phone, password_hash)
VALUES
('John', 'Doe', 'john@example.com', '1234567890', 'hashed_pw1'),
('Jane', 'Smith', 'jane@example.com', '0987654321', 'hashed_pw2');

-- Sample properties (owner_id references users inserted above)
INSERT INTO properties (owner_id, title, description, address, city, country, price_per_night)
VALUES
(1, 'Beach House', 'Beautiful sea view house with 3 bedrooms.', '12 Ocean Drive', 'Cape Town', 'South Africa', 1200.00),
(2, 'City Apartment', 'Modern apartment downtown, close to restaurants.', '45 Main St', 'Johannesburg', 'South Africa', 850.00);

-- Sample bookings
-- Booking 1: user 1 books property 2 from 2025-09-10 to 2025-09-15 (5 nights * 850 = 4250)
-- Booking 2: user 2 books property 1 from 2025-09-20 to 2025-09-22 (2 nights * 1200 = 2400)
INSERT INTO bookings (user_id, property_id, start_date, end_date, total_price, status)
VALUES
(1, 2, '2025-09-10', '2025-09-15', 4250.00, 'confirmed'),
(2, 1, '2025-09-20', '2025-09-22', 2400.00, 'pending');

-- Sample payments
INSERT INTO payments (booking_id, amount, method, status)
VALUES
(1, 4250.00, 'credit_card', 'paid');

-- Sample reviews
INSERT INTO reviews (booking_id, rating, comment)
VALUES
(1, 5, 'Fantastic stay! Highly recommend.');

