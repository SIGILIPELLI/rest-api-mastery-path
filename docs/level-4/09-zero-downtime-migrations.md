# 09 · Zero-Downtime Deployments & Migrations

Shipping a change to a live API without an outage — no dropped
requests, no downtime window, no "maintenance mode" — requires
deliberate technique at both the deployment and database layers.

## Rolling deployments

```
t0: [old, old, old, old]   4 instances, all serving traffic
t1: [new, old, old, old]   1 instance replaced, health-checked before receiving traffic
t2: [new, new, old, old]
t3: [new, new, new, old]
t4: [new, new, new, new]   fully rolled out
```

```bash
curl https://api.example.com/healthz
```

```json
{ "status": "healthy", "version": "2.4.1" }
```

The load balancer only routes traffic to instances passing health
checks; a new instance that fails its health check is never sent live
traffic, and the rollout halts automatically rather than taking the
whole fleet down with a bad build.

## The danger window: two versions running at once

During a rolling deploy, **both old and new code are serving traffic
simultaneously** — for anywhere from seconds to minutes. Any change
that isn't compatible with itself running alongside the previous
version breaks requests during that window, even though the final
state (all-new) is fine.

## The expand-contract pattern for database migrations

Renaming a column outright breaks whichever code version doesn't expect
the new name, for the entire rollout window. The safe path splits it
into independent, individually-safe deploys:

**Expand** — add the new column, write to both, without removing the
old one yet:

```sql
ALTER TABLE orders ADD COLUMN total_amount NUMERIC;
```

```python
def create_order(total):
    db.execute(
        "INSERT INTO orders (total, total_amount) VALUES (%s, %s)",
        total, total  # write both columns during the transition
    )
```

Deploy this. Old and new instances both work fine — old code doesn't
know `total_amount` exists and doesn't need to.

**Migrate** — backfill existing rows, then switch reads to the new
column:

```sql
UPDATE orders SET total_amount = total WHERE total_amount IS NULL;
```

```python
def get_order(id):
    row = db.query("SELECT total_amount AS total FROM orders WHERE id = %s", id)
```

Deploy this only after every row is backfilled — otherwise old rows
read as `null`.

**Contract** — once every instance is running code that only uses
`total_amount`, and enough time has passed that you're confident
nothing external still depends on `total`, drop the old column:

```sql
ALTER TABLE orders DROP COLUMN total;
```

Three independent deploys, each individually safe with both old and
new code running simultaneously — versus one deploy that's broken for
the entire rollout window.

## API-level compatibility during rollout

The same principle applies to the API surface, not just the database:
a response field being added or removed must also survive the mixed-
version window, which is exactly why module 6 (Level 3) insists new
fields are additive and old fields aren't dropped until a full
deprecation cycle completes — a rolling deploy is a live, if brief,
demonstration of the same compatibility rules deprecation policy
enforces over a longer timescale.

## Blue-green deployment: an alternative to rolling

```
Blue (live, v1)   ← 100% traffic
Green (staged, v2) ← 0% traffic, fully deployed, health-checked

# cutover: flip traffic atomically
Blue (v1)          ← 0% traffic
Green (v2)         ← 100% traffic
```

No mixed-version window at all — traffic switches from one complete
environment to another in one step. Trades the rolling deploy's gradual
exposure (a bad build only affects a fraction of instances briefly) for
an instant, all-or-nothing cutover that's trivially reversible (flip
back to blue) if something's wrong.

## Worked example: a real migration end to end

Renaming `orders.total` to `orders.total_amount` across an API with
multiple running instances:

1. **Deploy 1 (expand)**: add `total_amount`, write to both columns,
   API still returns `total` only. Safe during rollout — no reader
   depends on `total_amount` yet.
2. **Backfill**: run the `UPDATE` migration against existing rows,
   off-peak, in batches to avoid locking the table.
3. **Deploy 2 (migrate reads, add field)**: API response now includes
   both `total` and `total_amount`; internal reads use `total_amount`.
4. **Deprecation window** (Level 3, module 6): announce `total` as
   deprecated with a sunset date; monitor which clients still read it.
5. **Deploy 3 (contract)**: once usage of `total` drops to zero (or the
   sunset date passes), remove it from the response and drop the
   column.

## Exercise

1. Why does renaming a column in one step break requests during a
   rolling deploy, even though the deploy fully succeeds in the end?
2. Explain what would go wrong if step 2 (migrate reads) shipped before
   step 1's expand deploy had reached 100% of instances.
3. Compare rolling deployment and blue-green: which limits the blast
   radius of a bad build better, and which makes rollback faster?
4. Design the expand-contract steps for adding a `NOT NULL` constraint
   to an existing nullable column without downtime.
