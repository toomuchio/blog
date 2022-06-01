## Complying with Forward-confirmed reverse DNS after years of not, anycast PowerDNS

The reverse DNS server this company used came bundled with the DCIM software, this had a lot of issues.
- It was out of date
- It was managed by some random DCIM software which basically just ran some ancient bash script to install PowerDNS off apt, and exec raw commands to manage it.
- There was no secondary, and no option to add a secondary.
- No forward-confirmed reverse DNS (prone to abuse)

First. thing. first! The current server had to go. Looking at this there was a few ways to do it.
- I could setup many servers with zone transfers... Yuck
- I could stick with one and do some caching with anycast... No thanks
- Or I could set up multiple anycasted 'masters' taking advantage of the MySQL backend and autonomy PowerDNS offers. Ding ding

Basically what I did was setup 4 different master PowerDNS servers in core locations around the world, I decided this based on where root servers are and where IR servers for reverse DNS are.
- Brisbane (APNIC)
- Los Angeles (ARIN)
- Brazil (LACNIC)
- Amsterdam (RIPE / APNIC (They use RIPEs servers))

Used ```dig +trace``` to determine that.

Right, so eachPowerDNS all setup the identically and independently they aren't aware of each other.

But every authoritative server runs a MariaDB instance, that is synchronised leveraging master-to-master replication running in a Galera Cluster!

So that means that if one fails, every other node will still continue to operate and once the failed node rejoins the cluster it will synchronise again.

Since it's anycasted the advertisement will be withdrawn for the failed node as well and everything will just be redirected to the next closest functioning node.

Now I could have done this using master-slave (No full redundancy) or just had a central database (No redundancy and slow).

But yes Galera has horrible performance, BUT only for writes since it has to hit every online member of the cluster, and wait for acknowledgement before it's written. It really is only meant to be used in situations where latency is <1ms.

It's an acceptable trade off because at the end of the day we have extremely minimal amounts of reverse dns updates going on, I'm talking 10s per week on average, and it's heavily rate limited to users to prevent abuse anyway.
And what I've lost in speed for write operations I've gained so much in redundancy.

This is quite simple to setup so I'm not providing any configurations there's enough guides online about Galera and PowerDNS with MariaDB backend.

The only thing I'll note here is use their repositories and PowerDNS wont upgrade the database for you. I was upgrading really old instances, you can find the patch files here https://github.com/PowerDNS/pdns/tree/master/modules/gmysqlbackend

Lucky I checked.

One final thing I'll note here, yes I could have used Route53 or some other provider. But as this network is quite large the concern was cost ballooning most charge per query even though its fractions of cents.. And for something so critical it's probably best to host it on-net.

Additionally in case of Anycast announcement failures the secondary name server, ns2 is set to one of the 4 servers (Amsterdam's) primary IP, as well. 

------------------------

Anyway the last problem I had was now I wanted to comply with Forward-confirmed reverse DNS, the real reason I got into this.

There is really no easy way to do this so I just wrote a script, attached below it's quite simple it checks every PTR in the PowerDNS database has a valid A or AAAA record.

```python
import mysql.connector, dns.resolver, dns.reversename

DB_HOST = "localhost
DB_USER = "powerdns"
DB_PASS = ""
DB_NAME = DB_USER

WHITELISTED_DOMAINS = ('a.domain.the.company.owns.and.people.are.too.lazy.to.add.a.records.a.this.time.com')

db_context = mysql.connector.connect(host=DB_HOST, user=DB_USER, password=DB_PASS, database=DB_NAME)
db_cursor = db_context.cursor()

db_cursor.execute("SELECT `name`,`content` FROM `records` WHERE `type` = 'PTR'")
resolver_context = dns.resolver.Resolver()

log = open("verify.log", "w")

for x in db_cursor.fetchall():
	ptr, domain = x

	og_domain = domain
	domain = domain.rstrip('.')

	skip = False
	for x in WHITELISTED_DOMAINS:
		if domain.endswith(x):
			skip = True
			break

	if skip:
		continue


	ip = dns.reversename.to_address(dns.name.from_text(ptr))

	valid = False
	try:
		answers = resolver_context.resolve(domain, "AAAA" if ":" in ip else "A")
		for y in answers:
			if y.to_text() == ip:
				valid = True
				break
		except:
			pass

	if not valid:
		print(ptr, ip, domain)
		log.write(f"{ptr},{ip},{domain}\n")

		db_cursor.execute("DELETE FROM `records` WHERE `type` = 'PTR' AND `name` = %s AND `content` = %s LIMIT 1", (ptr, og_domain, ))
		db_context.commit()

		print("-Deleted")

log.close()
```
