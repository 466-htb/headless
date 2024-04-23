# Headless

## Setup

We used a [kali-linux vm](https://www.kali.org/get-kali/#kali-virtual-machines) to attack this machine. After downloading the vpn to connect to, I ran the following command to connect and was able to establish a connection after testing with `ping`.

`sudo openvpn ~/Downloads/lab_Deanathan.ovpn`

## Searching For Vulnerabilities

### nmap Scan

The first thing I did was run `sudo nmap -sV {target_ip}` to see what ports were being used and if any identifiable services could be found.

![nmap-scan](/images/nmap-scan.png)

I could see that they had a port for ssh connections and a service that I am not familiar with called `upnp?`. I am currently unsure if nmap is saying that the returned data shown is for that service or if it was for a service on a port not being displayed but I will try and see what kind of connections I can make to the ssh port. First I will need a username to use so I tried `guest`.

![ssh-guest](/images/ssh-guest.png)

This gave me a login attempt but we cannot be sure that's a valid user since the only way to know what users are on a system is by checking the passwd file which you must be logged into the system to see. If we are able to read this file at any point in our attack this could be useful to use but for now guessing random users and passwords is not a viable strategy.

This lead me to turn my attention to the `upnp?` service which upon researching is `Universal Plug and Play`. This service seems like it may have some vulnerabilities that we can look into. Reading from [this](https://www.upguard.com/blog/what-is-upnp) source, it seems that `upnp` may be an avenue in which we can bypass firewall policies. I tried running this script but was unable to find any devices.

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

Not being able to find any devices on the upnp port, I then decided to try and visit that page in a web browser since it was returning html. That lead me to a webpage which with one button on it.

![welcome](/images/welcome.png)

After inspecting the page using the dev tools, I found that a cookie is being used called `is_admin`. This is likely the path that we will want to follow to gain access to the machine.




