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

