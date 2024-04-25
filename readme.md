# Headless

## Setup

We used a [kali-linux vm](https://www.kali.org/get-kali/#kali-virtual-machines) to attack this machine. After downloading the vpn to connect to, we ran the following command to connect and was able to establish a connection after testing with `ping`.

`sudo openvpn ~/Downloads/lab_Deanathan.ovpn`

## Searching For Vulnerabilities

### nmap Scan

The first thing we did was run `sudo nmap -sV {target_ip}` to see what ports were being used and if any identifiable services could be found.

![nmap-scan](/images/nmap-scan.png)

We could see that they had a port for ssh connections and a service that we were not familiar with called `upnp?`. We are currently unsure if nmap is saying that the returned data shown is for that service or if it was for a service on a port not being displayed but we will try and see what kind of connections we can make to the ssh port. First we will need a username to use so we tried `guest`.

![ssh-guest](/images/ssh-guest.png)

This gave us a login attempt but we cannot be sure that's a valid user since the only way to know what users are on a system is by checking the passwd file which you must be logged into the system to see. If we are able to read this file at any point in our attack this could be useful to use but for now guessing random users and passwords is not a viable strategy.

This lead us to turn our attention to the `upnp?` service which upon researching is `Universal Plug and Play`. This service seems like it may have some vulnerabilities that we can look into. Reading from [this](https://www.upguard.com/blog/what-is-upnp) source, it seems that `upnp` may be an avenue in which we can bypass firewall policies. We tried running this script but were unable to find any devices.

```python
import miniupnpc

# Create a UPnP object
upnp = miniupnpc.UPnP()

# Discover UPnP devices on the network
upnp.discover()

# Get the list of discovered devices
devices = upnp.discover()

# Print information about each discovered device
for index, device in enumerate(devices):
    print(f"Device {index + 1}:")
    print(f"    Friendly Name: {device['desc']['friendly_name']}")
    print(f"    Manufacturer: {device['desc']['manufacturer']}")
    print(f"    Model Name: {device['desc']['model_name']}")
    print(f"    Location: {device['desc']['location']}")

```

Not being able to find any devices on the upnp port, we then decided to try and visit that page in a web browser since it was returning html. That lead us to a webpage which with one button on it.

![welcome](/images/welcome.png)

After inspecting the page using the dev tools, we found that a cookie is being used called `is_admin`. This is likely the path that we will want to follow to gain access to the machine.

We also went ahead and scanned all the UDP ports as well since the TCP ports did not provide a whole lot of options. After completion of that scan using `sudo nmap -sV -sU {target_ip}`, we found the following open ports.

![udp-scan](/images/udp-scan.png)

The only service of interest to us here was the one on port 68. `DHCPC` has a known vulnerability when sending it a packet with a DNS option maliciously configured to trigger a buffer overflow as explained in [this](https://www.mcafee.com/blogs/other-blogs/mcafee-labs/dhcp-client-remote-code-execution-vulnerability-demystified/#:~:text=A%20rogue%20DHCP%20server%20in%20the%20network%20can,the%20client%20and%20take%20control%20of%20the%20system.) article. However, we are not sure of the version of `DHCPC` being used here and would only want to come back here if the hosted web app on port `5000` leads us to a dead end. 

### dirsearch

One useful tool when attacking a web server is `dirsearch`. This allows us to possibly see any other routes that may be accessible from the browser. Running `dirsearch` in kali linux requires us to clone the GitHub repo and run the executable from there using `python dirsearch.py -u http://10.10.11.8:5000`. 

![run-dirsearch](/images/run-dirsearch.png)

Running `dirsearch` resulted in the following routes being exposed:
* /support
* /dashboard

## Attacking The Machine

### Identifying An Avenue

Clicking the only button on the page lead us to another page for contacting support. On this page was a form that we could submit to the server so perhaps some sort of injection could gain us access to the machine. 

![form](/images/form.png)

Submitting the form executes a POST request to `/support` so this is the initial destination of whatever malicious input we decide to submit. Since it seems that web requests will be heavily used, we utilized [`Burp Suite`](https://portswigger.net/burp/communitydownload), a tool that intercepts web requests and allows you to fully inspect and edit them before sending them to the server. This was a tool I picked up while doing natas and found it invaluable when working with web requests. We were able to fully capture the following request when submitting the form.

```raw
POST /support HTTP/1.1
Host: 10.10.11.8:5000
Content-Length: 69
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://10.10.11.8:5000
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/123.0.6312.122 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://10.10.11.8:5000/support
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9
Cookie: is_admin=InVzZXIi.uAlmXlTvm8vyihjNaPDWnvB_Zfs
Connection: close

fname=first&lname=last&email=a%40a.c&phone=1112223333&message=message
```

Another piece of information we were able to gather is that the cookie's value is static, meaning that it does not change when getting a new one. This could lead to hints about how sessions are derived and be useful later on, but for now we are just taking note of it. Also, using `Burp's` built-in decoder, we were able to decode it from base 64 and found that it does include the string "user" meaning that we will need this to be an admin cookie at some point.

![decoded-user-cookie](/images/decoded-user-cookie.png)

### Injecting Malicious Input

The first thing we tried was inputting `<script type=’text/javascript’>alert(‘test’);</script>` into the message box to see if that would yield any responses. It flagged the request as a hacking attempt which means that the server is likely doing some sanitization to avoid this from happening. We are also not 100% sure we should have seen an alert since we don't know how the message field is being handled. Hopefully the headless administrators are on vacation and we can continue to test before getting blacklisted :)

![hacking-attempt](/images/hacking-attempt.png)

Next we tried to url encode the string.

```
%3Cscript%20type%3D%E2%80%99text%2Fjavascript%E2%80%99%3Ealert%28%E2%80%98test%E2%80%99%29%3B%3C%2Fscript%3E
```

However this resulted in the same hacking detected attempt message. We could not find any source files in the dev tools so that means that the sanitization must be happening on the server and we don't know of any queries in the url so we can't blindly try to inject there for now. This is already more advanced than natas so we will have to find a more advanced way to inject code.

One thing we tried was changing the bytes in the cookie to read `admin` instead of `user`, changing the cookie to `ImFkbWluIi64CWZeVO+by/KKGM1o8Nae8F9aZnM=`. We then realized we have no way of knowing if that worked right now but it might be good to hold onto that value for when we run into an admin page. It likely won't be valid since we would assume that the cookie is calculated in a more complex way, but it is worth trying. We then remembered that our [dirsearch](#dirsearch) yielded a route called `/dashboard` with a `401 unauthorized` return status. This means that this page would be what we want to target when we have the admin cookie.


