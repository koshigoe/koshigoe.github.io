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

```ruby
require 'active_record'
require 'benchmark'
require 'csv'
require 'uri'

RECORD_SIZE = ENV.fetch('RECORD_SIZE', 10_000).to_i
BULK_SIZE = ENV.fetch('BULK_SIZE', 100).to_i
CONCURRENCY = ENV.fetch('CONCURRENCY', 4).to_i

dsn = URI.parse(ENV['DATABASE_URL'])

ActiveRecord::Base.establish_connection(
  adapter: 'postgresql',
  encoding: 'unicode',
  host: dsn.host,
  database: dsn.path.split('/').last,
  username: dsn.user,
  password: dsn.password,
)

columns = %i(a b c d e f g)
records = Array.new(RECORD_SIZE) { |i| [i] + Array.new(columns.size, nil) }

ActiveRecord::Base.connection_pool.with_connection do |c|
  c.create_table :test_copy_from_s, force: true do |t|
    columns.each { |name| t.string name }
  end

  c.create_table :test_bulk_insert_s, force: true do |t|
    columns.each { |name| t.string name }
  end

  c.create_table :test_single_insert_s, force: true do |t|
    columns.each { |name| t.string name }
  end

  c.create_table :test_copy_from_c, force: true do |t|
    columns.each { |name| t.string name }
  end

  c.create_table :test_bulk_insert_c, force: true do |t|
    columns.each { |name| t.string name }
  end

  c.create_table :test_single_insert_c, force: true do |t|
    columns.each { |name| t.string name }
  end

  puts c.execute('select version()').getvalue(0, 0)

  puts <<~EOF

  RECORD_SIZE = #{RECORD_SIZE}
  BULK_SIZE   = #{BULK_SIZE}
  CONCURRENCY = #{CONCURRENCY}
  EOF

  Benchmark.bm(25) do |x|
    x.report('sequential COPY FROM') do
      c.raw_connection.copy_data('COPY test_copy_from_s FROM STDIN WITH (FORMAT csv)') do
        records.each do |r|
          c.raw_connection.put_copy_data(CSV.generate_line(r, force_quotes: true))
        end
      end
    end

    x.report('sequential BULK INSERT') do
      records.each_slice(BULK_SIZE) do |rows|
        sql = 'INSERT INTO test_bulk_insert_s VALUES '
        sql << rows.map { |r| "(#{CSV.generate_line(r, force_quotes: true, quote_char: '\'').chomp})" }.join(',')
        c.execute(sql)
      end
    end

    x.report('sequential INSERT') do
      records.each do |r|
        c.execute("INSERT INTO test_single_insert_s VALUES (#{CSV.generate_line(r, force_quotes: true, quote_char: '\'').chomp})")
      end
    end

    copy_q = Queue.new
    records.each { |r| copy_q.push(r) }
    copy_q.close
    x.report('concurrent COPY FROM') do
      threads = []
      CONCURRENCY.times do |i|
        threads << Thread.start(i) do |t|
          ActiveRecord::Base.connection_pool.with_connection do |conn|
            conn.raw_connection.copy_data('COPY test_copy_from_c FROM STDIN WITH (FORMAT csv)') do
              while r = copy_q.pop
                conn.raw_connection.put_copy_data(CSV.generate_line(r, force_quotes: true))
              end
            end
          end
        end
      end
      threads.map(&:join)
    end

    bulk_q = Queue.new
    records.each_slice(BULK_SIZE / CONCURRENCY) { |rows| bulk_q.push(rows) }
    bulk_q.close
    x.report('concurrent BULK INSERT') do
      threads = []
      CONCURRENCY.times do |i|
        threads << Thread.start(i) do |t|
          ActiveRecord::Base.connection_pool.with_connection do |conn|
            while rows = bulk_q.pop
              sql = 'INSERT INTO test_bulk_insert_c VALUES '
              sql << rows.map { |r| "(#{CSV.generate_line(r, force_quotes: true, quote_char: '\'').chomp})" }.join(',')
              conn.execute(sql)
            end
          end
        end
      end
      threads.map(&:join)
    end

    ins_q = Queue.new
    records.each { |r| ins_q.push(r) }
    ins_q.close
    x.report('concurrent INSERT') do
      threads = []
      CONCURRENCY.times do |i|
        threads << Thread.start(i) do |t|
          ActiveRecord::Base.connection_pool.with_connection do |conn|
            while r = ins_q.pop
              conn.execute("INSERT INTO test_single_insert_c VALUES (#{CSV.generate_line(r, force_quotes: true, quote_char: '\'').chomp})")
            end
          end
        end
      end
      threads.map(&:join)
    end
  end
end

__END__

### macOS

ruby 2.5.1p57 (2018-03-29 revision 63029) [x86_64-darwin17]
PostgreSQL 9.6.9 on x86_64-apple-darwin17.5.0, compiled by Apple LLVM version 9.1.0 (clang-902.0.39.1), 64-bit

RECORD_SIZE = 10000
BULK_SIZE   = 100
CONCURRENCY = 4
                                user     system      total        real
sequential COPY FROM        0.638922   0.012242   0.651164 (  0.659755)
sequential BULK INSERT      0.599679   0.006814   0.606493 (  0.730043)
sequential INSERT           1.222432   0.149972   1.372404 (  3.901416)
concurrent COPY FROM        0.644782   0.046730   0.691512 (  0.672401)
concurrent BULK INSERT      0.643526   0.031800   0.675326 (  0.657583)
concurrent INSERT           1.487466   0.656513   2.143979 (  1.870171)


ruby 2.5.1p57 (2018-03-29 revision 63029) [x86_64-darwin17]
PostgreSQL 9.6.9 on x86_64-apple-darwin17.5.0, compiled by Apple LLVM version 9.1.0 (clang-902.0.39.1), 64-bit

RECORD_SIZE = 50000
BULK_SIZE   = 500
CONCURRENCY = 4
                                user     system      total        real
sequential COPY FROM        3.074100   0.044746   3.118846 (  3.122830)
sequential BULK INSERT      2.950805   0.027467   2.978272 (  3.341955)
sequential INSERT           6.175331   0.750013   6.925344 ( 19.470242)
concurrent COPY FROM        3.276832   0.229218   3.506050 (  3.376388)
concurrent BULK INSERT      3.048117   0.059778   3.107895 (  3.094699)
concurrent INSERT           7.345715   3.154695  10.500410 (  8.996774)


### Heroku

ruby 2.5.1p57 (2018-03-29 revision 63029) [x86_64-linux]
PostgreSQL 10.4 (Ubuntu 10.4-2.pgdg16.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 5.4.0-6ubuntu1~16.04.9) 5.4.0 20160609, 64-bit

RECORD_SIZE = 10000
BULK_SIZE   = 100
CONCURRENCY = 4
                                user     system      total        real
sequential COPY FROM        1.020000   0.004000   1.024000 (  1.033825)
sequential BULK INSERT      1.168000   0.036000   1.204000 (  1.556185)
sequential INSERT           2.548000   0.844000   3.392000 ( 19.355093)
concurrent COPY FROM        1.788000   0.200000   1.988000 (  1.961168)
concurrent BULK INSERT      1.512000   0.072000   1.584000 (  1.664655)
concurrent INSERT           3.028000   1.132000   4.160000 (  6.385213)


ruby 2.5.1p57 (2018-03-29 revision 63029) [x86_64-linux]
PostgreSQL 10.4 (Ubuntu 10.4-2.pgdg16.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 5.4.0-6ubuntu1~16.04.9) 5.4.0 20160609, 64-bit

RECORD_SIZE = 50000
BULK_SIZE   = 500
CONCURRENCY = 4
                                user     system      total        real
sequential COPY FROM        6.268000   0.084000   6.352000 (  6.510500)
sequential BULK INSERT      8.156000   0.316000   8.472000 ( 11.522646)
sequential INSERT          15.884000   6.164000  22.048000 (170.986838)
concurrent COPY FROM        8.132000   0.260000   8.392000 (  9.525535)
concurrent BULK INSERT      8.176000   0.012000   8.188000 ( 10.236186)
concurrent INSERT          14.576000   6.340000  20.916000 ( 51.955575)
```

```ruby
require 'active_record'
require 'csv'
require 'uri'

RECORD_SIZE = 10_000

dsn = URI.parse(ENV['DATABASE_URL'])

ActiveRecord::Base.establish_connection(
  adapter: 'postgresql',
  encoding: 'unicode',
  host: dsn.host,
  database: dsn.path.split('/').last,
  username: dsn.user,
  password: dsn.password,
)

columns = %i(a b c d e f g)
ActiveRecord::Base.connection_pool.with_connection do |c|
  puts c.execute('select version()').getvalue(0, 0)

  c.create_table :test_copy_from, force: true do |t|
    columns.each { |name| t.string name }
  end

  c.raw_connection.copy_data('COPY test_copy_from FROM STDIN WITH (FORMAT csv)') do
    RECORD_SIZE.times do |i|
      c.raw_connection.put_copy_data(CSV.generate_line([i] + columns.map { |x| x.to_s * 1_000 }, force_quotes: true))
    end
  end
end

rss = `ps -o rss= -p #{Process.pid}`.to_i

puts "RSS: #{rss / 1204} [MB]"

__END__

ruby 2.5.1p57 (2018-03-29 revision 63029) [x86_64-darwin17]
PostgreSQL 9.6.9 on x86_64-apple-darwin17.5.0, compiled by Apple LLVM version 9.1.0 (clang-902.0.39.1), 64-bit
RSS: 72 [MB]
```

```ruby
require 'active_record'
require 'csv'
require 'uri'

RECORD_SIZE = 10_000
BULK_SIZE = 1_000

dsn = URI.parse(ENV['DATABASE_URL'])

ActiveRecord::Base.establish_connection(
  adapter: 'postgresql',
  encoding: 'unicode',
  host: dsn.host,
  database: dsn.path.split('/').last,
  username: dsn.user,
  password: dsn.password,
)

columns = %i(a b c d e f g)
ActiveRecord::Base.connection_pool.with_connection do |c|
  puts c.execute('select version()').getvalue(0, 0)

  c.create_table :test_bulk_insert, force: true do |t|
    columns.each { |name| t.string name }
  end

  buffer = []
  RECORD_SIZE.times do |i|
    buffer << [i] + columns.map { |x| x.to_s * 1_000 }

    if buffer.size >= BULK_SIZE
      sql = 'INSERT INTO test_bulk_insert VALUES '
      sql << buffer.map { |r| "(#{CSV.generate_line(r, force_quotes: true, quote_char: '\'').chomp})" }.join(',')
      c.execute(sql)
      buffer.clear
    end
  end
  unless buffer.empty?
    sql = 'INSERT INTO test_bulk_insert VALUES '
    sql << buffer.map { |r| "(#{CSV.generate_line(r, force_quotes: true, quote_char: '\'').chomp})" }.join(',')
    c.execute(sql)
    buffer.clear
  end
end

rss = `ps -o rss= -p #{Process.pid}`.to_i

puts "RSS: #{rss / 1204} [MB]"

__END__

ruby 2.5.1p57 (2018-03-29 revision 63029) [x86_64-darwin17]
PostgreSQL 9.6.9 on x86_64-apple-darwin17.5.0, compiled by Apple LLVM version 9.1.0 (clang-902.0.39.1), 64-bit
RSS: 112 [MB]
```