# Change Data Capture with Postgres

Demo of CDC with Postgres

## Setup Postgres Tables

Create a `.sql` file to create the table to do CDC on and setup WAL blocks as well.

```sql
-- save this as a file cdc_pg_setup.sql

DROP TABLE IF EXISTS suppliers;
CREATE TABLE suppliers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    address TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    created_ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data with simplified columns
INSERT INTO suppliers (
    name, 
    address, 
    is_active
) VALUES 
(
    'Tech Solutions Inc.',
    '1234 Innovation Drive, Suite 200, San Francisco, CA 94105, USA',
    TRUE
),
(
    'Global Supply Co.',
    '789 Commerce Boulevard, Chicago, IL 60601, USA',
    TRUE
);

-- SCD2 dimension table with simplified columns
DROP TABLE IF EXISTS dim_suppliers;
CREATE TABLE dim_suppliers (
    dim_supplier_key SERIAL PRIMARY KEY,
    supplier_id INTEGER NOT NULL,
    name VARCHAR(255) NOT NULL,
    address TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    valid_from TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    valid_to TIMESTAMP DEFAULT '9999-12-31 23:59:59',
    is_current BOOLEAN DEFAULT TRUE,
    created_ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Logical replication setup for CDC 
ALTER SYSTEM SET wal_level = logical;

ALTER ROLE postgres WITH REPLICATION;
GRANT pg_read_all_data TO postgres; 

ALTER TABLE suppliers REPLICA IDENTITY FULL;

CREATE PUBLICATION cdc_example_publication FOR ALL TABLES;
SELECT pg_create_logical_replication_slot('cdc_example_slot', 'test_decoding');
```

Now start a postgres container with docker:

```bash
docker stop some-postgres
docker rm some-postgres

# Setup postgres db with the init sql query
docker run --name some-postgres -e POSTGRES_PASSWORD=mysecretpassword -v ./cdc_pg_setup.sql:/docker-entrypoint-initdb.d/init.sql -p 5432:5432 -d postgres:16
```

## Check WAL

Once the postgres container is running, insert some data and query the WAL as shown below:

```bash
docker exec -it some-postgres psql -U postgres -d postgres # start psql cli
```

```sql
-- simple insert check 
-- Generate some test data
CREATE TABLE test_wal (id int, data text);
INSERT INTO test_wal VALUES (1, 'test data');

ALTER TABLE test_wal REPLICA IDENTITY FULL;
-- Check the WAL output
SELECT * FROM pg_logical_slot_get_changes('cdc_example_slot', NULL, NULL);

-- Once you read it will be moved from WAL to disk, so a subsequent query as below will return 0 rows, unless you CUD other tables
SELECT * FROM pg_logical_slot_get_changes('cdc_example_slot', NULL, NULL);
```

## See WAL changes with new Insert-Delete-Updates

### COMMIT and ROLLBACKS
 
All the CUD commands are written to WAL, however there are cases where such commands may not be committed what happens in such a case?

Let's update some data, but not commit it:

```bash
docker exec -it some-postgres psql -U postgres -d postgres # start psql cli
```

```sql
Update test_wal set data = 'new test data' where id = 1; 

Update test_wal set data = 'new test data 2' where id = 1; 

SELECT * FROM pg_logical_slot_get_changes('cdc_example_slot', NULL, NULL);
```

psql auto commits, so the updates will show up in WAL  with BEGIN & COMMIT.

**Note** Only committed transactions will show up in the WAL
