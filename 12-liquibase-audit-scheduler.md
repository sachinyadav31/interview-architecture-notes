# 12: Schema migrations (Liquibase), audit trail (Javers), task scheduler

## A) Liquibase: migrations in every tenant DB
- Every service has `src/main/resources/liquibase/changelog-master.xml` including numbered changelogs (`013-dml-seed-module-function-acls.xml`, …).
- DDL (create table/column/index) **and** DML seed data (ACLs, roles, reference data) are both changelogs.
- **Per-tenant execution**: `DynamicDataSourceBasedMultiTenantSpringLiquibase` loops over all tenant datasources and runs `SpringLiquibase` on each, passing `tenant_qualifier` as a **changelog parameter** (so a changelog can say `${tenant_qualifier}_esuser`). Failures are per tenant: logged, then it continues.
- `contexts` (`LIQUIBASE_CONTEXTS`) switch between dev/test data and prod.
- Tracking tables: `DATABASECHANGELOG` (id + author + file + MD5 checksum per changeset) and `DATABASECHANGELOGLOCK` (stops two instances migrating at once).

Common problems I fixed ("liquibase fixed" commits in 2026):
- **Checksum mismatch**: someone edited an already-applied changeset → startup fails. Rule: never edit an applied changeset; add a new one (or as a last resort `validCheckSum` / `runOnChange` for views and procs).
- **Stuck lock** after a crash: `DATABASECHANGELOGLOCK.locked = 1`. Clear it once you're sure no migration is running.
- Wrong DB name / missing table on a new service.

## B) Javers: audit trail of entity changes
- `application-essentials` integrates **Javers**: on save, `javers.commit(author, entity)` stores a **snapshot** and a diff.
- Read side: `javers.findChanges(QueryBuilder.byInstanceId(id, Entity.class))` and `findSnapshots`, which give "who changed what field, from → to, when", shown in the UI's history tab (e.g. PO price changes).
- Audit fields on every entity (`createdBy`, `createdDate`, `lastModifiedBy`, `lastModifiedDate`) come from `EntityAuditFieldsUpdateListener` (`@EntityListeners`), filled from the user context.
- I fixed a Javers fetch bug (2024) where `simpleChange`-type deserialisation threw a type error.

## C) Task scheduler service (time-based events)
- Purpose: "do X at time T" for other services (RFQ closing time, contract renewal reminders, expiry).
- Submit: a producer sends `task-scheduler-submit-topic` with `ScheduleTaskEvent` {cron/expiry date, metadata (entity ID, type), response topic}.
- `TaskSchedulerSubmitEventListener` (`@KafkaListener`) stores `ScheduledTask` + `ScheduledTaskMetadata` rows.
- `ScheduledTaskExecuterService` has `@Scheduled(fixedDelayString = ...)`: it **polls the DB** for due tasks and publishes `schedule-executed-topic`; the requesting service reacts (e.g. eSourcing closes the RFQ). Expired tasks get marked.

### Weak spots and improvements
- With **several instances**, `@Scheduled` runs on every instance, so the same task fires twice. Fix: **ShedLock** (`@SchedulerLock` with a DB lock table) or `SELECT … FOR UPDATE SKIP LOCKED` to claim rows, or Quartz in clustered mode.
- Polling interval = precision. Fine for minutes, not for seconds.
- Liquibase on app startup slows boot and races across replicas (the lock helps). Better: run migrations as a separate pipeline step/job before the rollout.

## Interview Q&A

**Q: Liquibase vs Flyway?**
A: Both version DB changes. Flyway uses plain SQL files named V1__x.sql and is simple. Liquibase uses XML/YAML/SQL changelogs, has DB-agnostic change types, rollback support, contexts/labels and preconditions. We used Liquibase because of contexts and the per-tenant parameter substitution.

**Q: How do you change a column safely with zero downtime?**
A: Expand–contract: add the new column (nullable), deploy code that writes both, backfill, switch reads, then remove the old column in a later release. Never rename or drop in one step while old pods still run.

**Q: How do you audit changes?**
A: Two levels: audit columns (created/modified by/at) through JPA entity listeners, and full field-level history with Javers snapshots and diffs, queried by instance ID for the UI history view.

**Q: `@Scheduled` with 3 replicas: what goes wrong, and how do you fix it?**
A: It runs three times. Use ShedLock (a DB/Redis lock per job), or claim work rows with `SELECT … FOR UPDATE SKIP LOCKED`, or Quartz clustered with its JDBC job store.

**Q: fixedRate vs fixedDelay vs cron?**
A: `fixedRate`: start every N ms regardless (runs can overlap if a run is slow and the pool is >1). `fixedDelay`: N ms after the previous run **ends**. `cron`: calendar based. We used fixedDelay for polling so runs never overlap.
