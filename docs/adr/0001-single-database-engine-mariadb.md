# MariaDB as the single database engine

The development machine (macOS 12) has no Docker and no PostgreSQL. MariaDB 10.4 is already present through XAMPP and already serves another project on the same machine. We use MariaDB for development and for production alike, so the dialect and the query behaviour stay identical on both sides. The consequences: the MySQL/MariaDB dialect through the `mysql2` driver (see ADR-0005), and the PostgreSQL-only features (JSONB, partial index, array) are out of reach. A later change of engine means a full migration, so we took this decision knowingly, even though PostgreSQL is the richer database in general.

## Rejected alternatives

- **Local PostgreSQL**: needs a source build on this machine, because there is no Docker and Homebrew has no bottle for macOS 12. The setup cost is out of proportion for phase 1.
- **SQLite in development, PostgreSQL in production**: two engines mean the dialect, the column types, and the constraint behaviour differ between what we test and what Participants use.
