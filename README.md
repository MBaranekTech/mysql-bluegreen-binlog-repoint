# RDS MySQL Blue/Green Upgrade with External Binlog Replica Re-Point

A reproducible runbook for performing a **major-version upgrade** of an Amazon RDS for MySQL
instance using a **Blue/Green Deployment**, while keeping an **external (self-managed)
binlog replica** in sync across the switchover — **without data loss**.

> Example used throughout: upgrade RDS for MySQL `8.0.x` → `8.4.x`, with a local
> downstream replica replicating via classic **file + position** (`Auto_Position = 0`).

---

## Why this is not automatic

In an RDS Blue/Green Deployment the **green** instance is a *new* instance with its **own
binary-log sequence**. At switchover, the production endpoint flips to green and the old
blue instance is renamed with an `-oldN` suffix and made read-only.

An external downstream replica that reads blue's binlog **cannot follow this on its own**:
its stored coordinates (`blue-file` / `blue-position`) do not exist on green. The replica
must be **manually re-pointed** after switchover.

The core challenge with file+position replication is doing this **without creating a gap
(silent data loss) or duplicating transactions**. The procedure below solves it by first
fully draining the old blue instance, then switching to green's post-switchover coordinates.

> **If you use GTID** (`SOURCE_AUTO_POSITION = 1`) this is dramatically simpler — see
> [GTID alternative](#gtid-the-simpler-long-term-fix).

---

## Topology

```
                 binlog replication (file + position)
  ┌────────────┐  ───────────────────────────────────▶  ┌──────────────────┐
  │  RDS blue  │                                          │  external replica │
  │  (source)  │                                          │  (self-managed)   │
  └────────────┘                                          └──────────────────┘
```

After switchover:

```
  ┌────────────┐   (read-only, frozen binlog)
  │  -old1     │ ◀── step 2: drain remaining events
  └────────────┘
  ┌────────────┐   (new production, own binlog sequence)
  │  green     │ ◀── step 3: final re-point (coordinates from RDS Events)
  └────────────┘
```

---

## Prerequisites

- [ ] **Binlog retention** on the source is high enough to cover the whole operation:
  ```sql
  CALL mysql.rds_set_configuration('binlog retention hours', 48);
  CALL mysql.rds_show_configuration;   -- verify
  ```
- [ ] **`mysql_native_password` is loaded on green** (the target 8.4 parameter group).
  In 8.4 `mysql_native_password` is **disabled by default** and is **not a system
  variable**, so `SHOW VARIABLES LIKE 'mysql_native_password'` always returns nothing and
  tells you nothing. Check the **plugin status** instead:
  ```sql
  SELECT plugin_name, plugin_status
  FROM information_schema.plugins
  WHERE plugin_name = 'mysql_native_password';
  -- ACTIVE = OK; no rows = not loaded (fix parameter group + reboot)
  ```
- [ ] The replication user (e.g. `repl`) exists on the source and its auth plugin matches
  what green supports:
  ```sql
  SELECT user, host, plugin FROM mysql.user WHERE user = 'repl';
  ```
- [ ] Replication is **healthy** on the external replica:
  ```sql
  SHOW REPLICA STATUS\G
  -- expect: Replica_IO_Running: Yes, Replica_SQL_Running: Yes,
  --         Seconds_Behind_Source: 0, no Last_Error / Last_IO_Error
  ```
- [ ] You have an **admin/master DB user** (not the replication user) for running
  administrative queries.
- [ ] Network path from the replica to the **new `-old1` endpoint** is open (same VPC/SG as
  the original endpoint — the `-old1` hostname is new and only appears after switchover).

---

## Procedure

### Phase 1 — Switchover

1. Trigger the switchover from the RDS console (Blue/Green Deployment → **Switchover**).
   RDS will:
   - wait for green replica lag to reach ~0,
   - briefly block writes on both environments (guaranteeing no data loss),
   - promote green to production and flip the production endpoint to it,
   - rename the old blue instance to `<identifier>-old1` (read-only).
2. Wait for the **"Switchover completed"** event.

> **Do not stop the external replica before switchover.** Leaving it running means it
> drains as much of blue as possible up front, leaving less to catch up afterwards. Its
> IO thread will fail on its own right after switchover — that is expected.

> **Note the new `-old1` endpoint** from the RDS console (Databases → the `-old1`
> instance → Connectivity & security → Endpoint). You cannot know it in advance.

### Phase 2 — Drain the remaining events from `-old1`

The `-old1` instance is read-only after switchover, so its binlog is **frozen** at the
final position blue ever wrote.

1. On the external replica, stop and record the current position:
   ```sql
   STOP REPLICA;
   SHOW REPLICA STATUS\G
   -- record Relay_Source_Log_File and Exec_Source_Log_Pos
   ```
2. Re-point the replica to `-old1` using the **existing** coordinates (only the host
   changes):
   ```sql
   CHANGE REPLICATION SOURCE TO
     SOURCE_HOST='<OLD1_ENDPOINT>',
     SOURCE_PORT=3306,
     SOURCE_USER='repl',
     SOURCE_PASSWORD='<password>',
     SOURCE_LOG_FILE='<Relay_Source_Log_File from step 1>',
     SOURCE_LOG_POS=<Exec_Source_Log_Pos from step 1>;
   START REPLICA;
   ```
3. Get the final, frozen position of `-old1`. On MySQL 8.0 use `SHOW MASTER STATUS`; on 8.4
   use `SHOW BINARY LOG STATUS`:
   ```sql
   SHOW MASTER STATUS;   -- 8.0 (-old1)
   -- record File and Position; run twice to confirm it is stable (read-only)
   ```
4. **Verify the replica is fully drained** — the hard check, not just `Seconds_Behind_Source`:
   ```sql
   SHOW REPLICA STATUS\G
   ```
   The replica is fully caught up when **both** of the following hold:
   - `Replica_SQL_Running_State: Replica has read all relay log; waiting for more updates`
   - `Relay_Source_Log_File` + `Exec_Source_Log_Pos` **equal** `-old1`'s `File` + `Position`

   > Compare against `Exec_Source_Log_Pos` (position in the **source** binlog), **not**
   > `Relay_Log_Pos` (position in the local relay log — a different number).
   >
   > A read-only RDS instance may rotate to a fresh, near-empty binlog file (header only,
   > e.g. position `157`). If `-old1` is one file ahead, wait for the replica to read the
   > rotate event and land on the same file/position before proceeding.

### Phase 3 — Final re-point to green

1. Get green's post-switchover binlog coordinates from the **RDS console → Events**, filtered
   by the old green instance name. Look for:
   > `Binary log coordinates in green environment after switchover: file mysql-bin-changelog.NNNNNN and position PPPPPP`

   > **Use exactly these coordinates.** Do **not** read green's *current* position via
   > `SHOW BINARY LOG STATUS` — green is already taking application writes and has moved on,
   > which would create a gap.
2. Re-point the replica to green (production endpoint, which now resolves to green):
   ```sql
   STOP REPLICA;
   CHANGE REPLICATION SOURCE TO
     SOURCE_HOST='<PROD_ENDPOINT>',
     SOURCE_PORT=3306,
     SOURCE_USER='repl',
     SOURCE_PASSWORD='<password>',
     SOURCE_LOG_FILE='mysql-bin-changelog.NNNNNN',
     SOURCE_LOG_POS=PPPPPP;
   START REPLICA;
   SHOW REPLICA STATUS\G
   ```
3. Verify:
   - `Replica_IO_Running: Yes`, `Replica_SQL_Running: Yes`
   - no `Last_IO_Error` / `Last_SQL_Error`
   - `Source_Log_File` is now on green's sequence (a different number range than blue)
   - `Seconds_Behind_Source` decreasing toward 0

### Phase 4 — Finalize

1. Re-enable application writes if they were paused (they now target green).
2. Set/verify binlog retention on green:
   ```sql
   CALL mysql.rds_set_configuration('binlog retention hours', 48);
   ```
3. Monitor the replica for 30–60 minutes: lag stays at 0, no errors.
4. **Only after** confirming health, delete the Blue/Green Deployment and the `-old1`
   instance. Until then `-old1` is your rollback safety net — **do not delete it**.

---

## Why the ordering matters

Green's emitted coordinate represents the green-side equivalent of blue's **final** state
(this is why RDS can guarantee no data loss at switchover — green is fully caught up to
blue's last write). By draining `-old1` completely **first**, the external replica reaches
that exact same logical state. Re-pointing to green's emitted coordinate then resumes
seamlessly: no gap, no duplicates.

Re-pointing to green **before** the replica has drained `-old1` would leave the
un-applied transactions sitting *before* the emitted coordinate in green's binlog — they
would never be read again. **Silent data loss.**

---

## External replica state across the phases

| Phase            | `SOURCE_HOST`                  | `SOURCE_LOG_FILE` / `POS`              |
|------------------|--------------------------------|----------------------------------------|
| Before switchover| production endpoint (→ blue)    | current blue position                  |
| Phase 2 (drain)  | `-old1` endpoint                | existing position (recorded in 2.1)    |
| Phase 3 (final)  | production endpoint (→ green)   | green emitted coordinate (from Events) |

---

## Common pitfalls

- **`SHOW VARIABLES LIKE 'mysql_native_password'` returning nothing is normal** in 8.4 — it
  is not a system variable. Check `information_schema.plugins` instead.
- **Wrong verification field.** Compare the source-side `Exec_Source_Log_Pos`, not the local
  `Relay_Log_Pos`.
- **`Seconds_Behind_Source: 0` is not proof of full drain**, especially on fast hardware —
  it can read 0 while the last events are still being applied. Use the file+position match.
- **Stopping the replica before switchover gains nothing** — the re-point steps are
  unavoidable afterwards either way.
- **`SHOW MASTER STATUS` is removed in 8.4** — use `SHOW BINARY LOG STATUS`.
- **Reading green's *live* position instead of the Events coordinate** creates a gap.
- **Deleting `-old1` too early** removes your only rollback path.

---

## GTID: the simpler long-term fix

This entire manual dance is the cost of **file + position** replication. With GTID enabled
(`SOURCE_AUTO_POSITION = 1`), the post-switchover re-point becomes a single statement —
point the replica at the production endpoint and it figures out exactly what it is missing,
with no coordinate hunting and no gap/duplicate risk:

```sql
STOP REPLICA;
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='<PROD_ENDPOINT>',
  SOURCE_AUTO_POSITION=1;
START REPLICA;
```

AWS recommends enabling GTID **before** creating the Blue/Green Deployment so green inherits
the setting. Enabling GTID is itself a staged change (`gtid_mode`:
`OFF → OFF_PERMISSIVE → ON_PERMISSIVE → ON`, plus `enforce_gtid_consistency`) on both source
and replica, so plan it as a separate maintenance — not as part of the upgrade itself.

---

## Quick reference — verification queries

```sql
-- replica health
SHOW REPLICA STATUS\G

-- source final position
SHOW MASTER STATUS;          -- MySQL 8.0
SHOW BINARY LOG STATUS;      -- MySQL 8.4

-- native password plugin (8.4)
SELECT plugin_name, plugin_status
FROM information_schema.plugins
WHERE plugin_name = 'mysql_native_password';

-- replication user / auth plugin
SELECT user, host, plugin FROM mysql.user WHERE user = 'repl';

-- RDS binlog retention
CALL mysql.rds_show_configuration;
CALL mysql.rds_set_configuration('binlog retention hours', 48);
```

---

## Notes

- All terminology uses the modern `SOURCE` / `REPLICA` keywords (required on 8.4; the legacy
  `MASTER` / `SLAVE` keywords are removed).
- Replace all `<PLACEHOLDER>` values with your own. Never commit real passwords or private
  endpoints to a repository.
