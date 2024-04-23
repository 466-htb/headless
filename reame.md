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

## Attacking The Machine

### Malicious User Input

Clicking the only button on the page lead us to another page for contacting support. On this page was a form that we could submit to the server so perhaps some sort of injection could gain us access to the machine. 

![form](/images/form.png)

Submitting the form executes a POST request to `/support` so this is the initial destination of whatever malicious input we decide to submit.




