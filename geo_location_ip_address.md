## How geo location data / fixing geo location data on IP addresses.

Once again a colleague came to me with an odd problem, Akamai was incorrectly steering traffic to some IP addresses we own and announce.

All our IP addresses in Frankfurt would only on Akamai geo locate to Australia. Meaning all Akamai traffic would be served from there and carried over, also any geo restrictions enforced by the CDN would be incorrect.

The way most geo location services work is with a best guess, some use latency, some use 'in house methods' usually just user-data (parsing the location header from http queries, reverse dns etc...)

But a lot of it is actually voluntary, we publish a parse-able feed of where each one of our prefixes should be located most providers just take that and treat it as gospel, see below.

https://datatracker.ietf.org/doc/html/rfc8805

https://ripe82.ripe.net/presentations/84-RIPE82_geofeed.pdf

First thing I did was contact somebody at Akamai but I got the run around, nobody I was put in contact with seems to know how their system works or care, we aren't a customer so "Not our problem" and hard to escalate.

So I poked around a bit

Knew about (below) already which shows some pretty useful information about geo location and traffic steering, but wasn't really conclusive.

```curl -v -s -H "Pragma: akamai-x-cache-on, akamai-x-cache-remote-on, akamai-x-check-cacheable, akamai-x-get-cache-key, akamai-x-get-extracted-values, akamai-x-get-nonces, akamai-x-get-ssl-client-session-id, akamai-x-get-true-cache-key, akamai-x-serial-no, akamai-x-get-request-id" https://some.site.behind.akamai```

Then I found http://whatismyip.akamai.com/advanced which was hugely useful, so I ran this small python script on our probe machines which exist in every single location, and parsed all the information.

```python
import requests, re

AKAMAI_WMIP = http://whatismyip.akamai.com/advanced

akamai_res = {}

akamai_ip = requests.get(AKAMAI_WMIP)
if not akamai_ip.ok: exit()

text = re.sub('(<br>)+', '\n', akamai_ip.text)
text = re.sub('<[^<]+?>', '', text)

for line in text.split('\n'):
        line = line.strip()
        if not line or ':' not in line: continue
        x, y = line.split(':')
        akamai_res[x.strip()] = y.strip()

#Post to logging server
print(akamai_res)
```

I found some ranges actually had the correct geo location... Interesting they aren't older or setup any differently... Or so I thought.

I'll cut to the chase, turns out Akamai use the country element in whois records, for the lowest prefix that has a whois record.

```
inetnum:        1.1.1.0 - 1.1.1.1
netname:        MY-AWESOME-NETWORK
descr:          MY AWESOME NETWORK - New York <--- City Name
country:        US <--- Country Name
```

So I simply fixed this by adding whois records for every single /24 prefix, after a few weeks everything normalised.

I'm not sure what Akamai use for the city, I assume they parse it out of the description looking at what other networks do so I appended the city name there.

A few weeks later through a friend I confirmed that Akamai does depend on these entries to determine geo location.

So the moral of the story here is it's mostly always voluntary and as obvious as it may seem, make sure every bit of information is correct.

This isn't so obvious though, since IR's will just default country to where your company's primary address is and the geoloc entity is completely ignored / defunct. Clearly Akamai don't follow RFC8805 and parse geofeed whois entities either.

Interesting question is what to do when a range is Anycasted, set the Country to ZZ and City to "- Anycast"? Or just set it to somewhere 'central' in the internet (Like LA or AMS)?

Below is a script I wrote to quickly check records, this only works if your Maxmind data is already accurate.

```python
from ipwhois import IPWhois
import subprocess, geoip2.database, pprint, ipaddress

BGPQ4_CMD = "bgpq4 -6 AS1337"
MAXMIND_DB_LOC = "/usr/share/GeoIP/GeoLite2-City.mmdb"

ip_list = []
output = subprocess.check_output(BGPQ4_CMD, shell=True)
for line in output.decode('utf-8').split('\n'):
        prefix = line.strip().split(' ')[-1]
        if prefix == "NN" or not prefix: continue

		if '-6' in BGPQ4_CMD:
			minimal_subnet = [ipaddress.ip_network(prefix)]
		else:
			minimal_subnet = ipaddress.ip_network(prefix).subnets(new_prefix=24)

        for subnet in minimal_subnet:
                if subnet not in ip_list: ip_list.append(str(next(subnet.hosts())))

for ip in ip_list:
        with geoip2.database.Reader(MAXMIND_DB_LOC) as reader:
                maxmind_data = reader.city(ip)

        ipblock_whois = IPWhois(ip).lookup_whois()

        city_set = False
        whois_city = ipblock_whois['nets'][0]['description']
        if whois_city is None: whois_city = "-"
        maxmind_city = maxmind_data.city.name
        if maxmind_city is None: maxmind_city = "--"
        if whois_city.lower().endswith(maxmind_city.lower()): city_set = True

        country_set = False
        whois_country = ipblock_whois['nets'][0]['country']
        maxmind_country = maxmind_data.country.iso_code
        if whois_country == maxmind_country: country_set = True

        if country_set and city_set: continue

        print(' ')
        print(ip)
        print(f"WHOIS - Country: {whois_country} / City: {whois_city}")
        print(f"Maxmind: Country: {maxmind_country} / City: {maxmind_city}")
```
