# How to run the project

## Steps to start the project

### 1. Downloading the project

Clone the project with subodules using the following command:

```bash
git clone --recurse-submodules https://github.com/Lesnoi-Dmytro/sport-courts
```

### 2. Start the project

Start the project using the following command:

```bash
docker compose up
```

## 3. Creadentials

### Admin

- Email: `admin@example.com`

### Customers

- Email: `john.doe@example.com`
- Email: `jane.smith@example.com`

### Password

All seeded test users share the same password: `Password123!`

# Stack choices

## Database

PostgreSQL was used as a database bacause of it's mature ecosystem and large set of features. Important features for this project:
- gin_trgm_ops index for ILIKE search on name field for courts search.
- TimescaleDB extension can be added to speed up time-based queries

The workload is read-heavy, with no need to handle hundreds of writes per second, so PostgreSQL write throughput is not a concern.

MySQL was considered but dismissed. I ran into several bugs that have remained unfixed for years, and some are still open today. MySQL also lacks transactional DDL, so a failed migration can leave the schema half-applied, which PostgreSQL avoids.

MongoDB was dismissed because every entity has a well-defined schema, so the flexibility of a document store isn't needed. Relational data is also easier to model and query in SQL, with foreign keys enforcing integrity between entities.

## Backend

NestJS was chosen over Express because of it's predefined structure and huge set of built-in features. Express can be more flexible and pleasant to work with once it is configured well, but that setup takes time. For a small project with a limited timeframe NestJS is a better choice.

## Frontend

React was chosen over NextJS because this project doesn't need SSR. The content is highly dynamic, so SSR would add complexity without much benefit.

# How double booking is prevented under concurrency

The double booking for the same spot is ensured at the database level using locking mechanism. When `CourtBookingsService.create` creates a booking, it acquires a lock on the `courtId`. This forces other concurent transactions to wait until the lock is released(transaction is completed), preventing them from reading the existing bookings until new booking is created.

```js
const court = await courtsRepo.findOneOrFail({
  where: { id: data.courtId },
  lock: { mode: 'pessimistic_write' },
});
```

Alternative approaches(discarded):
- Serializable isolation level. It would work too, since Postgres fails one of the conflicting transactions, but then every booking needs a retry on serialization errors. It can also fail bookings that don't really conflict, and a busy court would keep redoing work
- Event-based approach. Processing events one-by-one in-order would prevent conflicts similar to rows locking. Would introduce too much complexity without benefits, especially if used in an environment that runs multiple instances of a server. Also, would be slower than locking

# Next steps

- Improve authentitcation logic by adding refresh token to http_only cookies, and moving access token to local storage and decreasing the lifetime
- Add a mechanism to manually trigger migrations/seeding instead of automatic execution
- Use PostgreSQL Pub/Sub to share events between connections from different servers(if used in production with multiple instances running at the same time)
- Save cancelled bookings instead of deleting them from the DB
- Support different court opening hours by days
- Add a filter by hourly price
