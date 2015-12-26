## Dns ReQuest checker

Basically a throwaway script to check some large requests to dns admins using
zone file formats.  Since the zones were a bit messy and the admins a bit
grumpy, drq was written to check the consistency of the requested
additions/deletions to be performed on various zones under their control.

### usage

```
usage: drq -a db.add -d db.ref [options]

 with at least one of:
 -a db.add  zone file with records to add
 -d db.del  zone file with records to delete

 options include:

 -r db.ref  zone file against which additions/removals are checked
 -q         query DNS to check add/del records (even if no -r db.ref was given)
 -c <num>   use <num> concurrent dns queries [default is 5]
 -n ns_ip   name server ip to be queried instead of default resolver
 -t         use tcp/53 queries instead of udp/53 queries
 -p         include/create PTR-records for A-records to add/del
 -s         use strict checking
 -D         print debugging info
 -v <num>   verbosity level
 -h         print this message and exit


 -a db.add
    drq checks whether these records (zone-file format) exist or not and reports
    them as ADD'd or NOT ADD'd (ie still to add).

 -d db.del
    drq checks whether these records (zone-file format) exist or not and reports
    them as DEL'd or NOT DEL'd (ie still to be deleted).

 -r db.ref
    drq uses these records as an off-line representation of DNS records
    (zone-file format) against which to check add/delete requests.  If not
    given, -q is implied.

 -q
    Query DNS to see of whether records proposed for addition (-a db.add)
    still need to be added (or not) and/or whether records proposed for deletion
    (-d db.del) still need to be removed (or not). If no -r db.ref is used, this
    option is implied (i.e. dns will be queried anyway).

 -c <num>
    Maximum number of concurrent sessions to use when sending out DNS queries.
    Defaults to 5.  You might want to lower this if using an ssh tunnel to
    direct tcp-based queries via a stepping stone onto a dns server.

 -n ns_ip
    Donot use the system's default resolver, use the nameserver at <ns_ip>
    instead.  Useful if your local system's idea of how to resolve names does
    not suit your purpose.

 -t
    Use tcp/53 to query nameservers instead of udp/53.  Mainly useful for
    proxying DNS queries via a remote jumphost using e.g. proxychains:

        $ ssh -fND 22222 your_acct@jumphost
        $ proxychains drq -r db.ref -a db.add -n <remote_ns_ip> -t

    -> 22222/tcp is the local port where proxychains expects its socks server
    (searches ./proxychains.conf, $HOME/.proxychains/proxychains.conf and
    finally /etc/proxychains.conf for its configuration).

    If you get 'connection refused' your SSH tunnel is probably not up &
    running.  Connection timeout's usually mean the -t was not given which means
    drq is using udp/53 which doesn't get proxied.

 -o file.csv
    Write results to file.csv (in csv format)

 -p
    Automatically add PTR-records for each A-record to be checked.  Sometimes
    local conventions call for such a 1-on-1 relationship.  Using -p you only
    need to provide the A-records.

 -s
    Strict checking: if a name maps to more than one address (and vice versa for
    reverse mappings) report those as 'RESIDUAL'.  Useful for situations where
    there should be only 1 mapping per name/address combination.

 -D
    Print some extra (debugging) info to stderr.

 -v <num>
    Controls the level of informational messages printed to stdout.  Defaults to
    0, which doesn't clutter the reports too much.
```

### example

Small contrived example.  Suppose you have in add.db:

```
> cat add.db

$ORIGIN example.com.
www     IN  A   93.184.216.34
addthis IN  A   10.10.10.1

$ORIGIN acme.com.
www     IN A  216.27.178.28
addthis IN A  10.10.10.2
```

and in del.db:

```
> cat del.db

$ORIGIN example.com.
ftp     IN  A   10.10.10.3

$ORIGIN acme.com.
www     IN A  216.27.178.28
```

Then running `drq -a add.db -d del.db` yields:

```
======================================================================
 RR-s from file
ZADD:  4 RR-s from 'add.db'
ZDEL:  2 RR-s from 'del.db'
ZREF:  0 RR-s from None
----------------------------------------------------------------------
 REF trim
ZREF:  0 RR-s after pruning irrelevant dns names
======================================================================
 TIMESTAMP drq vs SOA serials
>> 2015-12-26, 14:00:04
======================================================================
 Retrieving ZADD names currently missing from ZREF ..
 - retrieved 2 additional RR-s
 - ZREF now has  2 RR-s after updating
----------------------------------------------------------------------
 Retrieving ZDEL names currently missing from ZREF..
 - retrieved 1 addtional RR-s
 - ZREF now has  2 RR-s after updating
======================================================================
 REPORTS
----------------------------------------------------------------------
 CONFLICTING ADD/DEL RR-s (1)
+- www.acme.com.                       0 IN A     216.27.178.28
----------------------------------------------------------------------
 ADD'd RR-s (2/4)
+! www.example.com.                    0 IN A     93.184.216.34
+! www.acme.com.                       0 IN A     216.27.178.28
----------------------------------------------------------------------
 NOT ADD'd RR-s (2/4)
+? addthis.example.com.                0 IN A     10.10.10.1
+? addthis.acme.com.                   0 IN A     10.10.10.2
----------------------------------------------------------------------
 DEL'd RR-s (1/2)
-! ftp.example.com.                    0 IN A     10.10.10.3
----------------------------------------------------------------------
 NOT DEL'd RR-s (1/2)
-? www.acme.com.                       0 IN A     216.27.178.28
```

Which basically says:

- www.acme.com entry was listed (same entry) in both add/del db's
- www.example.com and www.acme.com were already added (ie they were in dns)
- not added were the addthis records, because they were not seen in dns
- ftp.example.com was already deleted (at least not in dns right now)
- www.acme.com is not yet deleted (still in dns)

