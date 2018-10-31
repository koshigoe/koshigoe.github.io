---
layout: post
title:  "Benchmark: Bulk INSERT vs COPY in PostgreSQL"
date:   2018-10-31 21:00:00 +09:00
categories:
- PostgreSQL
- Ruby
---


Knowledge
----

Specially about usage of [`pg`](https://bitbucket.org/ged/ruby-pg/wiki/Home) gem.

- Don't send too many rows using `COPY` at once.
- The TEXT format will be better than CSV format for `COPY`.
    - The encoder class `PG::TextEncoder::CopyRow` is nice.
- The `COPY` is faster than Bulk `INSERT`.


Benchmark
----

{% gist b8550364179af9a4536ec45e1c79ef3f %}