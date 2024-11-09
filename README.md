# Tutorial: PostgreSQL Row Level Security (RLS)

This tutorial demonstrates how to implement and test Row Level Security in PostgreSQL for multi-tenant data isolation.

## Prerequisites
- Docker and Docker Compose
- PostgreSQL knowledge
- pgAdmin (for database management)

## Preparation

First, start PostgreSQL using Docker Compose:

```
docker compose up -d
```

Once running, access pgAdmin through your web browser at http://localhost:8080/

### Setting up the Database Structure

The following SQL creates our sample table and implements basic multi-tenant data structure:

```sql
-- Create the table
create table employees (
  id bigint primary key generated always as identity,
  name text not null,
  tenant_id bigint not null
);

-- Insert some sample data into the table
insert into employees (name, tenant_id)
values
  ('John', 1),
  ('Rick', 1),
  ('Mike', 2);

alter table employees enable row level security;
```

### Implementing Row Level Security Policy

Create a policy that filters data based on the current tenant ID:

```sql
create policy "tenant_isolation_policy" on public.employees
using (tenant_id = CURRENT_SETTING('app.current_tenant_id')::INTEGER);
```

### Creating a Test User

Set up a user with standard read/write permissions to test the RLS implementation:

```sql
CREATE USER readwrite_user WITH LOGIN PASSWORD 'password';
GRANT USAGE ON SCHEMA public TO readwrite_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO readwrite_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO readwrite_user;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO readwrite_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT USAGE ON SEQUENCES TO readwrite_user;
```

## Testing Row Level Security

### Testing as Superuser

When querying as a superuser, all records are visible:

```sql
SELECT * FROM public.employees
ORDER BY id ASC LIMIT 100
```

Result shows all records across tenants:
```
"id"	"name"	"tenant_id"
1	"John"	1
2	"Rick"	1
3	"Mike"	2
```

Note: This comprehensive view is available because superusers and roles with BYPASSRLS attribute automatically bypass row security. Table owners also typically bypass row security unless explicitly configured otherwise through `ALTER TABLE ... FORCE ROW LEVEL SECURITY`.

### Testing as Regular User

Initial attempt as readwrite_user fails because the tenant context isn't set:

```sql
SELECT * FROM public.employees
ORDER BY id ASC LIMIT 100
```

Results in:
```
ERROR:  unrecognized configuration parameter "app.current_tenant_id"
```

### Testing with Tenant Context

Setting tenant context to tenant_id = 1:

```sql
select set_config('app.current_tenant_id', '1', false);

SELECT * FROM public.employees
ORDER BY id ASC LIMIT 100
```

Shows only tenant 1's data:
```
"id"	"name"	"tenant_id"
1	"John"	1
2	"Rick"	1
```

Switching to tenant_id = 2:

```sql
select set_config('app.current_tenant_id', '2', false);

SELECT * FROM public.employees
ORDER BY id ASC LIMIT 100
```

Shows only tenant 2's data:
```
"id"	"name"	"tenant_id"
3	"Mike"	2
```

This demonstrates successful implementation of row-level security, where users can only access data belonging to their assigned tenant.
