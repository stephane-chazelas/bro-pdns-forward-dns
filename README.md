bro-pdns-forward-dns
====================

How to setup a DNS server to serve PTR records based on Bro's log of forward DNS lookups

Logs from your IDS/servers... are full of hostnames like "23.23.170.22 [ec2-23-23-170-22.compute-1.amazonaws.com.]".

Not very useful. If it's a log of an outgoing connection, made by someone on your network, that IP address is likely
to have been resolved from a host name. You can look manually at Bro IDS dns.log for what that look up was.

This gives instructions on how to automate it, and make the information available over DNS queries.

The instructions are for debian derivative operating systems but can easily be adapted to other OSes.

How it Works
------------

Bro sniffs the network and logs all the DNS requests and their responses into a dns.log file.

A perl script watches that file (using Linux inotify to be notified whenever there are changes so it's
really real-time) and for every line added to that file that is a resolution of a host name into an
IPv4 Address, that script adds a couple of records to the database of a DNS server to specify that
the PTR record for the corresponding IP address is actually the hostname the IP address was resolved from
(in addition to the time of the request and the country where the IP address is located).

The DNS server is PowerDNS with its MySQL backend.

For example, if at 14:17:19, somebody makes a DNS request for foo.example.com, and that resolves to 1.2.3.4
(an IP address in Australia), Bro will spot that, add a corresponding entry to dns.log, the perl script will
geolookup the IP address and add a couple of entries to PowerDNS so that a PTR query of 4.3.2.1.in-addr.arpa.
(as in "dig -x 1.2.3.4") will return something like foo.example.com.C-AU.120823T141719.

Instructions
------------

You'll need to install a few packages:
  - pdns-server and pdns-backend-mysql for PowerDNS
  - liblinux-inotify2-perl, libdbd-mysql-perl, libgeo-ip-perl for the perl script

sudo apt-get install pdns-{server,backend-mysql} lib{linux-inotify2,dbd-mysql,geo-ip}-perl

Then you'll need to decide how to run the DNS server. You can either decided to make it the default DNS
server for the system (in which case you'll want to make it listen on the standard port (53) and uninstall
other DNS servers, and have "nameserver 127.0.0.1" in /etc/resolv.conf), or you'll want it to run as a
separate nameserver (possibly on a different port), which you'll have to query explicitely.

If the second approach is less intrusive, it also means you won't benefit from it straight away as all
the gethostbyaddr and anything that using it to resolve IP addresses won't be able to use it.



TBC