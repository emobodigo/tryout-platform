# Drizzle ORM, not Prisma

The schema is plain TypeScript under `apps/api/src/db/schema/`. `drizzle-kit generate` writes SQL migration files, and those files are committed. The connection uses the `mysql2` driver (`drizzle-orm` 0.45, `drizzle-kit` 0.31, `mysql2` 3.24).

Reasons: Drizzle is pure JavaScript, so `npm install` never downloads a separate engine binary — this development machine has stalled more than once on binary downloads from GitHub releases. The migrations are SQL files that a human can read and edit, and there is no `generate client` step to repeat after every schema change.

Consequences to accept: there is no eager loading like the Prisma `include`, so every query that touches several tables writes its own joins, and a wrong join shape still passes the type check; a cross-table query therefore needs its own test. The data-inspection GUI is not as complete as Prisma Studio. A move to another ORM means rewriting the whole data-access layer.

## Rejected alternatives

- **Prisma**: an engine binary at install time, and a `generate client` step that must run again after every schema change.
- **TypeORM**: decorators and entities spread across the code, and a migration history that collides easily on a fast-moving project.
- **Kysely / raw SQL**: full control, but the schema stops being one source of truth for both the types and the migrations.
