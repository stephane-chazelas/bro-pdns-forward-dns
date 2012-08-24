# bro-pdns-forward-dns

DISCLAIMER: use at your own risk. It's PoC quality for now. Expect it ***not***
to work out of the box.

How to setup a DNS server to serve PTR records based on Bro's log of forward
DNS lookups

Logs from your IDS/servers... are full of hostnames like `23.23.170.22
[ec2-23-23-170-22.compute-1.amazonaws.com.]`.

Not very useful. If it's a log of an outgoing connection, made by someone on
your network, that IP address is likely to have been resolved from a host name.
You can look manually at Bro IDS dns.log for what that look up was.

This gives instructions on how to automate it, and make the information
available over DNS queries.

The instructions are for debian derivative operating systems but can easily be
adapted to other OSes.

## How it Works

Bro sniffs the network and logs all the DNS requests and their responses into a
dns.log file.

A perl script watches that file (using Linux inotify to be notified whenever
there are changes so it's really real-time) and for every line added to that
file that is a resolution of a host name into an IPv4 Address, adds
a couple of records to the database of a DNS server to specify that the PTR
record for the corresponding IP address is actually the hostname the IP address
was resolved from (in addition to the time of the request and the country where
the IP address is located).

The DNS server is PowerDNS with its MySQL backend.

For example, if at 14:17:19, somebody makes a DNS request for foo.example.com,
and that resolves to 1.2.3.4 (an IP address in Australia), Bro will spot that,
add a corresponding entry to `dns.log`, the perl script will geolookup the IP
address and add a couple of entries to PowerDNS so that a PTR query of
`4.3.2.1.in-addr.arpa.` (as in `dig -x 1.2.3.4`) will return something like
`foo.example.com.C-AU.120823T141719`.

## Instructions

***DISCLAIMER***: those are instruction for debian based systems. It is recommanded to
backup every file, database... affected by those instructions before following them.

You'll need to install a few packages:
  - pdns-server and pdns-backend-mysql for PowerDNS
  - liblinux-inotify2-perl, libdbd-mysql-perl, libgeo-ip-perl for the perl script
  - optionally dnsmasq for the fallback DNS server

	sudo apt-get install pdns-{server,backend-mysql} lib{linux-inotify2,dbd-mysql,geo-ip}-perl

Then you'll need to decide how to run PowerDNS. You can either decide to
make it the default DNS server for the system (in which case you'll want to
make it listen on the standard port (53) and uninstall other DNS servers, and
have `nameserver 127.0.0.1` in `/etc/resolv.conf`), or you'll want it to run as a
separate nameserver (possibly on a different port), which you'll have to query
explicitely.

If the second approach is less intrusive, it also means you won't benefit from
it straight away as all the gethostbyaddr(3) and anything using it to resolve
IP addresses won't be able to make use of it.

Whatever approach, you'll probably want to have a fallback DNS server resolve
the queries that are not in the database (including all the A, CNAME...
queries). For that, I suggest using dnsmasq. dnsmasq is a simple lightweight
DNS server that is very easy to configure. You can configure it to query this
or that server for this or that domain. I used pdns-recursor before that, but
dnsmasq prove to be a lot more flexible. Or you can use an upstream server if
you don't need anything fancy.

Alternatively, you can have a DNS server (like dnsmasq) be the main DNS server,
so that it serves some domains like the internal ones by itself or forwarding to
other servers, and have PowerDNS be the _fallback server_. dnsmasq would then do
some caching which means some DNS responses may be up to one minute stale but on
the other end that could reduce the load on PowerDNS and MySQL.

The instruction and sample files here go with the approach with PowerDNS at the
front and dnsmasq as fallback.

### Setting up dnsmasq

Edit `/etc/default/dnsmasq` and set "ENABLED=1".

Copy etc/dnsmasq.conf to /etc, edit to change the forwarder and add other
per-domain ones if need be and restart dnsmasq:

	sudo service dnsmasq restart

Verify that it works by running:

	dig -p 5300 google.com @localhost

### Setting up PowerDNS

You'll need to create the MySQL database. Edit [share/pdns-schema.sql](bro-pdns-forward-dns/share/pdns-schema.sql) and
change the password.

	sudo mysql --defaults-file=/etc/mysql/debian.cnf < share/pdns-schema.sql

Copy [etc/powerdns/pdns.d/pdns.local](bro-pdns-forward-dns/etc/powerdns/pdns.d/pdns.local) and [etc/powerdns/pdns.d/pdns.local.gmysql](bro-pdns-forward-dns/etc/powerdns/pdns.d/pdns.local.gmysql)
to `/etc/powerdns/pdns.d.

Change the password in /etc/powerdns/pdns.d/pdns.local.gmysql to match the one
you set earlier.

The /etc/powerdns/pdns.d/pdns.local tells to use the dnsmasq we set up above as the
recursor. If you want to do without dnsmasq, you can use your upstream dns server
instead there.

Remove any other file in /etc/powerdns/pdns.d/

(Re)start pdns:

	sudo service pdns restart

Verify that it works:

	dig google.com @localhost

Then you can edit your /etc/resolv.conf to point to this name server:

	nameserver 127.0.0.1

If you're using any network manager like NetworkManager, wicd or resolvconf, use
them to set the name server instead.

### Setting up pdns-feed

That's the perl script that reads the dns.log and feeds the pdns database.

We need to create a user to run it as:

	sudo adduser --group --system --no-create-home --home=/ feed-pdns

Copy the [bin/feed-pdns](bro-pdns-forward-dns/bin/feed-pdns) to `/usr/local/sbin` and adjust the permissions:

	sudo install -m 0750 -o root -g feed-pdns bin/feed-pdns /usr/local/sbin/

Edit it to change the location of Bro logs and the database password. You may
also need to change which column in dns.log contains the IP addresses as it
seems to  be different in different versions of Bro.

Install the startup script

	sudo cp etc/init.d/feed-pdns /etc/init.d

Then install it using insserv, update-rc.d or the appropriate tool for that on
your system. Make sure it is started after pdns and mysql in the boot sequence.

Then start it:

	sudo service feed-pdns start

And test it:

	$ dig foo.com +short
	23.21.224.150
	23.21.179.138
	$ dig -x 23.21.224.150 +short
	foo.com.C-US.120824T145539.

Compare with the real PTR:

	# dig -p 5300 -x 23.21.224.150 +short
	ec2-23-21-224-150.compute-1.amazonaws.com.
