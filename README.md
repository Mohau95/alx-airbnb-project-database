# alx-airbnb-project-database
#!/bin/bash

set -e

# Prompt for PNG file path
read -p "Path to erd.png: " ERD_PNG

# Validate PNG file
if [ ! -f "$ERD_PNG" ]; then
    echo "Error: File $ERD_PNG not found."
    exit 1
fi

# Create directories
mkdir -p ERD database-script-0x01 database-script-0x02

# Copy PNG file
cp "$ERD_PNG" ERD/erd.png

# 0. Define Entities and Relationships in ER Diagram
echo "# Entity-Relationship Diagram Requirements

## Entities and Attributes
- **User**:
  - user_id (UUID, PK)
  - email (VARCHAR, unique)
  - password (VARCHAR)
  - name (VARCHAR)
  - role (ENUM: guest, host, admin)
  - photo (VARCHAR)
  - contact_info (VARCHAR)
  - created_at (TIMESTAMP)
- **Property**:
  - property_id (UUID, PK)
  - host_id (UUID, FK to User)
  - title (VARCHAR)
  - description (TEXT)
  - location (VARCHAR)
  - price (DECIMAL)
  - amenities (JSONB)
  - availability (JSONB)
  - created_at (TIMESTAMP)
- **Booking**:
  - booking_id (UUID, PK)
  - property_id (UUID, FK to Property)
  - guest_id (UUID, FK to User)
  - start_date (DATE)
  - end_date (DATE)
  - status (ENUM: pending, confirmed, canceled, completed)
  - created_at (TIMESTAMP)
- **Payment**:
  - payment_id (UUID, PK)
  - booking_id (UUID, FK to Booking)
  - user_id (UUID, FK to User)
  - amount (DECIMAL)
  - currency (VARCHAR)
  - status (ENUM: pending, completed, refunded)
  - created_at (TIMESTAMP)
- **Review**:
  - review_id (UUID, PK)
  - booking_id (UUID, FK to Booking)
  - user_id (UUID, FK to User)
  - property_id (UUID, FK to Property)
  - rating (INTEGER, 1-5)
  - comment (TEXT)
  - created_at (TIMESTAMP)

## Relationships
- User to Property: One-to-Many (host_id references user_id).
- User to Booking: One-to-Many (guest_id references user_id).
- Property to Booking: One-to-Many (property_id references property_id).
- Booking to Payment: One-to-One (booking_id references booking_id).
- Booking to Review: One-to-One (booking_id references booking_id).
- User to Review: One-to-Many (user_id references user_id).
- Property to Review: One-to-Many (property_id references property_id)." > ERD/requirements.md

# 1. Normalize Your Database Design
echo "# Normalize Your Database Design

## Normalization Steps
### 1NF
- Issue: Property.amenities and Property.availability are arrays.
- Resolution: Use JSONB (PostgreSQL) for atomicity, as JSONB supports querying.
- All tables have primary keys (UUID).
- Result: Schema is in 1NF.

### 2NF
- All tables use single-column primary keys (UUID).
- Non-key attributes (e.g., email, price) depend on primary keys.
- Result: Schema is in 2NF.

### 3NF
- No transitive dependencies (e.g., no derived total_cost in Booking).
- Non-key attributes depend only on primary keys.
- Result: Schema is in 3NF.

## Final Schema
- Tables: User, Property (JSONB for amenities, availability), Booking, Payment, Review.
- Foreign keys ensure data consistency." > normalization.md

# 2. Design Database Schema (DDL)
echo "# Database Schema
See [schema.sql](schema.sql) for DDL statements." > database-script-0x01/README.md

echo "CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    role VARCHAR(20) CHECK (role IN ('guest', 'host', 'admin')) NOT NULL,
    photo VARCHAR(255),
    contact_info VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE properties (
    property_id UUID PRIMARY KEY,
    host_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    location VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL CHECK (price > 0),
    amenities JSONB,
    availability JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE bookings (
    booking_id UUID PRIMARY KEY,
    property_id UUID REFERENCES properties(property_id) ON DELETE CASCADE,
    guest_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL CHECK (end_date > start_date),
    status VARCHAR(20) CHECK (status IN ('pending', 'confirmed', 'canceled', 'completed')) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE payments (
    payment_id UUID PRIMARY KEY,
    booking_id UUID REFERENCES bookings(booking_id) ON DELETE CASCADE UNIQUE,
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    amount DECIMAL(10,2) NOT NULL CHECK (amount > 0),
    currency VARCHAR(3) NOT NULL,
    status VARCHAR(20) CHECK (status IN ('pending', 'completed', 'refunded')) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE reviews (
    review_id UUID PRIMARY KEY,
    booking_id UUID REFERENCES bookings(booking_id) ON DELETE CASCADE UNIQUE,
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    property_id UUID REFERENCES properties(property_id) ON DELETE CASCADE,
    rating INTEGER NOT NULL CHECK (rating >= 1 AND rating <= 5),
    comment TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_properties_host_id ON properties(host_id);
CREATE INDEX idx_properties_location ON properties(location);
CREATE INDEX idx_bookings_property_id ON bookings(property_id);
CREATE INDEX idx_bookings_guest_id ON bookings(guest_id);
CREATE INDEX idx_bookings_dates ON bookings(start_date, end_date);
CREATE INDEX idx_payments_booking_id ON payments(booking_id);
CREATE INDEX idx_reviews_booking_id ON reviews(booking_id);
CREATE INDEX idx_reviews_property_id ON reviews(property_id);" > database-script-0x01/schema.sql

# 3. Seed the Database with Sample Data
echo "# Database Seed Data
See [seed.sql](seed.sql) for INSERT statements." > database-script-0x02/README.md

echo "-- Insert users
INSERT INTO users (user_id, email, password, name, role, photo, contact_info, created_at)
VALUES
    ('550e8400-e29b-41d4-a716-446655440000', 'guest1@example.com', 'hashed_password_1', 'John Doe', 'guest', 'https://s3.amazonaws.com/photos/guest1.jpg', '123-456-7890', CURRENT_TIMESTAMP),
    ('550e8400-e29b-41d4-a716-446655440001', 'host1@example.com', 'hashed_password_2', 'Jane Smith', 'host', 'https://s3.amazonaws.com/photos/host1.jpg', '987-654-3210', CURRENT_TIMESTAMP),
    ('550e8400-e29b-41d4-a716-446655440002', 'guest2@example.com', 'hashed_password_3', 'Alice Brown', 'guest', NULL, '555-123-4567', CURRENT_TIMESTAMP),
    ('550e8400-e29b-41d4-a716-446655440003', 'host2@example.com', 'hashed_password_4', 'Bob Wilson', 'host', NULL, '444-987-6543', CURRENT_TIMESTAMP),
    ('550e8400-e29b-41d4-a716-446655440004', 'admin@example.com', 'hashed_password_5', 'Admin User', 'admin', NULL, '111-222-3333', CURRENT_TIMESTAMP);

-- Insert properties
INSERT INTO properties (property_id, host_id, title, description, location, price, amenities, availability, created_at)
VALUES
    ('6b3b9c2d-4e5f-46a7-9123-59e7f8c9d123', '550e8400-e29b-41d4-a716-446655440001', 'Cozy Beach House', 'Beachfront property.', 'Miami, FL', 150.00, '{\"wifi\": true, \"pool\": true}', '[{\"start\": \"2025-09-01\", \"end\": \"2025-12-31\"}]', CURRENT_TIMESTAMP),
    ('6b3b9c2d-4e5f-46a7-9123-59e7f8c9d124', '550e8400-e29b-41d4-a716-446655440001', 'Downtown Apartment', 'Modern apartment.', 'New York, NY', 200.00, '{\"wifi\": true, \"parking\": true}', '[{\"start\": \"2025-10-01\", \"end\": \"2025-11-30\"}]', CURRENT_TIMESTAMP),
    ('6b3b9c2d-4e5f-46a7-9123-59e7f8c9d125', '550e8400-e29b-41d4-a716-446655440003', 'Mountain Cabin', 'Rustic cabin.', 'Aspen, CO', 120.00, '{\"fireplace\": true}', '[{\"start\": \"2025-11-01\", \"end\": \"2026-02-28\"}]', CURRENT_TIMESTAMP);

-- Insert bookings
INSERT INTO bookings (booking_id, property_id, guest_id, start_date, end_date, status, created_at)
VALUES
    ('7c4d0e3e-5f60-47b8-9234-6af8g9d0e234', '6b3b9c2d-4e5f-46a7-9123-59e7f8c9d123', '550e8400-e29b-41d4-a716-446655440000', '2025-09-10', '2025-09-15', 'confirmed', CURRENT_TIMESTAMP),
    ('7c4d0e3e-5f60-47b8-9234-6af8g9d0e235', '6b3b9c2d-4e5f-46a7-9123-59e7f8c9d124', '550e8400-e29b-41d4-a716-446655440002', '2025-10-05', '2025-10-10', 'pending', CURRENT_TIMESTAMP),
    ('7c4d0e3e-5f60-47b8-9234-6af8g9d0e236', '6b3b9c2d-4e5f-46a7-9123-59e7f8c9d125', '550e8400-e29b-41d4-a716-446655440000', '2025-11-15', '2025-11-20', 'canceled', CURRENT_TIMESTAMP);

-- Insert payments
INSERT INTO payments (payment_id, booking_id, user_id, amount, currency, status, created_at)
VALUES
    ('8d5e1f4f-6g71-48c9-9345-7bg9h0e1f345', '7c4d0e3e-5f60-47b8-9234-6af8g9d0e234', '550e8400-e29b-41d4-a716-446655440000', 750.00, 'USD', 'completed', CURRENT_TIMESTAMP),
    ('8d5e1f4f-6g71-48c9-9345-7bg9h0e1f346', '7c4d0e3e-5f60-47b8-9234-6af8g9d0e235', '550e8400-e29b-41d4-a716-446655440002', 1000.00, 'USD', 'pending', CURRENT_TIMESTAMP),
    ('8d5e1f4f-6g71-48c9-9345-7bg9h0e1f347', '7c4d0e3e-5f60-47b8-9234-6af8g9d0e236', '550e8400-e29b-41d4-a716-446655440000', 600.00, 'USD', 'refunded', CURRENT_TIMESTAMP);

-- Insert reviews
INSERT INTO reviews (review_id, booking_id, user_id, property_id, rating, comment, created_at)
VALUES
    ('9e6f2g5g-7h82-49d0-a456-8ch0i1f2g456', '7c4d0e3e-5f60-47b8-9234-6af8g9d0e234', '550e8400-e29b-41d4-a716-446655440000', '6b3b9c2d-4e5f-46a7-9123-59e7f8c9d123', 4, 'Great beach view!', CURRENT_TIMESTAMP),
    ('9e6f2g5g-7h82-49d0-a456-8ch0i1f2g457', '7c4d0e3e-5f60-47b8-9234-6af8g9d0e235', '550e8400-e29b-41d4-a716-446655440002', '6b3b9c2d-4e5f-46a7-9123-59e7f8c9d124', 5, 'Very clean.', CURRENT_TIMESTAMP);" > database-script-0x02/seed.sql

echo "Success: All files created in alx-airbnb-database."
