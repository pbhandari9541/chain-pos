# chain-pos

Offline-first chain point-of-sale: register, kitchen display, multi-store sync, head-office reports.

## Status

Planned. All data is synthetic (made-up stores, menus and customers). No internet, cloud services, payment providers or real customer data.

## Parts

- **Register (one per store):** Keyboard-driven order entry, modifiers, split bills, discounts, tax, tips, voids/refunds, cash-up and PDF receipts. Every sale is committed to a local append-only event log before the UI shows it as complete.
- **Kitchen display:** Live ticket queue per station, bump/recall, ticket timing.
- **Store sync:** Each store keeps selling without a connection. Events have unique idempotency keys and per-store sequence numbers. They are delivered at least once and applied exactly once through idempotency keys, sequence numbers and database transactions.
- **Fault-injecting proxy:** A local TCP/HTTP proxy that drops, delays, duplicates, reorders and cuts connections between stores and head office, and can kill a store process mid-sale.
- **Head office:** Central price and menu management, loyalty balances, and consolidated reports (sales by store/day/item, tax, cash-up variance) built from synced event logs.
- **Simulator:** A seeded load generator that drives 8 stores with one register each, using realistic sales, discounts and refunds. An injectable clock controls simulated time so test runs are deterministic.

## The hard part

I chose this project because I run a restaurant and have built ordering systems like B2K and ChefHub. I know how important accurate sales records are when the internet goes down or a register crashes. I want to test what happens if a register is killed after a payment is recorded but before the receipt prints. After restarting, the sale must appear exactly once. My biggest challenges will be crash recovery, duplicate transactions, offline synchronization, and keeping head-office totals equal to store totals to the cent.

## Data consistency rules

- **Confirmed sale:** A sale is confirmed only after its event has been committed to the local SQLite database before the UI displays success. Each order gets a client-generated order ID when it is started. After a crash and restart, the register reopens any committed sale it has no receipt for, so a cashier cannot accidentally enter it a second time. A sale killed before its commit was never confirmed, so the cashier re-enters it.
- **Payments:** Tenders are simulated (cash and a fake card). No real payment provider is used.
- **Durability:** SQLite uses WAL mode with `synchronous=FULL` to protect committed transactions, including against power loss within SQLite's documented durability guarantees.
- **Delivery:** Network delivery is at least once. Head office uses unique idempotency keys, per-store sequence numbers and atomic database transactions to ensure each event affects totals only once.
- **Ordering:** Events are applied in sequence per store. Missing sequence numbers pause processing for that store until the gap is recovered. Head office acknowledges the highest contiguous sequence number it has applied, and the store resends everything after that number.
- **Money:** All amounts are stored as integer cents. Tax is calculated per line item and rounded half-up to the cent. Discounts are applied before tax, and any leftover cent is allocated in a fixed order. The same code calculates totals in stores and at head office, so the totals should not differ, and the reconciliation tests check that.
- **Prices:** Each sale records the price-list version and actual prices used at checkout. Stores continue using their last downloaded price list while offline. New prices apply only after a store receives and activates the updated version.
- **Loyalty:** Offline redemptions are accepted against the store's last known balance, subject to a configured limit. Head office reconciles redemptions in a deterministic order, sorted by store ID and then per-store sequence number, because store clocks cannot be trusted. If multiple offline stores redeem the same points, a bounded negative balance is allowed and flagged for review rather than silently changing completed sales.
- **Reports:** Head-office reports are generated from synchronized events, using the same calculation rules as store reports.

## Success bar

- **100 random `kill -9` tests:** Kill the register at different points during a sale, including after the database commit but before the receipt prints. After restart, every confirmed sale exists exactly once, with no duplicates. These tests simulate process crashes, not power loss.
- **8-store simulation:** Run 8 stores with one register each over a simulated month. Randomly disconnect stores and inject network failures. After all events synchronize, head-office totals must equal the sum of store ledgers exactly, using integer cents.
- **Deterministic replay:** Replaying any store's event log from scratch produces identical report data (JSON/CSV). The same random seed and injected clock also produce the same simulated transactions. Generated PDFs are excluded from byte-for-byte comparisons because they may contain timestamps and unique IDs.
- **Idempotency:** Replaying or duplicating events must never create additional sales, refunds or loyalty transactions.
- **Recovery:** Missing events are detected using sequence numbers and recovered after reconnection.
- **End-to-end testing:** A Playwright suite covers every register flow (sale, split bill, discount, refund, cash-up, receipt) and the kitchen display.
- **Performance:** One register commits at least 200 sales per second on the test machine, and head office applies 100,000 synced events in under 60 seconds.

## Milestones / sessions

1. **Core ledger and register:** Money in integer cents, order model, tax, discounts, splits, refunds, cash-up, append-only event log, PDF receipts, unit tests.
2. **Durability:** Crash-safe writes using SQLite WAL with `synchronous=FULL`, recovery on restart, `kill -9` test harness.
3. **Sync and fault proxy:** At-least-once delivery, idempotent event processing, sequence-gap detection and recovery, fault-injecting proxy, convergence tests.
4. **Kitchen display and head office:** Tickets, central pricing, price-list versions, offline loyalty rules, consolidated reports.
5. **Scale and verification:** Seeded month-long simulation with an injectable clock, 8-store reconciliation runs, Playwright suite, performance targets (200 sales per second per register, 100,000 events applied in under 60 seconds).

## Stack

Python 3, Flask, SQLite (WAL, `synchronous=FULL`), plain HTML/JavaScript browser UI, pytest and Playwright. Everything runs locally on one machine.

## License

MIT
