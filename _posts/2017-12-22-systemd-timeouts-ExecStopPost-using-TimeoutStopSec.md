---
layout: post
title:  "Systemd timeouts ExecStopPost using TimeoutStopSec"
date:   2017-12-22 22:00:00 +0900
categories: systemd mackerel-agent
---


Motivation
----

The [`mackerel-agent.service`](https://github.com/mackerelio/mackerel-agent/blob/915434edaf55dfd2dcd0277903f9660a5a309a6d/packaging/rpm/src/mackerel-agent.service) often failed auto-retirement. So, I will do continuous research on it.

```https://github.com/mackerelio/mackerel-agent/blob/915434edaf55dfd2dcd0277903f9660a5a309a6d/packaging/rpm/src/mackerel-agent.service
[Unit]
Description=mackerel.io agent
Documentation=https://mackerel.io/
After=network-online.target nss-lookup.target

[Service]
Environment=MACKEREL_PLUGIN_WORKDIR=/var/tmp/mackerel-agent
Environment=ROOT=/var/lib/mackerel-agent
EnvironmentFile=-/etc/sysconfig/mackerel-agent
ExecStartPre=/usr/bin/mkdir -m 777 -p $MACKEREL_PLUGIN_WORKDIR
ExecStart=/usr/bin/mackerel-agent supervise --root $ROOT $OTHER_OPTS
ExecStopPost=/bin/sh -c '[ "$AUTO_RETIREMENT" == "" ] || [ "$AUTO_RETIREMENT" == "0" ] && true || /usr/bin/mackerel-agent retire -force --root $ROOT $OTHER_OPTS'
ExecReload=/bin/kill -HUP $MAINPID
LimitNOFILE=65536
LimitNPROC=65536

[Install]
WantedBy=multi-user.target
```


Test
----

### Setup Environment

```
[my@macOS]% vagrant --version
Vagrant 2.0.1
[my@macOS]% cat <<EOF > Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "bento/centos-7.4"
end
EOF
[my@macOS]% vagrant up
[my@macOS]% vagrant ssh
```

### Case 0. Create a simple service

To watch execution command specified by `ExecStopPost=`.

```
[root@localhost ~]# cat <<EOF > /etc/systemd/system/test.service
[Unit]
Description=TEST

[Service]
User=root
ExecStart=/usr/bin/tail --pid 1 -f /dev/null
ExecStop=/usr/bin/touch /home/vagrant/ExecStop
ExecStopPost=/usr/bin/touch /home/vagrant/ExecStopPost

[Install]
WantedBy=multi-user.target
EOF
[root@localhost ~]# systemctl start test
[root@localhost ~]# systemctl status test
● test.service - TEST
   Loaded: loaded (/etc/systemd/system/test.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2017-12-22 13:26:31 UTC; 4s ago
 Main PID: 4228 (tail)
   CGroup: /system.slice/test.service
           └─4228 /usr/bin/tail --pid 1 -f /dev/null
[root@localhost ~]# ls -al /home/vagrant/Exec*
ls: cannot access /home/vagrant/Exec*: No such file or directory
$ sudo systemctl stop test
$ systemctl status test
● test.service - TEST
   Loaded: loaded (/etc/systemd/system/test.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
$ ls -al /home/vagrant/Exec*
-rw-r--r--. 1 root root 0 Dec 22 13:27 /home/vagrant/ExecStop
-rw-r--r--. 1 root root 0 Dec 22 13:27 /home/vagrant/ExecStopPost
```

The systemd created expected files. And, what happens if shutdown the machine?

```
[root@localhost ~]# rm -f /home/vagrant/Exec*
[root@localhost ~]# systemctl start test
[root@localhost ~]# systemctl status test
● test.service - TEST
   Loaded: loaded (/etc/systemd/system/test.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2017-12-22 13:31:52 UTC; 6s ago
 Main PID: 4254 (tail)
   CGroup: /system.slice/test.service
           └─4254 /usr/bin/tail --pid 1 -f /dev/null
[root@localhost ~]# shutdown -h now
```

The systemd created expected files.

```
[root@localhost ~]# ls -al /home/vagrant/Exec*
-rw-r--r--. 1 root root 0 Dec 22 13:32 /home/vagrant/ExecStop
-rw-r--r--. 1 root root 0 Dec 22 13:32 /home/vagrant/ExecStopPost
[root@localhost ~]# rm -f /home/vagrant/Exec*
```

### Case 1. Wait a moment

To check timeout behavior, add command `sleep 89` to `ExecStop` and `ExecStopPost`. The `89s` less than default value of `TimeoutStopSec` (`DefaultTimeoutStopSec`).

```
[root@localhost] cat <<EOF > /etc/systemd/system/test1.service
[Unit]
Description=TEST-1

[Service]
User=root
ExecStart=/usr/bin/tail --pid 1 -f /dev/null
ExecStop=/usr/bin/bash -c "/usr/bin/sleep 89; /usr/bin/touch /home/vagrant/ExecStop.1"
ExecStopPost=/usr/bin/bash -c "/usr/bin/sleep 89; /usr/bin/touch /home/vagrant/ExecStopPost.1"

[Install]
WantedBy=multi-user.target
EOF
[root@localhost ~]# systemctl start test1
[root@localhost ~]# systemctl status test1
● test1.service - TEST-1
   Loaded: loaded (/etc/systemd/system/test1.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2017-12-22 13:56:26 UTC; 7s ago
 Main PID: 2165 (tail)
   CGroup: /system.slice/test1.service
           └─2165 /usr/bin/tail --pid 1 -f /dev/null
[root@localhost ~]# ls -al /home/vagrant/Exec*
ls: cannot access /home/vagrant/Exec*: No such file or directory
[root@localhost ~]# systemctl stop test1
[root@localhost ~]# ls -al /home/vagrant/Exec*
-rw-r--r--. 1 root root 0 Dec 22 13:58 /home/vagrant/ExecStop.1
-rw-r--r--. 1 root root 0 Dec 22 13:59 /home/vagrant/ExecStopPost.1
[root@localhost ~]# rm -f /home/vagrant/Exec*
```

The systemd created expected files. And, what happens if shutdown the machine?

```
[root@localhost ~]# systemctl start test1
[root@localhost ~]# systemctl status test1
● test1.service - TEST-1
   Loaded: loaded (/etc/systemd/system/test1.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2017-12-22 14:00:13 UTC; 3s ago
 Main PID: 2192 (tail)
   CGroup: /system.slice/test1.service
           └─2192 /usr/bin/tail --pid 1 -f /dev/null
[root@localhost ~]# shutdown -h now
```

The systemd created expected files.

```
[root@localhost ~]# ls -al /home/vagrant/Exec*
-rw-r--r--. 1 root root 0 12月 22 14:01 /home/vagrant/ExecStop.1
-rw-r--r--. 1 root root 0 12月 22 14:03 /home/vagrant/ExecStopPost.1
[root@localhost ~]# rm -f /home/vagrant/Exec*
```

### Case 2. Timeouted

To check timeout behavior, add command `sleep 91` to `ExecStop` and `ExecStopPost`. The `91s` is greater than default value of `TimeoutStopSec`.

```
[root@localhost ~]# cat <<EOF > /etc/systemd/system/test2.service
[Unit]
Description=TEST-2

[Service]
User=root
ExecStart=/usr/bin/tail --pid 1 -f /dev/null
ExecStop=/usr/bin/bash -c "/usr/bin/sleep 91; /usr/bin/touch /home/vagrant/ExecStop.2"
ExecStopPost=/usr/bin/bash -c "/usr/bin/sleep 91; /usr/bin/touch /home/vagrant/ExecStopPost.2"

[Install]
WantedBy=multi-user.target
EOF
[root@localhost ~]# systemctl start test2
[root@localhost ~]# systemctl status test2
● test2.service - TEST-2
   Loaded: loaded (/etc/systemd/system/test2.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2017-12-22 14:09:21 UTC; 4s ago
 Main PID: 2211 (tail)
   CGroup: /system.slice/test2.service
           └─2211 /usr/bin/tail --pid 1 -f /dev/null
[root@localhost ~]# ls -al /home/vagrant/Exec*
ls: cannot access /home/vagrant/Exec*: No such file or directory
[root@localhost ~]# systemctl stop test2
[root@localhost ~]# ls -al /home/vagrant/Exec*
ls: cannot access /home/vagrant/Exec*: No such file or directory
[root@localhost ~]# systemctl status test2
● test2.service - TEST-2
   Loaded: loaded (/etc/systemd/system/test2.service; disabled; vendor preset: disabled)
   Active: failed (Result: timeout) since Fri 2017-12-22 14:12:38 UTC; 16s ago
  Process: 2223 ExecStopPost=/usr/bin/bash -c /usr/bin/sleep 91; /usr/bin/touch /home/vagrant/ExecStopPost.2 (code=killed, signal=TERM)
  Process: 2221 ExecStop=/usr/bin/bash -c /usr/bin/sleep 91; /usr/bin/touch /home/vagrant/ExecStop.2 (code=killed, signal=TERM)
  Process: 2211 ExecStart=/usr/bin/tail --pid 1 -f /dev/null (code=killed, signal=TERM)
 Main PID: 2211 (code=killed, signal=TERM)

Dec 22 14:09:21 localhost.localdomain systemd[1]: Started TEST-2.
Dec 22 14:09:21 localhost.localdomain systemd[1]: Starting TEST-2...
Dec 22 14:09:38 localhost.localdomain systemd[1]: Stopping TEST-2...
Dec 22 14:11:08 localhost.localdomain systemd[1]: test2.service stopping timed out. Terminating.
Dec 22 14:12:38 localhost.localdomain systemd[1]: test2.service stop-post timed out. Terminating.
Dec 22 14:12:38 localhost.localdomain systemd[1]: Stopped TEST-2.
Dec 22 14:12:38 localhost.localdomain systemd[1]: Unit test2.service entered failed state.
Dec 22 14:12:38 localhost.localdomain systemd[1]: test2.service failed.
```

The systemd timeouted `ExecStop` and `ExecStopPost`. So, it did not create files.

### Case 3. Timeout using TimeoutStopSec

I know `ExecStop` is timeout using `TimeoutStopSec`, but I don't know what using `ExecStopPost` to timeout.

```
[root@localhost ~]# cat <<EOF > /etc/systemd/system/test3.service
[Unit]
Description=TEST-3

[Service]
User=root
ExecStart=/usr/bin/tail --pid 1 -f /dev/null
ExecStop=/usr/bin/bash -c "/usr/bin/sleep 10; /usr/bin/touch /home/vagrant/ExecStop.3"
ExecStopPost=/usr/bin/bash -c "/usr/bin/sleep 10; /usr/bin/touch /home/vagrant/ExecStopPost.3"
TimeoutStopSec=5

[Install]
WantedBy=multi-user.target
EOF
[root@localhost ~]# systemctl start test3
[root@localhost ~]# systemctl status test3
● test3.service - TEST-3
   Loaded: loaded (/etc/systemd/system/test3.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2017-12-22 14:18:33 UTC; 9s ago
 Main PID: 2236 (tail)
   CGroup: /system.slice/test3.service
           └─2236 /usr/bin/tail --pid 1 -f /dev/null
[root@localhost ~]# systemctl stop test3
[root@localhost ~]# ls -al /home/vagrant/Exec*
ls: cannot access /home/vagrant/Exec*: No such file or directory
[root@localhost ~]# systemctl status test3
● test3.service - TEST-3
   Loaded: loaded (/etc/systemd/system/test3.service; disabled; vendor preset: disabled)
   Active: failed (Result: timeout) since Fri 2017-12-22 14:19:10 UTC; 16s ago
  Process: 2246 ExecStopPost=/usr/bin/bash -c /usr/bin/sleep 10; /usr/bin/touch /home/vagrant/ExecStopPost.3 (code=killed, signal=TERM)
  Process: 2244 ExecStop=/usr/bin/bash -c /usr/bin/sleep 10; /usr/bin/touch /home/vagrant/ExecStop.3 (code=killed, signal=TERM)
  Process: 2236 ExecStart=/usr/bin/tail --pid 1 -f /dev/null (code=killed, signal=TERM)
 Main PID: 2236 (code=killed, signal=TERM)
   CGroup: /system.slice/test3.service

Dec 22 14:18:33 localhost.localdomain systemd[1]: Started TEST-3.
Dec 22 14:18:33 localhost.localdomain systemd[1]: Starting TEST-3...
Dec 22 14:19:00 localhost.localdomain systemd[1]: Stopping TEST-3...
Dec 22 14:19:05 localhost.localdomain systemd[1]: test3.service stopping timed out. Terminating.
Dec 22 14:19:10 localhost.localdomain systemd[1]: test3.service stop-post timed out. Terminating.
Dec 22 14:19:10 localhost.localdomain systemd[1]: Stopped TEST-3.
Dec 22 14:19:10 localhost.localdomain systemd[1]: Unit test3.service entered failed state.
Dec 22 14:19:10 localhost.localdomain systemd[1]: test3.service failed.
```

The systemd timeouted `ExecStop` and `ExecStopPost` using `TimeoutStopSec`.


Conclusion
----

- The timeout of `ExecStop` and `ExecStopPost` are specified by `TimeoutStopSec`
- The default value of `TimeoutStopSec` is 90s (its value specified by `DefaultTimeoutStopSec`)
- The mackerel-agent auto-retirement executed using `ExecStopPost`
    - It will abort by timeout


Appendix
----

### Case A. Timeout `ExecStop` and Complete `ExecStopPost`

```
[root@localhost ~]# cat <<EOF > /etc/systemd/system/test4.service
[Unit]
Description=TEST-4

[Service]
User=root
ExecStart=/usr/bin/tail --pid 1 -f /dev/null
ExecStop=/usr/bin/bash -c "/usr/bin/sleep 6; /usr/bin/touch /home/vagrant/ExecStop.4"
ExecStopPost=/usr/bin/bash -c "/usr/bin/sleep 4; /usr/bin/touch /home/vagrant/ExecStopPost.4"
TimeoutStopSec=5

[Install]
WantedBy=multi-user.target
EOF
[root@localhost ~]# systemctl start test4
[root@localhost ~]# systemctl status test4
● test4.service - TEST-4
   Loaded: loaded (/etc/systemd/system/test4.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2017-12-22 14:28:49 UTC; 3s ago
 Main PID: 2289 (tail)
   CGroup: /system.slice/test4.service
           └─2289 /usr/bin/tail --pid 1 -f /dev/null
[root@localhost ~]# systemctl stop test4
[root@localhost ~]# systemctl status test4
● test4.service - TEST-4
   Loaded: loaded (/etc/systemd/system/test4.service; disabled; vendor preset: disabled)
   Active: failed (Result: timeout) since Fri 2017-12-22 14:29:21 UTC; 9s ago
  Process: 2299 ExecStopPost=/usr/bin/bash -c /usr/bin/sleep 4; /usr/bin/touch /home/vagrant/ExecStopPost.4 (code=exited, status=0/SUCCESS)
  Process: 2297 ExecStop=/usr/bin/bash -c /usr/bin/sleep 6; /usr/bin/touch /home/vagrant/ExecStop.4 (code=killed, signal=TERM)
  Process: 2289 ExecStart=/usr/bin/tail --pid 1 -f /dev/null (code=killed, signal=TERM)
 Main PID: 2289 (code=killed, signal=TERM)

Dec 22 14:28:49 localhost.localdomain systemd[1]: Started TEST-4.
Dec 22 14:28:49 localhost.localdomain systemd[1]: Starting TEST-4...
Dec 22 14:29:12 localhost.localdomain systemd[1]: Stopping TEST-4...
Dec 22 14:29:17 localhost.localdomain systemd[1]: test4.service stopping timed out. Terminating.
Dec 22 14:29:21 localhost.localdomain systemd[1]: Stopped TEST-4.
Dec 22 14:29:21 localhost.localdomain systemd[1]: Unit test4.service entered failed state.
Dec 22 14:29:21 localhost.localdomain systemd[1]: test4.service failed.
[root@localhost ~]# ls -al /home/vagrant/Exec*
-rw-r--r--. 1 root root 0 Dec 22 14:29 /home/vagrant/ExecStopPost.4
```

The systemd timeouted `ExecStop`, and completed `ExecStopPost`. In other words, the systemd execute `ExecStopPost` even if timeouted `ExecStop`.

### Case B. Kill terminated process

Create script trapping SIGTERM to sleep 10 seconds.

```
[root@localhost ~]# cat <<SHELL > /home/vagrant/script
#!/bin/bash

/usr/bin/tail --pid 1 -f /dev/null &

trap "echo trap TERM; sleep 10; echo Stopping...; kill $!" TERM

wait
SHELL
[root@localhost ~]# chmod +x /home/vagrant/script
```

Create service to check behavior about default `ExecStop` (just send `SIGTERM`).

```
[root@localhost ~]# cat <<EOF > /etc/systemd/system/test5.service
[Unit]
Description=TEST-5

[Service]
User=root
ExecStart=/home/vagrant/script
ExecStopPost=/bin/bash -c "/usr/bin/ps aux | /usr/bin/grep 'script' > /home/vagrant/ExecStopPost.5"
TimeoutStopSec=3

[Install]
WantedBy=multi-user.target
EOF
[root@localhost ~]# systemctl start test5
[root@localhost ~]# systemctl status test5
● test5.service - TEST-5
   Loaded: loaded (/etc/systemd/system/test5.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2017-12-22 14:52:23 UTC; 7s ago
 Main PID: 2339 (script)
   CGroup: /system.slice/test5.service
           ├─2339 /bin/bash /home/vagrant/script
           └─2340 /usr/bin/tail --pid 1 -f /dev/null
[root@localhost ~]# systemctl stop test5
[root@localhost ~]# systemctl status test5
● test5.service - TEST-5
   Loaded: loaded (/etc/systemd/system/test5.service; disabled; vendor preset: disabled)
   Active: failed (Result: signal) since Fri 2017-12-22 14:52:51 UTC; 4s ago
  Process: 2349 ExecStopPost=/bin/bash -c /usr/bin/ps aux | /usr/bin/grep 'script' > /home/vagrant/ExecStopPost.5 (code=exited, status=0/SUCCESS)
  Process: 2339 ExecStart=/home/vagrant/script (code=killed, signal=KILL)
 Main PID: 2339 (code=killed, signal=KILL)

Dec 22 14:52:23 localhost.localdomain systemd[1]: Started TEST-5.
Dec 22 14:52:23 localhost.localdomain systemd[1]: Starting TEST-5...
Dec 22 14:52:47 localhost.localdomain systemd[1]: Stopping TEST-5...
Dec 22 14:52:47 localhost.localdomain script[2339]: trap TERM
Dec 22 14:52:50 localhost.localdomain systemd[1]: test5.service stop-sigterm timed out. Killing.
Dec 22 14:52:50 localhost.localdomain systemd[1]: test5.service: main process exited, code=killed, status=9/KILL
Dec 22 14:52:51 localhost.localdomain systemd[1]: Stopped TEST-5.
Dec 22 14:52:51 localhost.localdomain systemd[1]: Unit test5.service entered failed state.
Dec 22 14:52:51 localhost.localdomain systemd[1]: test5.service failed.
[root@localhost]# ls -al /home/vagrant/Exec*
-rw-r--r--. 1 root root 236 Dec 22 14:52 /home/vagrant/ExecStopPost.5
$ cat /home/vagrant/ExecStopPost.5
root      2349  0.0  0.1 113128  1200 ?        Ss   14:52   0:00 /bin/bash -c /usr/bin/ps aux | /usr/bin/grep 'script' > /home/vagrant/ExecStopPost.5
root      2351  0.0  0.0 112660   948 ?        R    14:52   0:00 /usr/bin/grep script
```

The default `ExecStop` works same as `kill -TERM $MAINPID`. But, the systemd kill process if it did not complete even after `TimeoutStopSec` seconds. In this case, the systemd execute `ExecStopPost`.
