-- Schema for ALX Airbnb-like project (PostgreSQL)
-- Save as: database-script-0x01/schema.sql

-- Make idempotent and enable UUID generator (pgcrypto)
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Users
CREATE TABLE IF NOT EXISTS users (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  full_name   text        NOT NULL,
  email       text        NOT NULL UNIQUE,
  phone       text        UNIQUE,
  is_host     boolean     NOT NULL DEFAULT false,
  created_at  timestamp   NOT NULL DEFAULT now()
);

-- Properties
CREATE TABLE IF NOT EXISTS properties (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  host_id       uuid        NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
  title         text        NOT NULL,
  description   text        NOT NULL,
  property_type text        NOT NULL,
  nightly_price numeric(10,2) NOT NULL CHECK (nightly_price >= 0),
  currency      char(3)     NOT NULL,
  max_guests    integer     NOT NULL CHECK (max_guests > 0),
  street        text        NOT NULL,
  city          text        NOT NULL,
  state         text,
  country       text        NOT NULL,
  postal_code   text,
  latitude      numeric(9,6),
  longitude     numeric(9,6),
  created_at    timestamp   NOT NULL DEFAULT now()
);
CREATE INDEX IF NOT EXISTS idx_properties_host ON properties(host_id);
CREATE INDEX IF NOT EXISTS idx_properties_city ON properties(city);

-- Property Images
CREATE TABLE IF NOT EXISTS property_images (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  property_id  uuid      NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
  url          text      NOT NULL,
  is_cover     boolean   NOT NULL DEFAULT false,
  position     integer   NOT NULL DEFAULT 0
);
CREATE INDEX IF NOT EXISTS idx_property_images_property ON property_images(property_id);

-- Amenities
CREATE TABLE IF NOT EXISTS amenities (
  id    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name  text NOT NULL UNIQUE
);

-- Property ↔ Amenities (junction)
CREATE TABLE IF NOT EXISTS property_amenities (
  property_id uuid NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
  amenity_id  uuid NOT NULL REFERENCES amenities(id) ON DELETE RESTRICT,
  PRIMARY KEY (property_id, amenity_id)
);
CREATE INDEX IF NOT EXISTS idx_property_amenities_amenity ON property_amenities(amenity_id);

-- Bookings
CREATE TABLE IF NOT EXISTS bookings (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  property_id   uuid      NOT NULL REFERENCES properties(id) ON DELETE RESTRICT,
  guest_id      uuid      NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
  check_in      date      NOT NULL,
  check_out     date      NOT NULL,
  guests_count  integer   NOT NULL CHECK (guests_count > 0),
  status        text      NOT NULL CHECK (status IN ('pending','confirmed','cancelled','completed')),
  total_amount  numeric(10,2) NOT NULL CHECK (total_amount >= 0),
  currency      char(3)   NOT NULL,
  created_at    timestamp NOT NULL DEFAULT now(),
  CONSTRAINT chk_dates CHECK (check_out > check_in)
);
CREATE INDEX IF NOT EXISTS idx_bookings_property_dates ON bookings(property_id, check_in, check_out);
CREATE INDEX IF NOT EXISTS idx_bookings_guest ON bookings(guest_id);

-- Payments (MVP: 1 payment per booking)
CREATE TABLE IF NOT EXISTS payments (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_id      uuid      NOT NULL UNIQUE REFERENCES bookings(id) ON DELETE CASCADE,
  amount          numeric(10,2) NOT NULL CHECK (amount >= 0),
  currency        char(3)   NOT NULL,
  method          text      NOT NULL,
  status          text      NOT NULL CHECK (status IN ('pending','paid','failed','refunded')),
  provider_txn_id text      UNIQUE,
  paid_at         timestamp,
  created_at      timestamp NOT NULL DEFAULT now()
);
CREATE INDEX IF NOT EXISTS idx_payments_booking ON payments(booking_id);

-- Reviews
CREATE TABLE IF NOT EXISTS reviews (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_id   uuid      NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
  property_id  uuid      NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
  reviewer_id  uuid      NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
  rating       integer   NOT NULL CHECK (rating BETWEEN 1 AND 5),
  comment      text,
  created_at   timestamp NOT NULL DEFAULT now(),
  CONSTRAINT uq_review_once UNIQUE (booking_id, reviewer_id)
);
CREATE INDEX IF NOT EXISTS idx_reviews_property ON reviews(property_id);
-- Seed data for ALX Airbnb-like project (PostgreSQL)
-- Save as: database-script-0x02/seed.sql

-- Make sure extensions are available
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Clear tables in safe order to preserve FK integrity
TRUNCATE reviews, payments, bookings, property_amenities, amenities, property_images, properties, users RESTART IDENTITY CASCADE;

-- Insert users, properties, images, amenities, bookings, payments, reviews
WITH
-- Insert users and capture ids
u AS (
  INSERT INTO users (full_name, email, phone, is_host)
  VALUES
    ('Lerato Mokoena', 'lerato@example.com', '+27-82-000-1001', true),
    ('Thabo Dlamini',  'thabo@example.com',  '+27-83-000-2002', false),
    ('Aisha Khan',     'aisha@example.com',  '+27-84-000-3003', true)
  RETURNING id, email
),

-- Insert properties using hosts by email
p AS (
  -- host: lerato@example.com
  INSERT INTO properties (host_id, title, description, property_type, nightly_price, currency, max_guests, street, city, state, country, postal_code, latitude, longitude)
  SELECT u.id, 'Sunny Loft in Maboneng', 'Open-plan loft with skyline views', 'apartment', 950.00, 'ZAR', 2, '123 Fox St', 'Johannesburg', 'GP', 'South Africa', '2001', -26.2041, 28.0473
  FROM u WHERE u.email = 'lerato@example.com'
  RETURNING id
),
p2 AS (
  -- host: aisha@example.com
  INSERT INTO properties (host_id, title, description, property_type, nightly_price, currency, max_guests, street, city, state, country, postal_code, latitude, longitude)
  SELECT u.id, 'Cape Breeze Cottage', 'Cozy cottage 5 min from the beach', 'house', 1800.00, 'ZAR', 4, '7 Marine Dr', 'Cape Town', 'WC', 'South Africa', '8001', -33.9249, 18.4241
  FROM u WHERE u.email = 'aisha@example.com'
  RETURNING id
),

-- Insert property images (link to properties)
imgs AS (
  INSERT INTO property_images (property_id, url, is_cover, position)
  SELECT id, 'https://pics.example.com/loft1.jpg', true, 1 FROM p
  UNION ALL
  SELECT id, 'https://pics.example.com/loft2.jpg', false, 2 FROM p
  UNION ALL
  SELECT id, 'https://pics.example.com/cottage1.jpg', true, 1 FROM p2
  RETURNING id
),

-- Insert amenities
ams AS (
  INSERT INTO amenities (name)
  VALUES ('Wi-Fi'), ('Parking'), ('Pool'), ('Washer'), ('Kitchen')
  RETURNING id, name
),

-- Link amenities to properties
pa AS (
  -- Wi-Fi & Kitchen for the loft (p)
  INSERT INTO property_amenities (property_id, amenity_id)
  SELECT p.id, a.id FROM p CROSS JOIN ams a WHERE a.name IN ('Wi-Fi','Kitchen')
  UNION ALL
  -- Wi-Fi, Parking, Pool for the cottage (p2)
  SELECT p2.id, a.id FROM p2 CROSS JOIN ams a WHERE a.name IN ('Wi-Fi','Parking','Pool')
  RETURNING property_id, amenity_id
),

-- Insert bookings
b AS (
  -- Find guest ids
  INSERT INTO bookings (property_id, guest_id, check_in, check_out, guests_count, status, total_amount, currency)
  SELECT p.id, u_g.id, DATE '2025-09-15', DATE '2025-09-18', 2, 'confirmed', 950.00 * 3, 'ZAR'
  FROM p CROSS JOIN (SELECT id FROM users WHERE email = 'thabo@example.com') AS u_g
  UNION ALL
  SELECT p2.id, u_g2.id, DATE '2025-10-01', DATE '2025-10-05', 3, 'pending', 1800.00 * 4, 'ZAR'
  FROM p2 CROSS JOIN (SELECT id FROM users WHERE email = 'aisha@example.com') AS u_g2
  RETURNING id
),

-- Insert payments for bookings (one per booking as MVP)
pay AS (
  INSERT INTO payments (booking_id, amount, currency, method, status, provider_txn_id, paid_at)
  SELECT b.id, 2850.00, 'ZAR', 'card', 'paid', 'txn_001', now() FROM b LIMIT 1
  UNION ALL
  SELECT b.id, 7200.00, 'ZAR', 'eft', 'pending', NULL, NULL FROM b OFFSET 1 LIMIT 1
  RETURNING id
)

-- Final SELECT to show counts (optional)
SELECT
  (SELECT count(*) FROM users)      AS users_count,
  (SELECT count(*) FROM properties) AS properties_count,
  (SELECT count(*) FROM property_images) AS images_count,
  (SELECT count(*) FROM amenities)  AS amenities_count,
  (SELECT count(*) FROM property_amenities) AS property_amenities_count,
  (SELECT count(*) FROM bookings)   AS bookings_count,
  (SELECT count(*) FROM payments)   AS payments_count;
