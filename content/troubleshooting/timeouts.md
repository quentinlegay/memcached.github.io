+++
title = 'Client Timeouts'
date = 2024-09-05T11:13:11-07:00
weight = 1
+++

## Troubleshooting Timeouts

Client complaining about "timeout errors", but not sure how to track it down?
This page will guide you through the most common causes.

### Check your firewalls

You may need to tune or disable firewalls that exist between the client and
memcached. Most firewalls track the connections in use (connection tracking)
and hitting that limit will often cause timeout errors as existing TCP
connections are dropped.

You will have to refer to your firewall vendor's documentation to figure this
out.

### Then, carefully check the usual suspects

Is the machine in swap? You will see random lag bubbles if your OS is swapping
memcached to disk periodically.

Is the machine overloaded? 0% CPU idle with a load of 400 and memcached
probably isn't getting enough CPU time. You can try `nice` or `renice`, or
just run less on the machine. If you're severely overloaded on CPU, you might
notice the "mc_conn_tester" below reporting very high wait times for `set`
commands.

### Next, mc_conn_tester.pl

Download this perl script:

https://www.memcached.org/files/mc_conn_tester.pl

```
$ ./mc_conn_tester.pl -s memcached-host -p 11211 -c 1000 --timeout 1
Averages: (conn: 0.00081381) (set: 0.00001603) (get: 0.00040122)
$ ./mc_conn_tester.pl --help
Usage:
    mc_conn_tester.pl [options]

Options:
    -s --server hostname
            Connect to an alternate hostname.
[...etc...]
```

This is a small utility for testing a quick routine directly against a memcached
instance. It will connect, attempt a couple sets, attempt a few gets, then loop and
repeat.

The utility does not use any memcached client and instead does minimal, raw
commands with the ASCII protocol. Thus helping to rule out client bugs.

If it reaches a timeout, you can see how far along in the cycle it was:

```
Fail: (timeout: 1) (elapsed: 1.00427794) (conn: 0.00000000) (set: 0.00000000) (get: 0.00000000)
Fail: (timeout: 1) (elapsed: 1.00133896) (conn: 0.00000000) (set: 0.00000000) (get: 0.00000000)
Fail: (timeout: 1) (elapsed: 1.00135303) (conn: 0.00000000) (set: 0.00000000) (get: 0.00000000)
Fail: (timeout: 1) (elapsed: 1.00145602) (conn: 0.00000000) (set: 0.00000000) (get: 0.00000000)
```

In the above line, it has a total elapsed time of the test, and then the times
at which each sub-test succeeded. In the above scanario it wasn't able to
connect to memcached, so all tests failed.

```
Fail: (timeout: 1) (elapsed: 0.00121498) (conn: 0.00114512) (set: 1.00002694) (get: 0.00000000)
Fail: (timeout: 1) (elapsed: 0.00368810) (conn: 0.00360799) (set: 1.00003314) (get: 0.00000000)
Fail: (timeout: 1) (elapsed: 0.00128603) (conn: 0.00117397) (set: 1.00004005) (get: 0.00000000)
Fail: (timeout: 1) (elapsed: 0.00115108) (conn: 0.00108099) (set: 1.00002789) (get: 0.00000000)
```

In this case, it failed waiting for "get" to complete.

If you want to log all of the tests mc_conn_tester.pl runs, open the file and
change the line:

```
my $debug = 0;
```

to

```
my $debug = 1;
```

You will then see normal lines begin with `loop:` and failed tests will start
with `Fail:` as usual.

### You're probably dropping packets.

In many cases, you're likely dropping
packets for some reason. Either a firewall is in the way and has run out of
stateful tracking slots, or your network card or switch is dropping packets.

You'll most likely see this manifest as:

```
Fail: (timeout: 1) (elapsed: 1.00145602) (conn: 0.00000000) (set: 0.00000000) (get: 0.00000000)
```

... where `conn:` and the rest are all zero. So the test was not able to
connect to memcached. You may also see random numbers across conn/set/get
mixed in with total failures.

On most systems SYN retries are 3 seconds, which is awfully long. Losing a
single SYN packet will certainly mean a timeout. This is easily proven:

```
$ ./mc_conn_tester.pl -s memcached-host -c 5000 --timeout 1 > log_one_second
&& ./mc_conn_tester.pl -s memcached-host -c 5000 --timeout 4 > log_three_seconds
&& ./mc_conn_tester.pl -s memcached-host -c 5000 --timeout 8 > log_eight_seconds
```

... Run 5000 tests each round (you can adjust this if you wish). The first one
having a timeout of 1s, which is often the client default. Then next with 4s,
which would allow for one SYN packet to be lost but still pass the test. Then
finally 8s, which allows two SYN packets to be lost in a row and yet still
succeed.

If you see the number of `Fail:` lines in each log file *decrease*, then your
network is likely dropping SYN packets.

Fixing that, however, is beyond the scope of this document.

### TIME_WAIT buckets or local port exhaustion

If mc_conn_tester.pl is seeing connection timeouts (conn: is 0), you may be
running out of local ports, firewall states, or TIME_WAIT buckets. This can
happen if you are opening and closing connections quicker than the sockets can
die off.

Use netstat to see how many you have open, and if the number is high enough
that it may be problematic. `netstat -n | grep -c TIME_WAIT`.

Details of how to tune these variables are outside the scope of this document,
but google for "Linux TCP network tuning TIME_WAIT" (or whatever OS you have) will
usually give you good results. Look for the variables below and understand
their meaning before tuning.

```
!THESE ARE EXAMPLES, NOT RECOMMENDED VALUES!
net.ipv4.ip_local_port_range = 16384 65534
net.ipv4.tcp_max_tw_buckets = 262144
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
```

Also read up on iptables and look up information on managing conntrack states
or conntrack buckets. If you find some links you love e-mail us and we'll link
them here.

### But your utility never fails!

Odds are good your client has a bug :( Try reaching out to the client author
for help. If not, please reach out to us for support via github discussions.
