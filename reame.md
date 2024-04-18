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

This appeared to be a valid username for the port so I can try a couple of generic passwords to see if I can get signed in at all. I tried `password`, `guest`, and `guest123` before it kicked me out of the connection. Maybe we can try brute forcing some passwords using a simple dictionary but the delay between failed attempts and getting kicked after three successful tries would make that take a while so that may be something we come back to.

