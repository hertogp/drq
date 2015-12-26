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
    (-d db.del) still need to be removed (or not).

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

