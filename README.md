bro-pdns-forward-dns
====================

How to setup a DNS server to serve PTR records based on Bro's log of forward DNS lookups

Logs from your IDS/servers... are full of hostnames like "23.23.170.22 [ec2-23-23-170-22.compute-1.amazonaws.com.]".

Not very useful. If it's a log of an outgoing connection, made by someone on your network, that IP address is likely
to have been resolved from a host name. You can look manually at Bro IDS dns.log for what that look up was.

This gives instructions on how to automate it, and make the information available over DNS queries.

The instructions are for debian derivative operating systems but can easily be adapted to other OSes.

TBC