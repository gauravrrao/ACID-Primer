# Database Transactions, ACID, Isolation Levels & Locking

A practical and in-depth guide to **ACID properties, transaction isolation, pessimistic locking, optimistic locking, deadlocks, and Redis-based distributed locking**.

These concepts are fundamental when building systems such as:

- Booking systems
- Payment systems
- E-commerce
- Inventory management
- Banking systems
- Seat reservation systems
- Order processing
- Wallet/balance systems
- Distributed backend services

---

# Table of Contents

1. [What is a Transaction?](#1-what-is-a-transaction)
2. [ACID Properties](#2-acid-properties)
3. [Atomicity](#21-atomicity)
4. [Consistency](#22-consistency)
5. [Isolation](#23-isolation)
6. [Durability](#24-durability)
7. [Isolation Levels](#3-isolation-levels)
8. [Read Uncommitted](#31-read-uncommitted)
9. [Read Committed](#32-read-committed)
10. [Repeatable Read](#33-repeatable-read)
11. [Serializable](#34-serializable)
12. [Isolation Level Comparison](#35-isolation-level-comparison)
13. [Pessimistic Locking](#4-pessimistic-locking)
14. [Optimistic Locking](#5-optimistic-locking)
15. [Pessimistic vs Optimistic Locking](#6-pessimistic-vs-optimistic-locking)
16. [Deadlocks](#7-deadlocks)
17. [How Databases Handle Deadlocks](#8-how-databases-handle-deadlocks)
18. [How to Prevent Deadlocks](#9-how-to-prevent-deadlocks)
19. [Redis and Database Locking](#10-redis-and-database-locking)
20. [Redis Distributed Lock](#11-redis-distributed-lock)
21. [Redis vs Database Locks](#12-redis-vs-database-locks)
22. [BookMyShow Example](#13-bookmyshow-example)
23. [E-commerce Inventory Example](#14-e-commerce-inventory-example)
24. [Interview Questions](#15-interview-questions)
25. [Interview Summary](#16-interview-summary)

---

# 1. What is a Transaction?

A **transaction** is a group of database operations that should be treated as one logical unit of work.

For example, consider transferring ₹1,000 from Account A to Account B.

We need to perform:

```text
1. Deduct ₹1,000 from Account A
2. Add ₹1,000 to Account B
```

These two operations should not partially succeed.

Bad scenario:

```text
Account A: ₹10,000
Account B: ₹5,000

Deduct ₹1,000 from A
        ↓
A = ₹9,000

Application crashes

Add ₹1,000 to B never happens
        ↓

B = ₹5,000
```

₹1,000 has effectively disappeared.

A transaction prevents this:

```text
BEGIN TRANSACTION

Deduct ₹1,000 from A
Add ₹1,000 to B

COMMIT
```

If something fails:

```text
ROLLBACK
```

The database returns to the previous valid state.

---

# 2. ACID Properties

ACID stands for:

```text
A → Atomicity
C → Consistency
I → Isolation
D → Durability
```

These properties describe how reliable database transactions should behave.

---

# 2.1 Atomicity

Atomicity means:

> A transaction is treated as one indivisible unit. Either all operations succeed, or none of them do.

Example:

```text
Transfer ₹1,000

A → -₹1,000
B → +₹1,000
```

Both operations must succeed.

If one fails:

```text
A → rollback
B → rollback
```

### Example

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 1000
WHERE id = 1;

UPDATE accounts
SET balance = balance + 1000
WHERE id = 2;

COMMIT;
```

If the second operation fails:

```sql
ROLLBACK;
```

The first operation is also undone.

### Interview answer

> Atomicity guarantees that a transaction either completes completely or has no effect. If any operation inside the transaction fails, the database rolls back the transaction.

---

# 2.2 Consistency

Consistency means:

> A transaction must move the database from one valid state to another valid state while respecting all defined rules and constraints.

For example:

```text
Account balance >= 0
```

If the database has a constraint:

```sql
CHECK (balance >= 0)
```

A transaction attempting:

```text
Balance = ₹500

Withdraw ₹1,000

Balance = -₹500
```

should fail.

Other examples of consistency constraints:

```text
PRIMARY KEY
FOREIGN KEY
UNIQUE
CHECK
NOT NULL
```

### Example

Suppose:

```text
users
-----
id = 10
```

and:

```text
orders
------
user_id = 10
```

If the user does not exist:

```text
user_id = 999
```

a foreign-key constraint can prevent the invalid state.

### Important distinction

Consistency is not simply:

> "The database always has correct data."

It means the transaction respects the **rules/invariants defined by the database/application**.

---

# 2.3 Isolation

Isolation means:

> Concurrent transactions should not incorrectly interfere with each other.

Imagine two transactions running simultaneously.

```text
Transaction A
Transaction B
```

Both access the same data.

Without proper isolation, one transaction could observe another transaction's intermediate state.

Example:

```text
Initial balance = ₹10,000

Transaction A:
Deduct ₹2,000

Transaction B:
Reads balance
```

If B reads A's uncommitted change:

```text
B sees ₹8,000
```

But A could still rollback.

Then B observed data that never actually existed in the committed database state.

This is called a:

# Dirty Read

Isolation levels control these concurrency problems.

---

# 2.4 Durability

Durability means:

> Once a transaction has been successfully committed, its changes should survive crashes or restarts.

Example:

```sql
COMMIT;
```

After this:

```text
Application crashes
Database server restarts
Machine loses power
```

The committed transaction should still exist.

Databases achieve durability using mechanisms such as:

- Write-ahead logging (WAL)
- Transaction logs
- Persistent storage
- Recovery mechanisms
- Checkpoints

Example:

```text
Transaction
     ↓
Write to transaction/WAL log
     ↓
COMMIT
     ↓
Data persisted
```

---

# 3. Isolation Levels

Isolation levels define:

> How much one transaction is isolated from other concurrent transactions.

The commonly discussed SQL isolation levels are:

```text
1. Read Uncommitted
2. Read Committed
3. Repeatable Read
4. Serializable
```

Higher isolation generally provides stronger consistency but can reduce concurrency.

---

# 3.1 Read Uncommitted

This is the weakest isolation level.

A transaction can potentially read data modified by another transaction **before it commits**.

Example:

Initial balance:

```text
₹10,000
```

Transaction A:

```text
UPDATE accounts
SET balance = 8000;
```

A has not committed.

Transaction B reads:

```text
₹8,000
```

Then A rolls back:

```text
ROLLBACK
```

Actual balance becomes:

```text
₹10,000
```

But B already saw:

```text
₹8,000
```

This is a:

## Dirty Read

### Problems

Read Uncommitted can allow:

- Dirty reads
- Unstable/inconsistent observations

### Practical usage

It is rarely appropriate for correctness-critical application logic.

---

# 3.2 Read Committed

Read Committed guarantees:

> A transaction only reads data that has already been committed.

Example:

```text
Initial balance = ₹10,000
```

Transaction A:

```text
UPDATE balance = 8000;
```

A has not committed.

Transaction B:

```text
SELECT balance;
```

B sees:

```text
₹10,000
```

A then commits:

```text
COMMIT;
```

If B performs another read later, it may see:

```text
₹8,000
```

Therefore, the same transaction can potentially get different results from two reads.

This is called:

# Non-repeatable Read

Example:

```text
Transaction B

Read → ₹10,000

Transaction A updates and commits

Read again → ₹8,000
```

The row changed between B's reads.

---

# 3.3 Repeatable Read

Repeatable Read provides stronger guarantees.

The idea is:

> If a transaction reads a row, repeated reads of that same row should return a consistent result within that transaction.

Example:

```text
Transaction B

SELECT balance
→ ₹10,000
```

Another transaction changes the balance:

```text
₹10,000 → ₹8,000
```

Depending on the database's implementation, B can continue seeing its transaction-consistent version.

This prevents the classic:

```text
Read
Update by another transaction
Read again
```

from producing a different value for the same previously read row.

---

# 3.4 Serializable

Serializable is generally considered the strongest standard isolation level.

The goal is:

> Concurrent transactions should produce a result equivalent to some serial execution of those transactions.

Imagine:

```text
Transaction A
Transaction B
```

Instead of allowing unsafe concurrency:

```text
A
B
A
B
```

the database ensures the outcome is equivalent to some valid ordering such as:

```text
A
A
B
B
```

or:

```text
B
B
A
A
```

This provides very strong correctness but can significantly reduce concurrency.

It may result in:

- More blocking
- More retries
- More serialization failures
- Lower throughput

Therefore, Serializable should not automatically be chosen for every operation.

---

# 3.5 Isolation Level Comparison

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read | Concurrency |
|---|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible | Highest |
| Read Committed | Prevented | Possible | Possible | High |
| Repeatable Read | Prevented | Prevented | DB-dependent | Medium |
| Serializable | Prevented | Prevented | Prevented | Lowest |

> Exact behavior can vary by database engine and implementation, particularly around Repeatable Read and phantom reads.

---

# 4. Pessimistic Locking

Pessimistic locking follows the assumption:

> "A conflict is likely, so lock the resource before modifying it."

The database prevents other transactions from modifying the locked resource until the lock is released.

A common SQL example is:

```sql
SELECT *
FROM seats
WHERE id = 101
FOR UPDATE;
```

The database obtains a lock on the selected row.

Then:

```text
Transaction A
     ↓
Lock seat 101
     ↓
Check availability
     ↓
Book seat
     ↓
COMMIT
     ↓
Release lock
```

While A holds the lock:

```text
Transaction B
     ↓
tries to modify seat 101
     ↓
waits
```

---

# 4.1 Example: Seat Booking

Suppose:

```text
seat_id = 101
status = AVAILABLE
```

Two users attempt to book it simultaneously.

Without locking:

```text
User A → READ → AVAILABLE
User B → READ → AVAILABLE

User A → BOOK
User B → BOOK
```

Both users may believe they successfully booked the same seat.

With pessimistic locking:

```sql
BEGIN;

SELECT *
FROM show_seats
WHERE id = 101
FOR UPDATE;
```

Now:

```text
Transaction A
    ↓
locks seat 101

Transaction B
    ↓
tries to lock seat 101
    ↓
WAIT
```

A books the seat:

```sql
UPDATE show_seats
SET status = 'BOOKED'
WHERE id = 101;

COMMIT;
```

The lock is released.

B then obtains access and sees:

```text
status = BOOKED
```

and does not book it.

---

# 4.2 Advantages of Pessimistic Locking

- Strong protection against concurrent modifications
- Useful when conflicts are frequent
- Good for critical inventory/seat operations
- Database directly manages the lock
- Simple mental model

---

# 4.3 Problems with Pessimistic Locking

The biggest problem is:

# Waiting

If Transaction A holds a lock for a long time:

```text
A
│
├── LOCK
│
├── expensive operation
│
├── network call
│
└── COMMIT
```

Transactions B, C and D may wait.

This can cause:

```text
Lock contention
      ↓
More waiting
      ↓
Higher latency
      ↓
Connection pool exhaustion
      ↓
Lower throughput
```

It can also contribute to:

```text
Deadlocks
```

---

# 5. Optimistic Locking

Optimistic locking follows the opposite assumption:

> "Conflicts are relatively rare, so don't lock the resource. Detect conflicts when updating."

A common implementation uses a:

```text
version
```

column.

Example:

```text
product_id = 10
stock = 5
version = 7
```

Two requests read:

```text
stock = 5
version = 7
```

Both work with version 7.

Transaction A updates:

```sql
UPDATE products
SET stock = 4,
    version = 8
WHERE id = 10
AND version = 7;
```

One row is updated.

Now:

```text
version = 8
```

Transaction B tries:

```sql
UPDATE products
SET stock = 4,
    version = 8
WHERE id = 10
AND version = 7;
```

No row matches because:

```text
Current version = 8
Expected version = 7
```

Therefore:

```text
UPDATE affected 0 rows
```

The application knows that another transaction modified the record.

It can:

```text
Retry
OR
Return conflict
```

---

# 5.1 Optimistic Locking Example

Table:

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    stock INT NOT NULL,
    version INT NOT NULL
);
```

Initial state:

```text
id | stock | version
---|-------|--------
10 |   5   |   7
```

Request A:

```sql
UPDATE products
SET stock = stock - 1,
    version = version + 1
WHERE id = 10
AND version = 7
AND stock > 0;
```

Result:

```text
1 row updated
```

Request B:

```sql
UPDATE products
SET stock = stock - 1,
    version = version + 1
WHERE id = 10
AND version = 7
AND stock > 0;
```

Result:

```text
0 rows updated
```

Therefore B knows that its version is stale.

---

# 5.2 Advantages of Optimistic Locking

- Doesn't hold database locks while application logic executes
- Better concurrency
- Good when conflicts are relatively rare
- Reduces lock waiting
- Useful for APIs where requests may take longer

---

# 5.3 Problems with Optimistic Locking

If conflicts happen frequently:

```text
Request
 ↓
Read
 ↓
Process
 ↓
Conflict
 ↓
Retry
 ↓
Conflict
 ↓
Retry
```

The system may waste work.

Therefore optimistic locking is particularly useful when:

```text
Read frequency >> Write conflict frequency
```

---

# 6. Pessimistic vs Optimistic Locking

| Feature | Pessimistic | Optimistic |
|---|---|---|
| Assumption | Conflicts likely | Conflicts unlikely |
| Lock acquired | Yes | Usually no |
| Conflict handling | Prevent conflict | Detect conflict |
| Waiting | Possible | Usually avoided |
| Retry | Less common | Common |
| Concurrency | Lower | Higher |
| Deadlock risk | Possible | Much lower |
| Good for | High-conflict resources | Low-conflict updates |

### Simple interview explanation

> Pessimistic locking prevents conflicts by locking the resource before modifying it. Optimistic locking allows concurrent work and detects a conflict at update time, commonly using a version column.

---

# 7. Deadlocks

A deadlock occurs when transactions are waiting for each other indefinitely.

Consider:

```text
Transaction A
locks Row 1

Transaction B
locks Row 2
```

Then:

```text
A wants Row 2
B wants Row 1
```

We now have:

```text
A → waiting for B

B → waiting for A
```

Neither can continue.

This is a:

# Deadlock

---

# 7.1 Simple Example

Transaction A:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;
```

A locks:

```text
Account 1
```

Transaction B:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 50
WHERE id = 2;
```

B locks:

```text
Account 2
```

Now A tries:

```sql
UPDATE accounts
SET balance = balance + 100
WHERE id = 2;
```

A waits for B.

B tries:

```sql
UPDATE accounts
SET balance = balance + 50
WHERE id = 1;
```

B waits for A.

Graph:

```text
Transaction A
      ↓
   waits for
      ↓
Transaction B
      ↓
   waits for
      ↓
Transaction A
```

Deadlock.

---

# 7.2 Deadlock vs Lock Contention

These are not the same.

## Lock Contention

One transaction waits for another:

```text
A → holds lock

B → waits for A
```

Eventually:

```text
A finishes
B continues
```

This is contention.

## Deadlock

Transactions wait for each other:

```text
A → waits for B
B → waits for A
```

Neither can continue without intervention.

---

# 8. How Databases Handle Deadlocks

Modern relational databases can detect deadlocks.

The database builds/analyzes a dependency relationship between transactions.

If it detects:

```text
A → B → A
```

it chooses a transaction as the victim and aborts/rolls it back.

For example:

```text
Transaction A → aborted
Transaction B → continues
```

The application may then retry Transaction A.

Therefore:

> Deadlock handling is not only a database responsibility. Applications should generally be prepared to retry transactions that fail because of transient concurrency errors.

---

# 9. How to Prevent Deadlocks

## 9.1 Always acquire locks in the same order

Bad:

```text
Transaction A:
Lock 1
Lock 2

Transaction B:
Lock 2
Lock 1
```

Better:

```text
Transaction A:
Lock 1
Lock 2

Transaction B:
Lock 1
Lock 2
```

Both transactions follow:

```text
1 → 2
```

This dramatically reduces the possibility of circular waiting.

---

## 9.2 Keep transactions short

Bad:

```text
BEGIN

LOCK

Call external API
Wait 2 seconds
Perform calculation
Send notification
UPDATE

COMMIT
```

Better:

```text
BEGIN

LOCK
UPDATE

COMMIT
```

Then perform non-critical external operations outside the transaction.

---

## 9.3 Don't hold database locks during network calls

Avoid:

```text
BEGIN
 ↓
LOCK ROW
 ↓
HTTP request
 ↓
wait
 ↓
database update
 ↓
COMMIT
```

Instead:

```text
Validate/process
 ↓
BEGIN
 ↓
LOCK
 ↓
UPDATE
 ↓
COMMIT
```

Keep the critical section as small as possible.

---

## 9.4 Update only the required rows

Large locking scope increases contention.

Prefer:

```sql
UPDATE products
SET stock = stock - 1
WHERE id = 10
AND stock > 0;
```

instead of unnecessarily locking a large set of records.

---

## 9.5 Use retry with backoff

If the database reports a transient deadlock/serialization failure:

```text
Attempt 1
    ↓
Deadlock
    ↓
wait 50ms

Attempt 2
    ↓
Deadlock
    ↓
wait 100ms

Attempt 3
    ↓
Success
```

Use a bounded retry count.

Example:

```text
maxRetries = 3
```

with exponential backoff and jitter.

---

# 10. Redis and Database Locking

Redis is often introduced when applications have high concurrency.

But an important distinction is:

> Redis does not "fix" a database deadlock automatically.

Instead, Redis can sometimes be used to **coordinate access before requests reach the database**, reducing database lock contention.

For example:

```text
Without Redis

100 requests
     ↓
Database
     ↓
Many transactions compete for same row
     ↓
Lock contention
     ↓
Potential deadlocks
```

With an application-level Redis lock:

```text
100 requests
     ↓
Redis lock
     ↓
Only lock owner enters critical section
     ↓
Database
     ↓
Short transaction
```

This can reduce the number of concurrent operations competing for the same database resource.

---

# 10.1 Why Use Redis?

Redis is extremely fast because it primarily operates in memory.

A common distributed-lock pattern is:

```text
SET key value NX PX timeout
```

Conceptually:

```text
SET lock:seat:101 unique-token NX PX 10000
```

Meaning:

```text
NX
→ Create only if key does not already exist

PX 10000
→ Automatically expire after 10 seconds
```

If the command succeeds:

```text
You acquired the lock
```

If it fails:

```text
Someone else currently owns the lock
```

---

# 11. Redis Distributed Lock

Imagine:

```text
User A → Server 1
User B → Server 2
User C → Server 3
```

All three servers want to process:

```text
Seat 101
```

Redis provides a shared coordination point:

```text
             Redis
               |
       lock:seat:101
          /    |    \
      Server1 Server2 Server3
```

Server 1 successfully acquires:

```text
lock:seat:101
```

Server 2:

```text
SET ... NX
```

fails.

Server 3:

```text
SET ... NX
```

fails.

Only Server 1 proceeds.

---

# 11.1 Always Use a Unique Lock Token

Suppose:

```text
Server A
```

creates:

```text
lock:seat:101 = token-A
```

Later its lock expires.

Another server acquires:

```text
lock:seat:101 = token-B
```

Server A must not accidentally delete Server B's lock.

Therefore, releasing a distributed lock should verify ownership.

Conceptually:

```text
IF lock value == my-token
    DELETE lock
```

This check-and-delete should be atomic, typically implemented using a small Lua script or an equivalent atomic server-side mechanism.

---

# 11.2 Redis Lock TTL

A distributed lock should generally have an expiration.

Example:

```text
lock acquired
TTL = 10 seconds
```

Why?

Imagine:

```text
Server A
   ↓
acquires lock
   ↓
Server crashes
```

Without expiration:

```text
Lock remains forever
```

Other requests can never proceed.

With TTL:

```text
Server crashes
      ↓
10 seconds
      ↓
Redis automatically removes lock
      ↓
Another request can proceed
```

---

# 11.3 Important Redis Lock Caveat

A Redis lock should **not automatically be treated as a replacement for the database's transactional guarantees**.

For correctness-critical data:

```text
Redis lock
+
Database transaction
+
Database constraints
```

may be used together.

The database should remain the authoritative source of truth for persistent state.

---

# 12. Redis vs Database Locks

| Feature | Database Lock | Redis Lock |
|---|---|---|
| Location | Database | Redis |
| Purpose | Protect database resources | Coordinate distributed application workers |
| Transaction aware | Yes | No |
| Automatic transaction rollback | Yes | No |
| Speed | Relatively slower | Very fast |
| Persistence | Database controlled | Redis configuration dependent |
| Useful for | Row-level data consistency | Distributed coordination |
| Deadlock handling | Database mechanisms | TTL/lock design |
| Source of truth | Database | Usually not |

---

# 12.1 When Should You Use a Database Lock?

Use database locking when the operation directly depends on database consistency.

Example:

```text
Read seat
Check availability
Update seat
Create booking
```

A database transaction can provide the required atomicity and consistency.

---

# 12.2 When Can Redis Help?

Redis can help when:

```text
Many application instances
        ↓
compete for the same logical resource
```

For example:

```text
10 application servers
        ↓
same inventory item
```

Redis can reduce the number of requests entering the database critical section simultaneously.

---

# 13. BookMyShow Example

Consider:

```text
Movie
   ↓
Show
   ↓
ShowSeat
```

Suppose:

```text
ShowSeat 101
status = AVAILABLE
```

Two users:

```text
User A
User B
```

try to book the same seat.

---

## Approach 1: Database Pessimistic Lock

```sql
BEGIN;

SELECT *
FROM show_seats
WHERE id = 101
FOR UPDATE;
```

Suppose A gets the lock.

```text
A → LOCK 101

B → WAIT
```

A:

```sql
UPDATE show_seats
SET status = 'BOOKED'
WHERE id = 101;

COMMIT;
```

B then checks:

```text
status = BOOKED
```

and rejects the booking.

---

# 13.1 Adding Redis

For high traffic:

```text
User A
   ↓
Application Server
   ↓
Redis lock:show:seat:101
   ↓
Acquire
   ↓
Database transaction
   ↓
UPDATE
   ↓
COMMIT
   ↓
Release Redis lock
```

Another request:

```text
User B
   ↓
Application Server
   ↓
Redis
   ↓
Lock already exists
   ↓
Reject / retry
```

This can reduce the number of simultaneous database transactions competing for the same seat.

---

# 13.2 But There Is a Better Booking Model

A real booking flow often separates:

```text
AVAILABLE
    ↓
HELD
    ↓
BOOKED
```

For example:

```text
AVAILABLE
    ↓
temporary reservation
    ↓
HELD
    ↓
payment
    ↓
BOOKED
```

If payment fails or the timer expires:

```text
HELD
  ↓
timeout
  ↓
AVAILABLE
```

This is different from holding a database row lock for the entire payment process.

You generally do **not** want:

```text
DB transaction
    ↓
lock seat
    ↓
call payment gateway
    ↓
wait 5 seconds
    ↓
payment
    ↓
commit
```

because the database lock is being held while waiting on an external service.

Instead, the reservation itself can have an expiration:

```text
seat
-----
status = HELD
hold_expires_at = timestamp
```

Then payment can happen outside the long database lock.

---

# 14. E-commerce Inventory Example

Suppose:

```text
Product
-------
id = 10
stock = 1
```

Two users purchase simultaneously.

---

## Pessimistic Approach

```sql
BEGIN;

SELECT stock
FROM products
WHERE id = 10
FOR UPDATE;
```

Then:

```text
stock = 1
```

Decrease:

```sql
UPDATE products
SET stock = stock - 1
WHERE id = 10;

COMMIT;
```

Second transaction waits and then discovers:

```text
stock = 0
```

---

# 14.1 Optimistic Approach

Use:

```text
version
```

or perform an atomic conditional update:

```sql
UPDATE products
SET stock = stock - 1
WHERE id = 10
AND stock > 0;
```

Then check:

```text
affectedRows
```

If:

```text
affectedRows = 1
```

the purchase reserved one unit.

If:

```text
affectedRows = 0
```

there was no stock available.

This can be very effective because the database performs the condition and update atomically.

---

# 15. Relationship Between ACID, Isolation and Locking

These concepts are related but they are **not the same thing**.

Think of them as layers:

```text
                 TRANSACTION
                     |
                    ACID
                     |
          +----------+----------+
          |          |          |
      Atomicity  Consistency  Durability
                     |
                  Isolation
                     |
              Isolation Levels
                     |
        +------------+------------+
        |                         |
  Read Committed          Serializable
        |
        |
   Concurrency Control
        |
   +----+----+
   |         |
Pessimistic Optimistic
 Locking     Locking
   |
   |
Lock Contention
   |
Deadlocks
   |
   |
Redis can sometimes reduce
application-level contention
```

---

# 16. Important Interview Distinctions

## ACID vs Isolation Level

ACID is a set of transaction properties.

Isolation level is specifically about:

```text
How concurrent transactions interact.
```

---

## Isolation vs Locking

Isolation is the **transactional guarantee**.

Locking is one of the **mechanisms used to control concurrent access**.

Modern databases can also use techniques such as:

```text
MVCC
Snapshots
Versioning
Locks
Serialization
```

depending on the engine and isolation level.

---

## Pessimistic vs Optimistic

Pessimistic:

```text
Prevent conflict
```

Optimistic:

```text
Detect conflict
```

---

## Database Lock vs Redis Lock

Database lock:

```text
Protect database state
```

Redis lock:

```text
Coordinate distributed application workers
```

Redis should not be considered a universal replacement for database transactions.

---

# 17. Interview Questions

## Q1. What is ACID?

**Answer:**

> ACID stands for Atomicity, Consistency, Isolation and Durability. Atomicity means all operations in a transaction succeed or none do. Consistency means transactions preserve defined data constraints and invariants. Isolation controls how concurrent transactions interact. Durability ensures committed changes survive failures.

---

## Q2. What is a dirty read?

> A dirty read occurs when a transaction reads data modified by another transaction before that modification has been committed.

---

## Q3. What is a non-repeatable read?

> A non-repeatable read occurs when a transaction reads the same row twice and gets different committed values because another transaction modified and committed that row between the reads.

---

## Q4. What is a phantom read?

> A phantom read occurs when repeating a query that matches a range of rows produces a different set of matching rows because another transaction inserted, deleted, or otherwise changed rows affecting that range.

---

## Q5. What is pessimistic locking?

> Pessimistic locking assumes conflicts are likely and locks the resource before modifying it. For example, `SELECT ... FOR UPDATE` can lock selected rows inside a transaction.

---

## Q6. What is optimistic locking?

> Optimistic locking assumes conflicts are relatively rare. Instead of holding a lock, the application uses something like a version number and updates only if the version it read is still current.

---

## Q7. How does optimistic locking detect a conflict?

Example:

```sql
UPDATE users
SET name = 'Gaurav',
    version = version + 1
WHERE id = 10
AND version = 5;
```

If:

```text
affectedRows = 1
```

the update succeeded.

If:

```text
affectedRows = 0
```

another transaction has already changed the row.

---

## Q8. What is a deadlock?

> A deadlock occurs when two or more transactions hold resources while waiting for resources held by each other, creating a circular dependency.

---

## Q9. How can you prevent deadlocks?

Main strategies:

```text
1. Acquire locks in a consistent order
2. Keep transactions short
3. Avoid network calls inside transactions
4. Lock only required resources
5. Use appropriate indexes/query plans
6. Detect and retry transient failures
7. Keep retry counts bounded
```

---

## Q10. Why does Redis help with database contention?

> Redis can act as a fast distributed coordination layer. Multiple application servers can attempt to acquire a Redis lock before entering a database critical section. This can reduce the number of concurrent operations competing for the same database resource, although Redis does not replace database transactions or eliminate all deadlock scenarios.

---

## Q11. Does Redis eliminate deadlocks?

**No.**

Redis can reduce certain forms of contention, but it does not magically eliminate all deadlocks.

You can still create bad designs such as:

```text
Redis lock A
    ↓
Database lock B
```

while another request does:

```text
Database lock B
    ↓
Redis lock A
```

Now you can still create circular waiting.

The real solution is:

```text
Consistent lock ordering
+
Short critical sections
+
Correct transaction design
+
Timeouts
+
Retry handling
```

---

# 18. Practical Decision Guide

When designing a backend system, think about the workload.

### High conflict

Example:

```text
100 users
     ↓
same seat
```

Potential approaches:

```text
Database atomic update
+
Short transaction
+
Possibly application-level distributed coordination
```

---

### Low conflict

Example:

```text
User profile
```

Optimistic locking can be appropriate:

```text
version = 10

UPDATE ... WHERE version = 10
```

---

### Financial / critical database state

Prefer the database as the source of truth:

```text
Database transaction
+
Constraints
+
Appropriate isolation
+
Idempotency
+
Careful locking
```

Do not rely solely on Redis for correctness.

---

# 19. One Complete Mental Model

When an interviewer gives you a concurrency problem, think in this order:

```text
                CONCURRENT REQUESTS
                        |
                        ↓
              What can conflict?
                        |
                        ↓
             What data must be
                consistent?
                        |
                        ↓
              Choose transaction
                        |
                        ↓
              Choose isolation
                        |
             +----------+----------+
             |                     |
             ↓                     ↓
      High conflict          Low conflict
             |                     |
             ↓                     ↓
      Pessimistic             Optimistic
        locking                 locking
             |                     |
             ↓                     ↓
       Lock contention       Version conflict
             |
             ↓
          Deadlock?
             |
             ↓
     Consistent lock order
     Short transactions
     Retry transient errors
             |
             ↓
       Very high distributed
          contention?
             |
             ↓
        Consider Redis
        coordination
             |
             ↓
       Database remains
       source of truth
```

---

# 20. Final Interview Cheat Sheet

```text
ACID
│
├── Atomicity
│   └── All or nothing
│
├── Consistency
│   └── Valid state → valid state
│
├── Isolation
│   └── Concurrent transactions don't incorrectly interfere
│
└── Durability
    └── Committed data survives failure
```

## Isolation Levels

```text
Read Uncommitted
    ↓
Weakest
    ↓
Read Committed
    ↓
Repeatable Read
    ↓
Serializable
    ↓
Strongest standard level
```

## Locking

```text
Pessimistic
    ↓
Lock first
    ↓
Prevent conflict
```

```text
Optimistic
    ↓
Don't lock during normal work
    ↓
Detect conflict
    ↓
Retry / reject
```

## Deadlock

```text
A holds X
A waits for Y

B holds Y
B waits for X

        ↓

      DEADLOCK
```

Prevent with:

```text
Consistent lock ordering
+
Short transactions
+
Minimal lock scope
+
No unnecessary external calls
+
Retry transient failures
```

## Redis

```text
Application Servers
        ↓
      Redis
        ↓
Distributed coordination
        ↓
Short database critical section
        ↓
    Database
```

Remember:

> **Redis can coordinate access; the database should enforce persistent data correctness.**

---

# One-Line Interview Definitions

| Concept | One-line answer |
|---|---|
| Transaction | A group of database operations treated as one logical unit |
| Atomicity | All operations succeed or none do |
| Consistency | Transaction preserves defined constraints/invariants |
| Isolation | Concurrent transactions are prevented from causing incorrect interference |
| Durability | Committed changes survive failures |
| Dirty Read | Reading another transaction's uncommitted data |
| Non-repeatable Read | Same row returns a different value within a transaction |
| Phantom Read | Repeating a range query returns a different set of rows |
| Pessimistic Lock | Lock the resource assuming a conflict is likely |
| Optimistic Lock | Detect conflicts using a version/check rather than holding a lock |
| Deadlock | Transactions wait for each other in a cycle |
| Redis Lock | Distributed coordination mechanism implemented using Redis |
| Database Lock | Mechanism used by the database to control concurrent access to database resources |
| Lock Contention | Multiple transactions competing for the same lock |
| Serializable | Isolation level intended to make concurrent execution equivalent to some serial execution |

---

# The Most Important Mental Model

If you remember only one thing for a backend interview:

```text
ACID
 ↓
Isolation
 ↓
Concurrency
 ↓
Locking
 ↓
Pessimistic / Optimistic
 ↓
Contention
 ↓
Deadlocks
 ↓
Consistent lock ordering + short transactions + retry
 ↓
Redis can coordinate distributed requests
 ↓
Database remains the source of truth
```

This chain connects almost every database concurrency question asked in backend interviews.