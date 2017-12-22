---
layout: post
title:  "Why you shouldn't use delayed_job_active_record with PostgreSQL"
date:   2017-12-23 01:00:00 +0900
categories: ruby delayed_job
---


First
----

The [`delayed_job`](https://github.com/collectiveidea/delayed_job) is job worker system for Ruby.
The [`delayed_job_active_record`](https://github.com/collectiveidea/delayed_job_active_record) is an adapter for delayed_job worker to connect RDBMS using [`ActiveRecord`](https://github.com/rails/rails/tree/master/activerecord).
The `PostgreSQL` is one of RDBMS supported by `delayed_job_active_record`.

In this article, architecture of `delayed_job` is out of scope. And, I don't compare any other similar system.


Problem 1. A Worker Takes All Jobs
----

The `delayed_job_active_record` allows to choose SQL strategy from two. The default strategy is `:optimized_sql`. It looks efficient. But it will occur problem.

- [Single worker picks up all jobs · Issue \#915 · collectiveidea/delayed\_job](https://github.com/collectiveidea/delayed_job/issues/915)
- [Reserving jobs hits Postgresq Update\.\.\(subquery \.\.limit 1\) bug · Issue \#143 · collectiveidea/delayed\_job\_active\_record](https://github.com/collectiveidea/delayed_job_active_record/issues/143)

A worker sometimes takes all jobs in queue at that time. Normally, a worker takes a job at a time. It will be delay job performing if a worker can take many jobs at a time.

### Cause

The `:optimized_sql` strategy build following SQL.

```sql
UPDATE "delayed_jobs"
   SET locked_at = '{Current Time}'
     , locked_by = '{This Worker Name}'
 WHERE id IN
   (
     SELECT "delayed_jobs"."id"
       FROM "delayed_jobs"
      WHERE
        (
          (
            run_at <= '{Current Time}' AND
            (
              locked_at IS NULL OR
              locked_at < '{Max Run Time Ago}'
            ) OR
            locked_by = '{This Worker Name}'
          ) AND
          failed_at IS NULL
        )
      ORDER BY priority ASC, run_at ASC
      LIMIT 1
        FOR UPDATE
   )
RETURNING *
```

The subquery will return a row using `LIMIT 1`. But, it sometimes return multiple rows. Why?

> The planner may choose to generate a plan that executes a nested loop over the LIMITing subquery, causing more UPDATEs than LIMIT
> [sql \- Postgres UPDATE \.\.\. LIMIT 1 \- Database Administrators Stack Exchange](https://dba.stackexchange.com/questions/69471/postgres-update-limit-1/69497#comment321713_69497)

The PostgreSQL will cause this problem.

__*So, you shouldn't use `:optimized_sql` if you use `delayed_job_active_recoed` with PostgreSQL.*__


Problem 2. Workers Dispatch Same Job
----

You can also choose an old fashion `:default_sql` strategy.

> It is usually slower but works better for certain people.
> [collectiveidea/delayed\_job\_active\_record: ActiveRecord backend integration for DelayedJob 3\.0\+](https://github.com/collectiveidea/delayed_job_active_record#problems-locking-jobs)

It is not truth about PostgreSQL. Some workers sometimes takes same job if it is locking. It means duplicate execution. Normally, a locked job will not execute by other worker.

### Cause

The `:default_sql` strategy build following SQL.

```sql
UPDATE "delayed_jobs"
   SET "locked_at" = '{Current Time}'
     , "locked_by" = '{This Worker Name}'
 WHERE "delayed_jobs"."id" IN
   (
     SELECT "delayed_jobs"."id"
       FROM "delayed_jobs"
      WHERE
        (
          (
            run_at <= '{Current Time}' AND
            (
              locked_at IS NULL OR
              locked_at < '{Max Run Time Ago}'
            ) OR
            locked_by = '{This Worker Name}'
          ) AND failed_at IS NULL
        ) AND "delayed_jobs"."id" = {Job ID}
      ORDER BY priority ASC, run_at ASC
   )
```

This SQL is not lock in sub query. So it allows update same record by concurrent sessions if other session set locked_at field before.

__*So, you shouldn't use `:default_sql` if you use `delayed_job_active_recoed` with PostgreSQL.*__


Solution?
----

If you want to keep using `delayed_job_active_record` with PostgreSQL, I show two solutions.

- Choose `:optimized_sql` (If you can accept delay)
- Write SQL yourself (If you are SQL expert)

I think you may resolve problem re-writing SQL using `WITH` clause.

Following SQL is a sample. But, I don't guarantee correctness. It just only "SAMPLE".

```sql
WITH cte AS (
  SELECT "delayed_jobs"."id"
    FROM "delayed_jobs"
   WHERE
     (
       (
         run_at <= '{Current Time}' AND
         (
           locked_at IS NULL OR
           locked_at < '{Max Run Time Ago}'
         ) OR
         locked_by = '{This Worker Name}'
       ) AND
       failed_at IS NULL
     )
   LIMIT 1
     FOR UPDATE
)
UPDATE "delayed_jobs" AS t
   SET
     locked_at = '{Current Time}'
   , locked_by = '{This Worker Name}'
  FROM cte
 WHERE t.id = cte.id
RETURNING *
```


Appendix
----

### Why don't you send Pull Request?

The `delayed_job_active_record` has opened some PRs since 2013. It seems like an inactive project.
And this bug reported to `delayed_job` a year ago, but reacted a few peaple.
Further, I think delayed_job is not popular software now.

So I'm not motivated to send PR to resolve this problem...
