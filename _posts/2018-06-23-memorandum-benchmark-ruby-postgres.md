---
layout: post
title:  "Benchmark: What is the efficient way to import too many records?"
date:   2018-06-23 12:00:00 +09:00
categories:
- Ruby
- PostgreSQL
- Memorandum
---

_NOTE: This memorandum is write about PostgreSQL._

I'm interested in the way for to import too many records efficiently. The `COPY FROM` query looks good to me.

I like the bulk `INSERT` query, because it easy to use. But the bulk `INSERT` query requires memory size to buffer records.

The `COPY FROM` query doesn't require buffering records. The wrote records will buffer at PostgreSQL server. (May it cause I/O wait at PostgreSQL?)

{% gist ff6dba4e4cc1d0bcd24e7645595c3445 %}