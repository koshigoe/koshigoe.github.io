---
layout: post
title:  "Connection leaking by Net::FTP#put (Ruby)"
date:   2019-12-11 19:00:00 +09:00
categories:
- Ruby
---


What
----

The `Net::FTP#put` doesn't close FTP data connection when exception occurred.
Even though you call `Net::FTP#close`, the FTP data connection will not be closed.

```
$ ruby -v
ruby 2.6.5p114 (2019-10-01 revision 67812) [x86_64-darwin18]
```


Why
----

The `Net::FTP#storbinary` establish FTP data connection but it not close connection when exception occurred.

```ruby
# https://github.com/ruby/ruby/blob/00f4c13e22967de1d7e42085b2fc32bf6718f580/lib/net/ftp.rb#L683-L707
    def storbinary(cmd, file, blocksize, rest_offset = nil) # :yield: data
      if rest_offset
        file.seek(rest_offset, IO::SEEK_SET)
      end
      synchronize do
        with_binary(true) do
          conn = transfercmd(cmd)
          loop do
            buf = file.read(blocksize)
            break if buf == nil
            conn.write(buf)
            yield(buf) if block_given?
          end
          conn.close
          voidresp
        end
      end
    rescue Errno::EPIPE
      # EPIPE, in this case, means that the data connection was unexpectedly
      # terminated.  Rather than just raising EPIPE to the caller, check the
      # response on the control connection.  If getresp doesn't raise a more
      # appropriate exception, re-raise the original exception.
      getresp
      raise
    end
```


Reproduce problem
----

### (1) Launch FTP server

```
$ docker run --rm \
  -p 20-21:20-21 \
  -p 21100-21110:21100-21110 \
  -e FTP_USER=user \
  -e FTP_PASS=pass \
  -e PASV_ADDRESS=localhost \
  -e PASV_MIN_PORT=21100 \
  -e PASV_MAX_PORT=21110 \
  fauria/vsftpd
```

### (2) Run script

```ruby
require 'net/ftp'

def upload
  ftp = Net::FTP.new
  ftp.read_timeout = 0.1
  ftp.passive = true
  ftp.connect('localhost')
  ftp.login('user', 'pass')

  i = 0
  ftp.put(__FILE__, '/uploaded', 1) do |data|
    i += 1
    raise 'Bomb' if i > 3
  end
rescue => e
  puts e.message
ensure
  unless ftp.closed?
    ftp.close
    puts 'closed.'
  end
end

11.times { upload }

puts 'wait'
loop {}
```

```
$ ruby -v test.rb
ruby 2.6.5p114 (2019-10-01 revision 67812) [x86_64-linux]
Bomb
closed.
Net::ReadTimeout with #<Socket:fd 5>
closed.
Net::ReadTimeout with #<Socket:fd 5>
closed.
Net::ReadTimeout with #<Socket:fd 5>
closed.
Net::ReadTimeout with #<Socket:fd 5>
closed.
Net::ReadTimeout with #<Socket:fd 5>
closed.
Net::ReadTimeout with #<Socket:fd 5>
closed.
Net::ReadTimeout with #<Socket:fd 5>
closed.
Net::ReadTimeout with #<Socket:fd 5>
closed.
Net::ReadTimeout with #<Socket:fd 5>
closed.
500 OOPS: vsf_sysutil_bind, maximum number of attempts to find a listening port exceeded
closed.
wait
```

### (3) Check connection

```
$ lsof -i:21,21100-21110
COMMAND   PID     USER   FD   TYPE  DEVICE SIZE/OFF NODE NAME
ruby    31219 koshigoe    7u  IPv6 4135154      0t0  TCP localhost:36676->localhost:21106 (ESTABLISHED)
ruby    31219 koshigoe    8u  IPv6 4135255      0t0  TCP localhost:46728->localhost:21102 (ESTABLISHED)
ruby    31219 koshigoe    9u  IPv6 4137067      0t0  TCP localhost:37788->localhost:21107 (ESTABLISHED)
ruby    31219 koshigoe   10u  IPv6 4137128      0t0  TCP localhost:46008->localhost:21108 (ESTABLISHED)
ruby    31219 koshigoe   11u  IPv6 4133654      0t0  TCP localhost:39866->localhost:21101 (ESTABLISHED)
ruby    31219 koshigoe   12u  IPv6 4133727      0t0  TCP localhost:49142->localhost:21104 (ESTABLISHED)
ruby    31219 koshigoe   13u  IPv6 4133784      0t0  TCP localhost:47804->localhost:21109 (ESTABLISHED)
ruby    31219 koshigoe   14u  IPv6 4133841      0t0  TCP localhost:46430->localhost:21103 (ESTABLISHED)
ruby    31219 koshigoe   15u  IPv6 4135804      0t0  TCP localhost:52886->localhost:21105 (ESTABLISHED)
ruby    31219 koshigoe   16u  IPv6 4135854      0t0  TCP localhost:36904->localhost:21100 (ESTABLISHED)
```

### (4) Check process (vsftpd)

```
sh-4.2# ps aux | grep '[v]sftpd'
root         1  0.1  0.0  11704  2816 ?        Ss   09:56   0:00 /bin/bash /usr/sbin/run-vsftpd.sh
root        13  0.0  0.0  53296  3980 ?        S    09:56   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
nobody      56  0.0  0.0  75752  4584 ?        Ss   09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
ftp         58  0.0  0.0  75852  3884 ?        S    09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
nobody      59  0.0  0.0  75752  4584 ?        Ss   09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
ftp         61  0.0  0.0  75776  3884 ?        S    09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
nobody      62  0.0  0.0  75752  4584 ?        Ss   09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
ftp         64  0.0  0.0  75776  3884 ?        S    09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
nobody      65  0.0  0.0  75752  4584 ?        Ss   09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
ftp         67  0.0  0.0  75776  3884 ?        S    09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
nobody      68  0.0  0.0  75752  4584 ?        Ss   09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
ftp         70  0.0  0.0  75776  3884 ?        S    09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
nobody      71  0.0  0.0  75752  4584 ?        Ss   09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
ftp         73  0.0  0.0  75776  3884 ?        S    09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
nobody      74  0.0  0.0  75752  4584 ?        Ss   09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
ftp         76  0.0  0.0  75776  3884 ?        S    09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
nobody      77  0.0  0.0  75752  4584 ?        Ss   09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
ftp         79  0.0  0.0  75776  3884 ?        S    09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
nobody      80  0.0  0.0  75752  4584 ?        Ss   09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
ftp         82  0.0  0.0  75776  3884 ?        S    09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
nobody      83  0.0  0.0  75752  4584 ?        Ss   09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
ftp         85  0.0  0.0  75776  3884 ?        S    09:59   0:00 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
```


Solution
----

- [ensure to close the data connection [Bug #16413]](https://github.com/ruby/ruby/pull/2737)
