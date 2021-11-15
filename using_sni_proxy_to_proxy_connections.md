## Using an SNI to transparently bypass a rate limit
 
A colleague came to me with an odd problem, a free service he uses was banning his IP due to the amount of legitimate non-robotic/scripted requests he was making.

There's no option to pay and they've basically just said to him "Use a VPN when that happens", which he's been doing. But it's cumbersome turning on and off the VPN.
 
After hearing him complaining about it a few times, I decided to solve this problem for him properly.
 
```
git clone https://github.com/dlundquist/sniproxy.git
cd sniproxy/
./autogen.sh && ./configure && dpkg-buildpackage
dpkg -i ../sniproxy_0.*_amd64.deb```

service sniproxy stop

echo 'user daemon

pidfile /var/run/sniproxy.pid

error_log {
    syslog daemon
    priority notice
}

listen 443 {
    proto tls
    table https_hosts
}

table https_hosts {
    .*\\.THE-DOMAIN.com *:443
}' > /etc/sniproxy.conf
```

That's SNIProxy setup, any request going to that domain will be accepted and be resolved.
Now for the proxy trick...

```
echo 'strict_chain
quiet_mode
tcp_read_time_out 15000
tcp_connect_time_out 8000

[ProxyList]
http PROXY.END.POINT PROXY_PORT customer-****-cc-US PASSWORD' > /etc/proxychains.conf' > /etc/proxychains.conf
```
 
Anybody that has used a residential proxy service before should pick up on what that's going to do based on the username field, every new request will get a clean IP.
 
Some services let you peg an IP by passing the '-session-id' parameter as part of the username field, if you need to keep the same IP.
 
(You can use any proxy service or tor if you wish of course)
 
Modifying the services file to launch: proxychains /usr/sbin/sniproxy -c /etc/sniproxy.conf

Or just launching it from shell, yields an SNIProxy on listening on 443 that will transparently proxy any request through a random residential proxy in America to the defined destination.
 
The last step is injecting the host override either in the campus DNS server or in this case just the hosts file of my colleges computers.
 
This is essentially how 'SmartDNS' services work, which allow you to watch video services from other regions just by changing your DNS settings.

Yes you can just use a browner extension set to just proxy that domain, but then that has to be setup on every browser used and proxy parameters passed to every curl command.
 
Solution will always ensure privacy as the TLS termination still takes place on the client as well, instead of doing a reverse proxy-proxy with Nginx/HAProxy there are some modules you can use that allow upstream connections via socks or http proxies. 
