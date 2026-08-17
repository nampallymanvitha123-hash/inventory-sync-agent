# Idempotent Multi-Warehouse Inventory Sync Agent

## Overview

This project demonstrates an inventory synchronisation agent that combines stock movements from two independent warehouse systems into one central ledger.

The main goal is to prevent inventory from being corrupted when:

- Two warehouse systems report the same real-world stock movement.
- A warehouse sends an incorrect movement and later sends a correction.
- The sync agent is run more than once with the same source data.

The agent uses local SQLite databases to simulate Warehouse A, Warehouse B, and a central inventory ledger.

## Problem being solved

Warehouse systems do not always operate independently. For example, the same customer sale may appear in two warehouse feeds. If the central ledger simply imports every source record, it may subtract the same stock twice.

This project avoids that problem by treating stock changes as immutable movements and assigning each real-world movement a deterministic idempotency key. The central ledger accepts each idempotency key only once.

## Architecture

The project contains three SQLite databases:

```text
Warehouse A database
        \
         --> Inventory Sync Agent --> Central Ledger database
        /
Warehouse B database
```

### Warehouse A

Warehouse A contains stock movements with:

- A source sequence number.
- A unique source movement ID.
- Order reference.
- SKU.
- Movement type.
- Quantity.
- Timestamp.
- Optional correction reference.

### Warehouse B

Warehouse B uses the same movement structure but contains the duplicate conflict and correction examples.

### Central ledger

The central ledger contains two tables:

- `ledger_movements`: the authoritative record of accepted inventory movements.
- `sync_watermarks`: the last processed source sequence for each warehouse.

The ledger stores an `idempotency_key` with a `UNIQUE` constraint. This is the final protection against duplicate movements.

## Agent workflow

Every time `sync_inventory()` runs, it performs the following steps:

1. Read the saved watermark for Warehouse A and Warehouse B.
2. Query each warehouse for events with a `source_sequence` greater than its saved watermark.
3. Build a deterministic idempotency key for every new movement.
4. Attempt to insert the movement into the central ledger.
5. If the idempotency key is new, store the movement in the ledger.
6. If the idempotency key already exists, skip the movement as a duplicate.
7. Move each warehouse watermark to the highest successfully processed sequence number.
8. Commit the ledger changes and watermark updates together.

## Watermark-based incremental sync

The agent uses a separate watermark for each warehouse.

For example:

```text
Warehouse A watermark: 2
Warehouse B watermark: 4
```

When the agent runs again, it only reads source events with a sequence number above the saved watermark. This makes the sync incremental instead of rereading every warehouse movement every time.

Watermarks improve efficiency, but they are not the only duplicate protection. The central ledger's unique idempotency key is still necessary as a safety net if an event is resent or a previous batch is replayed.

## Idempotency strategy

The agent is idempotent, meaning that repeated runs with the same input produce the same final ledger state.

For normal stock movements, the idempotency key is created from:

```text
movement type + order reference + SKU + quantity
```

For correction movements, the idempotency key is created from:

```text
correction marker + original movement ID + SKU + correction quantity
```

The combined value is hashed using SHA-256.

This means that if Warehouse A and Warehouse B both report the same sale, they generate the same idempotency key even though their source movement IDs are different.

The central ledger enforces this rule with:

```sql
idempotency_key TEXT NOT NULL UNIQUE
```

The insert uses:

```sql
ON CONFLICT(idempotency_key) DO NOTHING
```

Therefore, a duplicate event cannot create a second ledger movement.

## Conflict scenario

The notebook demonstrates a duplicate stock movement conflict.

Warehouse A reports:

```text
A-001 | ORDER-1001 | SKU-CHAIR | sale | -5
```

Warehouse B reports:

```text
B-001 | ORDER-1001 | SKU-CHAIR | sale | -5
```

These are two source reports for the same real-world sale of five chairs.

Both movements generate the same idempotency key. The agent accepts the first one inserted into the central ledger and skips the second one.

As a result:

```text
Source reports received: 6
Central ledger movements stored: 5
```

The chair sale is recorded once, not twice.

## Correction scenario

Warehouse B also demonstrates an error and correction:

```text
B-003 | ORDER-9999 | SKU-CHAIR | sale             | -3
B-004 | ORDER-9999 | SKU-CHAIR | sale_correction  | +3
```

`B-004` references `B-003` through the `correction_of` field.

Both events are recorded once because the correction is a legitimate separate movement. Their net inventory effect is zero:

```text
-3 + 3 = 0
```

The final stock summary confirms that the only remaining chair reduction is the valid five-chair sale from `ORDER-1001`:

```text
SKU-CHAIR: -5
SKU-DESK:  -2
SKU-LAMP:  20
```

## Demonstrating idempotency

On the first call to:

```python
sync_inventory()
```

the agent processes six source reports:

```text
5 movements applied
1 duplicate skipped
```

On the second call to:

```python
sync_inventory()
```

the agent finds no movements beyond either warehouse watermark:

```text
Warehouse A: no new movements.
Warehouse B: no new movements.
SYNC COMPLETE: 0 applied, 0 duplicates skipped.
```

An assertion verifies that the central ledger still contains exactly five movements after the second run.

This proves the agent does not create:

- Duplicate movements.
- Duplicate corrections.
- Reversed adjustments.
- Extra stock changes from repeated runs.

## How to run

### Requirements

- Python 3
- Jupyter Notebook or JupyterLab
- No external packages are required; the project uses Python's standard library.

### Steps

1. Clone or download this repository.
2. Open the notebook in Jupyter:

```text
inventory-sync-agent.ipynb
```

3. Run all cells from top to bottom.
4. Run the first sync:

```python
sync_inventory()
```

5. Inspect the central ledger.
6. Run the sync a second time:

```python
sync_inventory()
```

7. Confirm that no additional movements are created.
8. Run the stock summary query to confirm that corrections produce the expected net stock values.

## Key design decisions

- **Movement ledger instead of current stock snapshots:** movements provide an audit trail and make it possible to identify duplicates and corrections.
- **Independent warehouse sources:** Warehouse A and Warehouse B are separate SQLite databases, representing independent source systems.
- **Source sequence numbers:** sequence numbers support per-warehouse incremental sync through watermarks.
- **Deterministic idempotency keys:** equivalent real-world events from different sources produce the same key.
- **Database-level uniqueness:** the ledger database, rather than application code alone, prevents duplicate inserts.
- **Corrections are additive:** corrections are stored as new linked movements rather than editing or deleting history.

## Limitations

This is a local demonstration rather than a production deployment.

A production version would add:

- Real warehouse API integrations.
- Authentication and secret management.
- Retry logic and a dead-letter queue.
- Structured logging and monitoring.
- Source event versioning.
- Pagination for large source feeds.
- A scheduler or queue-based trigger.
- A production database such as PostgreSQL.
- More robust handling of out-of-order events and late-arriving corrections.

## Conclusion

The project demonstrates a multi-step inventory synchronisation agent that:

- Reads from two independent warehouse systems.
- Uses watermarks for incremental processing.
- Detects duplicate real-world movements across sources.
- Records corrections without double-counting them.
- Maintains a central ledger safely.
- Produces the same correct result when run repeatedly.
