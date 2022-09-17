## Defeating Captive Portals on a Headless Machine

Another day random issue.


As part of our OOB system we opted to build our own solutions instead of using appliances like OpenGear, we use RP3's with redundant PSUs, connected to facility WiFi and 4G when possible, sometimes you just can't get a signal.

Eth is connected to our network (disabled) or a 3rd party (enabled), but WiFi or 4G is preferred using network manager to perform ping checks every 20 seconds to switch to a different interface if needed.


Everything punches out to two different cloud providers using Wireguard tunnels, with SSH firewalled off except over Eth and the tunnel, plugged into the RP3's there's USB to x4 serial converters that go into the equipment.


It's a fairly hacky solution but it works very well. Gives us the advantage of knowing exactly what it's doing and it's a full fledged Linux machine unlike any appliance. In many instances it's been very useful like installing Nginx to quickly transport firmware files to equipment during deployments. Or being able to flash busted IPMI firmware locally when no other machine is operating.


Anyway all our facilities generally provide free WiFi, either with no password or a static once that never changes.


Incidentally though one facility we use globally decided to start using a Captive portal out of basically nowhere... At first this wasn't an issue the tunnels bypassed it, but eventually it got hardened down further.

I wasn't having any of that so I thought about it a little bit tried to curl the login URL on a systemd task, but the authentication hash is unique... Hmmm defeated? No fine, I'll just use Selenium then.

The only issue with this was I wrote the script below on my laptop, RP3 is ARM, ARM doesn't have Chrome builds. But luckily Chromium works just fine, but it's not technically "supported".

apt-get install chromium-driver

pip3 install selenium


Job done.

------------------

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.service import Service
from time import sleep

CHECK_URL = "http://detectportal.firefox.com/canonical.html"
CHECK_TEXT = "XXXXXXXX"
LOGIN_WAIT = 10
LOGIN_BUTTON_TEXT = "Continue to the Internet"

chrome_options = Options()
chrome_options.BinaryLocation = "/usr/bin/chromium-browser"
chrome_options.headless = True

chrome_service = Service("/usr/bin/chromedriver")
driver = webdriver.Chrome(service=chrome_service, options=chrome_options)
driver.get(CHECK_URL)
if CHECK_TEXT not in driver.title:
    print("Logged in already", driver.title)
    driver.quit()
    exit()

try:
    connect_button = driver.find_element(By.PARTIAL_LINK_TEXT, LOGIN_BUTTON_TEXT)
    if connect_button:
        connect_button.click()
        sleep(LOGIN_WAIT)
except:
    pass

driver.quit()
```
