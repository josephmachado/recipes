# Change Data Capture with Postgres

Demo of CDC with Postgres

## Prerequisites

1. uv 
2. Docker 

## Setup Infrastructure

Use docker to run containers

```bash
docker stop some-postgres
docker rm some-postgres
docker run --name some-postgres -e POSTGRES_PASSWORD=mysecretpassword -p 5432:5432 -d postgres:16


docker stop warehouse
docker rm warehouse
docker run --name warehouse -e POSTGRES_PASSWORD=mysecretpassword -p 5433:5432 -d postgres:16
```

## Setup tables 

Create tables and setup publication and replication slots.

```bash
docker exec -ti some-postgres psql -U postgres -c "DROP TABLE IF EXISTS suppliers;
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
"

docker exec -it some-postgres psql -U postgres -c "ALTER SYSTEM SET wal_level = logical;"
docker restart some-postgres && sleep 5
docker exec -it some-postgres psql -U postgres -c "ALTER ROLE postgres WITH REPLICATION;"
docker exec -it some-postgres psql -U postgres -c "GRANT pg_read_all_data TO postgres;"
docker exec -it some-postgres psql -U postgres -c "CREATE PUBLICATION cdc_example_publication FOR ALL TABLES;"
docker exec -it some-postgres psql -U postgres -c "SELECT pg_create_logical_replication_slot('cdc_example_slot', 'test_decoding');"
docker exec -it some-postgres psql -U postgres -c "SELECT pg_create_logical_replication_slot('cdc_pgoutput_slot_v2', 'pgoutput');"

# Check that supplier table was created and data was inserted 
docker exec -it some-postgres psql -U postgres -c "SELECT * FROM suppliers;"

# Create a warehouse table 
docker exec -it warehouse psql -U postgres -c "
DROP TABLE IF EXISTS dim_suppliers;
CREATE TABLE dim_suppliers (
    dim_supplier_key SERIAL PRIMARY KEY,
    supplier_id INTEGER NOT NULL,
    name VARCHAR(255) NOT NULL,
    address TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    snapshot_ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_ts TIMESTAMP,
    updated_ts TIMESTAMP
);
"
```

## See WAL 

Make changes to `suppliers` table and see if it shows up in the WAL files.

```bash
docker exec -it some-postgres psql -U postgres -c "INSERT INTO suppliers (
    name, 
    address, 
    is_active
) VALUES 
(
    'Solutions Inc.',
    'Japan',
    TRUE
);"

docker exec -it some-postgres psql -U postgres -c "update suppliers set name = 'asia soln' where name = 'Solutions Inc.';"
docker exec -it some-postgres psql -U postgres -c "select * from pg_logical_slot_peek_changes('cdc_example_slot', NULL, NULL, 'include-xids', '0');"
# You will see each INSERT and UPDATE surrounded by a BEGIN -- COMMIT commands, this represents a transaction block
```

### Script to monitor WAL 

We can use a Python script to monitor WAL. 

```python 
#!/usr/bin/env python3
# /// script
# requires-python = ">=2.13"
# dependencies = [
#     "psycopg2-binary",
#     "pypgoutput",
# ]
# ///

import psycopg2
from psycopg2.extras import LogicalReplicationConnection

# Connect to PostgreSQL - corrected connection string
conn = psycopg2.connect("postgresql://postgres:mysecretpassword@localhost:5432/postgres" ,
    connection_factory=LogicalReplicationConnection
)
cur = conn.cursor()

# Start streaming changes
cur.start_replication(slot_name='cdc_example_slot')

try:
    while True:
        msg = cur.read_message()
        if msg:
            print(f"{msg}")
            print("---")
            cur.send_feedback(flush_lsn=msg.data_start)
except KeyboardInterrupt:
    print("Stopping replication...")
finally:
    cur.close()
    conn.close()
```

## ETL Code 

Simple ETL to create a snapshot table:


```python
#!/usr/bin/env python3
# /// script
# requires-python = ">=2.13"
# dependencies = [
#     "psycopg2-binary",
#     "pypgoutput",
# ]
# ///
import psycopg2
import pypgoutput

# Connect to PostgreSQL - corrected connection string
conn = psycopg2.connect("postgresql://postgres:mysecretpassword@localhost:5432/postgres")
cursor = conn.cursor()

# Get data from suppliers table
cursor.execute("SELECT id, name, address, is_active, created_ts, updated_ts FROM suppliers")
rows = cursor.fetchall()
print("#" * 100)
print("INPUT DATA")
for row in rows:
    print(row)
print("#" * 100)

cursor.close()
conn.close()

# Connect to warehouse database
warehouse_conn = psycopg2.connect("postgresql://postgres:mysecretpassword@localhost:5433/postgres")
warehouse_cursor = warehouse_conn.cursor()

for row in rows:
    warehouse_cursor.execute("INSERT INTO dim_suppliers (supplier_id, name, address, is_active, created_ts, updated_ts) VALUES (%s, %s, %s, %s, %s, %s)", row)

warehouse_conn.commit()

warehouse_cursor.execute("SELECT * FROM dim_suppliers")
output_rows = warehouse_cursor.fetchall()

print("#" * 100)
print("OUTPUT DATA")
for row in output_rows:
    print(row)
print("#" * 100)
```
