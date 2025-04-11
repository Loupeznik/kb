# Postgres

## Assign privileges to all objects in schema

The following assigns all privileges on all objects in non-system schemas in the current database. Can be adjusted to assign only certain (i.e. READ) privileges.
Swap `mydb` and `grp_mydb` placeholders for database and role names respectivelly.

```sql
GRANT CONNECT ON DATABASE mydb TO grp_mydb;

DO
$$
DECLARE
    r RECORD;
BEGIN
    FOR r IN
        SELECT schema_name
        FROM information_schema.schemata
        WHERE schema_name NOT IN ('information_schema', 'pg_catalog')
    LOOP
        -- Allow the user to use the schema
        EXECUTE format('GRANT USAGE ON SCHEMA %I TO %I;', r.schema_name, 'grp_mydb');

        -- Grant all privileges on all tables in the schema
        EXECUTE format('GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA %I TO %I;', r.schema_name, 'grp_mydb');

        -- Grant all privileges on all sequences in the schema (if any)
        EXECUTE format('GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA %I TO %I;', r.schema_name, 'grp_mydb');

        -- Optionally, grant privileges on functions if needed
        EXECUTE format('GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA %I TO %I;', r.schema_name, 'grp_mydb');

		EXECUTE format('ALTER DEFAULT PRIVILEGES IN SCHEMA %I GRANT ALL PRIVILEGES ON TABLES TO %I;', r.schema_name, 'grp_mydb');
    END LOOP;
END
$$;
```
