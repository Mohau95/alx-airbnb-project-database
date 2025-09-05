# alx-airbnb-project-database
# 1. Database Normalization
echo "# Database Normalization
This file explains the steps taken to normalize the Airbnb database to Third Normal Form (3NF).

See [normalization.md](normalization.md) for details." > database-script-0x01/README.md

# 2. Database Schema (DDL)
echo "# Database Schema (DDL)
This directory contains SQL scripts to define the Airbnb database schema including tables, primary keys, foreign keys, and indexes.

See [schema.sql](schema.sql) for the SQL statements." > database-script-0x01/README.md

# 3. Seed Data
echo "# Database Seed Data
This directory contains SQL scripts to populate the Airbnb database with sample data.

See [seed.sql](seed.sql) for the SQL insert statements." > database-script-0x02/README.md

# Create placeholder Markdown and SQL files

# Normalization explanation
echo "# Normalization Steps
- Applied 1NF: Ensured atomic values in all columns.
- Applied 2NF: Removed partial dependencies from tables.
- Applied 3NF: Removed transitive dependencies, ensuring all non-key attributes depend only on the primary key." > normalization.md

# Database schema placeholder
echo "-- SQL CREATE TABLE statements for Airbnb Database
-- Users, Properties, Bookings, Reviews, Payments
-- Add proper data types, primary keys, foreign keys, and constraints" > database-script-0x01/schema.sql

# Sample data placeholder
echo "-- SQL INSERT statements to populate Airbnb Database with sample data
-- Include multiple users, properties, bookings, reviews, and payments" > database-script-0x02/seed.sql

echo "Airbnb Database project structure created successfully."
