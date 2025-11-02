# Ubuntu Ethernet Login Page Not Loading (Fixed – Docker Conflict)

so this was a weird one.

in my college, when i connect the ethernet cable, a login page automatically opens (something like `gateway.iitj.ac.in`) where i have to put my username and password to access the internet.

on windows, it works perfectly.  
on ubuntu, it _used_ to work fine too — until one random day, the login page just stopped opening.  
the ethernet said “connected”, but any site (even google.com) gave **“This site can’t be reached”**.  
i even tried opening the gateway IP manually — nothing. it just said “Destination Host Unreachable”.

---

## digging into it

so after checking stuff like DNS and network status, we found something _super_ unexpected.

when i ran:

```bash
ip route
i noticed there was this line with docker0 and an IP range like 172.17.0.0/16.
and when i pinged the gateway (which for our campus is 172.17.0.3), the response said:

nginx
Copy code
From 172.17.0.1 icmp_seq=1 Destination Host Unreachable
that was the clue.
docker had literally hijacked the same IP range that my college network was using 😭
so instead of sending packets to the actual gateway, ubuntu was sending them inside docker’s virtual network.
they never even left my laptop.

the fix
so we basically told docker to stop using that range.

step 1: open (or create) this file

bash
Copy code
sudo nano /etc/docker/daemon.json
step 2: add this inside it (this changes docker’s internal network range):

json
Copy code
{
  "bip": "172.26.0.1/16"
}
(step 3: save and exit)

then run:

bash
Copy code
sudo systemctl stop docker
sudo systemctl stop docker.socket
sudo systemctl daemon-reload
sudo systemctl start docker
and reconnect ethernet.

instantly, the login page started working again ✨

what actually happened (in simple words)
docker creates its own virtual network called docker0

by default, it uses the IP range 172.17.0.0/16

our college network’s gateway also uses 172.17.x.x

so ubuntu got confused and thought the gateway was part of docker’s network

changing docker’s subnet fixed the conflict

takeaway
if your login page / intranet / proxy network stops working randomly on ubuntu,
check if docker is stealing the same subnet:

bash
Copy code
ip route
if you see 172.17.0.0/16 dev docker0,
change it in /etc/docker/daemon.json to something else like 172.26.0.1/16
and restart docker.

that’s literally it.
```
