---
layout: post
title:  "Introduction to dbm"
date:   2019-02-17 20:00:00 +09:00
categories:
- dbm
- Ruby
---


Which dbm?
----

It is [`dbm`](https://en.wikipedia.org/wiki/Dbm) that DataBase Manager. Neither `Deep Boltzmann Machine` nor `decibel mill watt`.


What's dbm?
----

> The name is a three letter acronym for DataBase Manager, and can also refer to the family of database engines with APIs and features derived from the original dbm.
> The dbm library stores arbitrary data by use of a single key (a primary key) in fixed-size buckets and uses hashing techniques to enable fast retrieval of the data by key.
> 
> [https://en.wikipedia.org/wiki/Dbm](https://en.wikipedia.org/wiki/Dbm)

The `dbm` (family) is useful for using as embedded KVS. It can use as dictionary like modern programing language has.

```ruby
# in Ruby
DBM.open('db') { |db| db['key'] = 'value' }
```


History
----

### `dbm` (original implementation)

> The dbm library was a simple database engine, originally written by Ken Thompson and released by AT&T in 1979.
> 
> [https://en.wikipedia.org/wiki/Dbm](https://en.wikipedia.org/wiki/Dbm)

This included in [`Version 7 Unix`](https://en.wikipedia.org/wiki/Version_7_Unix).

The length of key/content pair must be less than equal 512 bytes (includes overhead). If the keys that hash value is same, the total length of key and these contents must be less than equal 512 bytes.

### `ndbm`

The `ndbm` (New DataBase Manager) was released by Berkeley in 1986. This is a clone of `dbm`, and this implementation licensed by AT&T.

This included in 4.3 [`BSD`](https://en.wikipedia.org/wiki/Berkeley_Software_Distribution).

NOTE: The API was standardized as [The Single UNIX Specification](http://pubs.opengroup.org/onlinepubs/7908799/xsh/ndbm.h.html).

The length of key/content pair must be less than equal 1,024 bytes (includes overhead).

### `sdbm`

The `sdbm` was written by Ozan Yigit in 1987. This created as clone of `ndbm` to solve a problem about license. This is a public domain software.

### `gdbm`

The [`gdbm`](https://www.gnu.org.ua/software/gdbm/) (GNU DataBase Manager) was originally written by Philip A. Nelson in 1990. This is licensed under the GNU General Public License.

This was created to include in GNU operating system.

The length of key/content pair isn't limited.

### Berkeley DB

[`Berkeley DB`](https://en.wikipedia.org/wiki/Berkeley_DB) was created by Berkeley in 1994. This is an alternative implementation to solve a problem about license of `ndbm`.

NOTE: Now, this is [licensed by Oracle](https://www.oracle.com/technetwork/products/berkeleydb/downloads/oslicense-093458.html).

The length of key/content pair is limited by size that can load in memory. The database size is limited 256 TiB.

### cdb

The [`cdb`](https://en.wikipedia.org/wiki/Cdb_(software)) was created by Daniel J. Bernstein. This is a public domain software.

This is fast but can't modification operation (allows only create and read).

The database size is limited 4 GiB.

### QDBM

The [`QDBM`](https://fallabs.com/qdbm/) (Quick DataBase Manager) was created by Mikio Hirabayashi. This is licensed under the GNU General Public License.

It claim fast and efficient more than `gdbm`.

### Tokyo/Kyoto Cabinet

[`Tokyo Cabinet`](https://fallabs.com/tokyocabinet/index.html) and [`Kyoto Cabinet`](https://fallabs.com/kyotocabinet/) are created by Mikio Hirabayashi. This is licensed under the GNU General Public License.

These are developed as the successor of GDBM and QDBM. It claim they replaces conventional DBM products.

### etc.

- [`MDBM`](https://github.com/yahoo/mdbm)
- [`LMDB`](https://en.wikipedia.org/wiki/Lightning_Memory-Mapped_Database) (Lightning Memory-Mapped Database)
- [`JDBM`](http://jdbm.sourceforge.net/)


Performance
----

I'm interested in difference to built-in dictionary type.

[I tested about following case](https://github.com/koshigoe/benchmark-ruby-dbm):

- 500,000 items
- 1,008 Bytes each item

### Time

- time for writing all of items
- time for reading all of items

| type            | write (u/s) [s]                   | read (u/s) [s]                 |
| --------------- | --------------------------------- | ------------------------------ |
| `Hash`          | 1.572769 (1.281160 / 0.263842)    | 0.335770 (0.300842 / 0.007701) |
| `sdbm`          | 7.689297 (1.739403 / 5.900456)    | 4.128331 (1.215769 / 2.888625) |
| `gdbm`          | 33.555130 (11.001524 / 22.242232) | 0.860940 (0.802744 / 0.031668) |
| `Berkeley DB`   | 33.210002 (13.640456 / 19.278301) | 1.522672 (0.821140 / 0.675800) |
| `qdbm`          | 21.922223 (6.524669 / 15.090120)  | 1.270316 (0.598023 / 0.646751) |
| `cdb`           | 2.248181 (1.628379 / 0.509936)    | 0.514831 (0.443600 / 0.044133) |
| `Kyoto Cabinet` | 9.228834 (4.819895 / 4.346801)    | 1.154338 (0.607743 / 0.521693) |

### File size

| type          | dir [B]   | pag [B]        |
| ------------- | --------- | -------------- |
| `Hash`        | N/A       | N/A            |
| `sdbm`        | 4,194,304 | 34,356,848,640 |
| `gdbm`        | 16        | 536,682,496    |
| `qdbm`        | 181       | 526,291,472    |

| type            | DB [B]      |
| --------------- | ----------- |
| `Berkeley DB`   | 774,090,752 |
| `cdb`           | 516,002,048 |
| `Kyoto Cabinet` | 522,297,720 |

### Memory size (RSS)

| type            | write [KiB] | read [KiB] |
| --------------- | ------------| ---------- |
| `Hash`          | 1,029,272   | 15,312     |
| `sdbm`          | 4,536       | 3,420      |
| `gdbm`          | 4,096       | 3,176      |
| `Berkeley DB`   | 5,076       | 3,952      |
| `qdbm`          | 4,052       | 2,748      |
| `cdb`           | 14,488      | 7,004      |
| `Kyoto Cabinet` | 5,288       | 4,016      |


Consideration
----

If the program is allowed to use enough memory, built-in dictionary (in-memory) may good choice.

If the program isn't allowed to use enough memory, `cdb` (`TinyCDB`) or `gdbm` may good choice.

If the database may become huge, maybe the `Kyoto Cabinet` help you. (I don't know...)

Enjoy your benchmark!


Appendix A: dbm for Ruby
----

Ruby standard library includes [`dbm`](https://docs.ruby-lang.org/ja/latest/library/dbm.html). Ruby's `dbm` library is frontend for `ndbm` compatibilities.

### Detect `ndbm` compatibility

Default, builder detect one of `ndbm` compatibility:

1. 4.3BSD original ndbm
1. Berkeley DB
1. `gdbm`
1. `qdbm`

```ruby
# https://github.com/ruby/dbm/blob/18da9b1c4248b2b92046bd7150a08ee645d63a50/ext/dbm/extconf.rb#L24-L42
if dblib = with_config("dbm-type", nil)
  dblib = dblib.split(/[ ,]+/)
else
  dblib = %w(libc db db2 db1 db6 db5 db4 db3 gdbm_compat gdbm qdbm)
end

headers = {
  "libc" => ["ndbm.h"], # 4.3BSD original ndbm, Berkeley DB 1 in 4.4BSD libc.
  "db" => ["db.h"],
  "db1" => ["db1/ndbm.h", "db1.h", "ndbm.h"],
  "db2" => ["db2/db.h", "db2.h", "db.h"],
  "db3" => ["db3/db.h", "db3.h", "db.h"],
  "db4" => ["db4/db.h", "db4.h", "db.h"],
  "db5" => ["db5/db.h", "db5.h", "db.h"],
  "db6" => ["db6/db.h", "db6.h", "db.h"],
  "gdbm_compat" => ["gdbm-ndbm.h", "gdbm/ndbm.h", "ndbm.h"], # GDBM since 1.8.1
  "gdbm" => ["gdbm-ndbm.h", "gdbm/ndbm.h", "ndbm.h"], # GDBM until 1.8.0
  "qdbm" => ["qdbm/relic.h", "relic.h"],
}
```

### Build with Bundler

The [`ruby/dbm`](https://github.com/ruby/dbm) was promoted default gems at Ruby 2.5. So, we can chose `ndbm` compatibility each project (before Ruby 2.4, it was decided at compile Ruby).

For example, if you'd like to use `qdbm`:

1. Install header file
    - e.g. `sudo apt install libqdbm-dev`
1. Specify build optoin `--with-dbm-type=qdbm`
    - e.g. `bundle config --local build.dbm --with-dbm-type=qdbm`
1. Add `gem 'dbm'` to your `Gemfile`
1. Install gem using bundler

```
$ bundle exec ruby -rbundler/setup -rdbm -e 'puts DBM::VERSION'
QDBM 1.8.78
```
