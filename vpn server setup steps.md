Yes. Since your PostgreSQL DB is already ready, now we will build the **WireGuard VPN server from zero**.

I'll assume:

```text
VPC:                 172.31.0.0/16
VPN server:          Ubuntu 24.04
VPN server subnet:   Public subnet
VPN server private IP: 172.31.10.10   ← example
DB server private IP:   172.31.20.10   ← your actual DB IP
WireGuard network:      10.200.0.0/24
VPN server WG IP:       10.200.0.1
Your laptop WG IP:      10.200.0.2
WireGuard port:         UDP 51820
```

**Important:** replace the example private IPs with your actual AWS IPs.

The architecture will be:

```text
                 YOUR WINDOWS LAPTOP
                     10.200.0.2
                          |
                          | UDP 51820
                          | encrypted
                          |
                          v
              +-----------------------+
              |   WIREGUARD SERVER    |
              |                       |
              | Public IP / EIP       |
              | Private: 172.31.10.10 |
              | WG:      10.200.0.1   |
              +-----------+-----------+
                          |
                          | AWS VPC
                          |
                          v
              +-----------------------+
              |    PRIVATE DB SERVER  |
              |                       |
              | 172.31.20.10          |
              | PostgreSQL :5432      |
              | NO PUBLIC IP           |
              +-----------------------+
```

AWS requires source/destination checking to be disabled when an EC2 instance is acting as a router/NAT/firewall. ([AWS Documentation][1])

---

# PART 1 — Prepare the AWS VPN Server

## Step 1 — Create the EC2

Create an EC2 instance:

```text
Name:
wireguard-vpn-server
```

Use:

```text
AMI:
Ubuntu Server 24.04 LTS
```

Instance type:

```text
t3.micro
```

is enough for this lab.

Networking:

```text
VPC:
YOUR EXISTING VPC

Subnet:
PUBLIC SUBNET

Auto-assign Public IP:
Disable/Enable depending on your setup
```

I strongly recommend attaching an **Elastic IP** afterward so your WireGuard endpoint doesn't change.

---

# Step 2 — Make sure the VPN server has a public IP

Go:

```text
AWS Console
→ EC2
→ Elastic IPs
→ Allocate Elastic IP
```

Then:

```text
Actions
→ Associate Elastic IP address
→ Select your wireguard-vpn-server
```

You will now have something like:

```text
Public/EIP:
3.x.x.x

Private IP:
172.31.10.10
```

Write both down.

You will eventually put the public IP into your Windows WireGuard configuration.

---

# Step 3 — Create VPN Security Group

Create:

```text
wireguard-vpn-sg
```

Inbound rules:

### Rule 1 — WireGuard

```text
Type:       Custom UDP
Port:       51820
Source:     YOUR_LAPTOP_PUBLIC_IP/32
```

For your first lab test, you can temporarily use:

```text
0.0.0.0/0
```

for UDP 51820.

But after everything works, restrict it to your public IP.

### Rule 2 — ICMP

For testing:

```text
Type: ICMP IPv4
Source: YOUR_PUBLIC_IP/32
```

You don't actually need ICMP for WireGuard itself, but it helps with troubleshooting.

### Outbound

Keep:

```text
All traffic
0.0.0.0/0
```

AWS security groups start with outbound access and you can add inbound rules as needed. ([AWS Documentation][2])

---

# Step 4 — Attach the Security Group

Go:

```text
EC2
→ Instances
→ wireguard-vpn-server
→ Security
→ Security groups
```

Attach:

```text
wireguard-vpn-sg
```

---

# PART 2 — Disable AWS Source/Destination Check

This is **mandatory** for our routing setup.

Go:

```text
EC2
→ Instances
→ wireguard-vpn-server
```

Then:

```text
Actions
→ Networking
→ Change source/destination check
```

Select:

```text
Stop
```

Save.

AWS documents this specifically for instances performing routing/NAT/firewall functions. ([AWS Documentation][1])

---

# PART 3 — Connect to VPN Server through SSM

Now:

```text
EC2
→ wireguard-vpn-server
→ Connect
→ Session Manager
→ Connect
```

You should get a shell.

Run:

```bash
sudo -i
```

---

# PART 4 — Check the VPN Server

Before installing anything, run:

```bash
hostname
```

Then:

```bash
cat /etc/os-release
```

Then:

```bash
ip addr
```

Then:

```bash
ip route
```

You should find your AWS network interface.

For example:

```text
ens5
```

and:

```text
172.31.10.10
```

Your route may look similar to:

```text
default via 172.31.10.1 dev ens5
172.31.0.0/16 dev ens5
```

### IMPORTANT

Don't assume the interface is `ens5`.

Run:

```bash
ip route
```

If you see:

```text
default via ... dev ens5
```

then your interface is:

```text
ens5
```

I'll use `ens5` from this point.

If yours is `eth0`, replace `ens5` with `eth0`.

---

# PART 5 — Test VPN Server Internet Access

Run:

```bash
ping -c 4 8.8.8.8
```

Then:

```bash
apt update
```

If `apt update` works, your VPN server has Internet access.

---

# PART 6 — Install WireGuard

Run:

```bash
apt install wireguard -y
```

Check:

```bash
wg --version
```

You should get something like:

```text
wireguard-tools v1.x.x
```

WireGuard's official documentation uses the `wg`/`wg-quick` configuration model for creating interfaces and peers. ([AWS Documentation][3])

---

# PART 7 — Create WireGuard Directory

Run:

```bash
mkdir -p /etc/wireguard
```

Then:

```bash
cd /etc/wireguard
```

Protect the directory:

```bash
chmod 700 /etc/wireguard
```

---

# PART 8 — Generate VPN Server Keys

Run:

```bash
umask 077
```

Then:

```bash
wg genkey | tee server_private.key
```

Then:

```bash
cat server_private.key | wg pubkey | tee server_public.key
```

Check the server public key:

```bash
cat server_public.key
```

You will get something like:

```text
ABCDEFxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
```

**Save this value.**

Do NOT share the private key.



# PART 10 — Enable IP Forwarding

This is what allows:

```text
Laptop
   ↓
VPN Server
   ↓
DB
```

instead of stopping at the VPN server.

Run:

```bash
nano /etc/sysctl.d/99-wireguard.conf
```

Put:

```text
net.ipv4.ip_forward=1
```

Save.

Then:

```bash
sysctl --system
```

Verify:

```bash
sysctl net.ipv4.ip_forward
```

You must get:

```text
net.ipv4.ip_forward = 1
```

---

# PART 11 — Create WireGuard Configuration

Now:

```bash
nano /etc/wireguard/wg0.conf
```

Put this:

```ini
[Interface]
Address = 10.200.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

PostUp = iptables -A FORWARD -i wg0 -o ens5 -j ACCEPT; iptables -A FORWARD -i ens5 -o wg0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT; iptables -t nat -A POSTROUTING -s 10.200.0.0/24 -o ens5 -j MASQUERADE

PostDown = iptables -D FORWARD -i wg0 -o ens5 -j ACCEPT; iptables -D FORWARD -i ens5 -o wg0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT; iptables -t nat -D POSTROUTING -s 10.200.0.0/24 -o ens5 -j MASQUERADE


[Peer]
PublicKey = LAPTOP_PUBLIC_KEY
AllowedIPs = 10.200.0.2/32
```

Now replace:

```text
SERVER_PRIVATE_KEY
```

with:

```bash
cat /etc/wireguard/server_private.key
```

And:

```text
LAPTOP_PUBLIC_KEY
```

with:

```bash
you are laptop wireguared tunnel publick key  for this follow the steps 
- > first open wiredguard in yout laptop  -> create empty tunnel  -> now youwill get public and private keys of laptop 
```

### Do NOT put your DB IP here.

That's important based on your previous question.

This:

```ini
AllowedIPs = 10.200.0.2/32
```

means:

> This peer is the owner of WireGuard address `10.200.0.2`.

The DB routing will happen through Linux/AWS routing.

---

# PART 12 — Why `ens5` appears here

These rules:

```text
-i wg0
-o ens5
```

mean:

```text
Traffic comes from:
WireGuard

Traffic leaves through:
AWS network interface
```

So:

```text
Laptop
10.200.0.2
    |
    | wg0
    ↓
VPN Server
10.200.0.1
    |
    | ens5
    ↓
AWS VPC
```

---

# PART 13 — Why MASQUERADE is there

This line:

```bash
iptables -t nat -A POSTROUTING -s 10.200.0.0/24 -o ens5 -j MASQUERADE
```

translates the VPN client's source address when traffic enters the VPC.

So instead of:

```text
10.200.0.2 → 172.31.20.10
```

the DB effectively sees traffic coming from the VPN server's AWS private address.

That makes return routing straightforward for this lab.

---

# PART 14 — Secure the configuration

Run:

```bash
chmod 600 /etc/wireguard/wg0.conf
```

Check:

```bash
ls -l /etc/wireguard/
```

You should have:

```text
server_private.key
server_public.key
wg0.conf
```

---

# PART 15 — Start WireGuard

Run:

```bash
systemctl enable wg-quick@wg0
```

Then:

```bash
systemctl start wg-quick@wg0
```

Check:

```bash
systemctl status wg-quick@wg0
```

Then:

```bash
wg show
```

You should see something similar to:

```text
interface: wg0
  public key: SERVER_PUBLIC_KEY
  private key: (hidden)
  listening port: 51820

peer: LAPTOP_PUBLIC_KEY
  allowed ips: 10.200.0.2/32
```

There won't be a handshake yet because your Windows client isn't configured.

---

# PART 16 — Verify the WireGuard interface

Run:

```bash
ip addr show wg0
```

You should see:

```text
inet 10.200.0.1/24
```

Then:

```bash
ss -lunp | grep 51820
```

You should see UDP 51820 listening.

---

# PART 17 — Check the VPN Server Routing Table

Run:

```bash
ip route
```

You should now have something conceptually like:

```text
10.200.0.0/24 dev wg0
172.31.0.0/16 dev ens5
default via 172.31.10.1 dev ens5
```

This is extremely important.

It means:

```text
10.200.0.0/24
        ↓
WireGuard

172.31.0.0/16
        ↓
AWS VPC
```

Your DB:

```text
172.31.20.10
```

is inside:

```text
172.31.0.0/16
```

Therefore Linux sends it through:

```text
ens5
```

---

# PART 18 — Now prepare your Windows WireGuard client

You said you've already installed WireGuard.

Open:

```text
WireGuard
```

Select:

```text
Add Tunnel
```

Then:

```text
Add empty tunnel
```

For now, don't activate it yet.

We'll create:

```text
[Interface]
```

and:

```text
[Peer]
```

---

# PART 19 — Windows WireGuard configuration

Your configuration should eventually look like:

```ini
[Interface]
PrivateKey = LAPTOP_PRIVATE_KEY
Address = 10.200.0.2/24

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = VPN_SERVER_PUBLIC_IP:51820
AllowedIPs = 172.31.0.0/16
PersistentKeepalive = 25
```

Replace:

```text
LAPTOP_PRIVATE_KEY
```



Replace:

```text
SERVER_PUBLIC_KEY
```

with:

```bash
cat /etc/wireguard/server_public.key
```

And:

```text
VPN_SERVER_PUBLIC_IP
```

with your VPN server's Elastic IP.

For example:

```ini
Endpoint = 13.234.56.78:51820
```

---

# PART 20 — Understand the most important line

This:

```ini
AllowedIPs = 172.31.0.0/16
```

means:

> Send AWS VPC traffic through WireGuard.

So when you eventually run:

```powershell
Test-NetConnection 172.31.20.10 -Port 5432
```

Windows sees:

```text
172.31.20.10
       ↓
matches 172.31.0.0/16
       ↓
send through WireGuard
```

That's how your laptop reaches your DB.

---

# PART 21 — Activate the Windows VPN

Click:

```text
Activate
```

Now go back to your VPN server SSM session.

Run:

```bash
wg show
```

This time you should see:

```text
latest handshake: ...
```

For example:

```text
peer: xxxxxxxxx
  endpoint: YOUR_PUBLIC_IP:xxxxx
  allowed ips: 10.200.0.2/32
  latest handshake: 20 seconds ago
  transfer: ...
```

### This is your first major success point.

If you see:

```text
latest handshake
```

then:

```text
Laptop
   |
   | encrypted WireGuard
   |
   v
VPN Server
```

is working.

---

# PART 22 — Test VPN server from Windows

On Windows PowerShell:

```powershell
ping 10.200.0.1
```

If ICMP is allowed, you should get replies.

More importantly, on the VPN server:

```bash
wg show
```

must show a recent handshake.

---

# PART 23 — Now test the DB

Your DB private IP is whatever you recorded earlier.

For example:

```text
172.31.20.10
```

On Windows:

```powershell
Test-NetConnection 172.31.20.10 -Port 5432
```

Expected:

```text
ComputerName     : 172.31.20.10
RemoteAddress    : 172.31.20.10
RemotePort       : 5432
TcpTestSucceeded : True
```

If this returns:

```text
TcpTestSucceeded : True
```

you have completed the entire VPN path:

```text
Windows
   ↓
WireGuard
   ↓
Internet
   ↓
VPN Server
   ↓
AWS VPC
   ↓
Private DB
   ↓
PostgreSQL :5432
```

---

# PART 24 — One AWS change still matters for DB Security Group

Your DB Security Group needs to allow PostgreSQL traffic from the VPN server.

Go:

```text
EC2
→ Security Groups
→ private-db-sg
→ Inbound rules
```

Add:

```text
Type:       PostgreSQL
Protocol:   TCP
Port:       5432
Source:     wireguard-vpn-sg
```

Using the VPN server's **security group as the source** is preferable to opening `5432` to the Internet. AWS supports referencing a security group as the source for inbound rules. ([AWS Documentation][2])

So:

```text
private-db-sg
        |
        | TCP 5432
        |
        v
wireguard-vpn-sg
```

Do **not** add:

```text
0.0.0.0/0
```

to PostgreSQL.

---

# PART 25 — Final complete configuration

When finished, you should have:

### AWS

```text
VPC
172.31.0.0/16

        |
        |
        +----------------------+
        |                      |
        v                      v

VPN SERVER                 DB SERVER
Public subnet              Private subnet

Private IP:                Private IP:
172.31.10.10               172.31.20.10

Public/EIP:                No public IP
3.x.x.x                    PostgreSQL 5432

WireGuard:
10.200.0.1
```

### WireGuard

```text
VPN network:

10.200.0.0/24

VPN server:
10.200.0.1

Laptop:
10.200.0.2
```

### Laptop routing

```text
172.31.0.0/16
        ↓
WireGuard
```

### VPN server routing

```text
10.200.0.0/24
        ↓
wg0

172.31.0.0/16
        ↓
ens5
```

### DB

```text
172.31.20.10:5432
```

---

# Don't move forward if one of these fails

Use this exact troubleshooting order:

```text
1. AWS Security Group
       ↓
2. VPN server has Internet
       ↓
3. wg0 exists
       ↓
4. UDP 51820 listening
       ↓
5. Windows WireGuard activated
       ↓
6. wg show → latest handshake
       ↓
7. ping 10.200.0.1
       ↓
8. VPN server → DB :5432
       ↓
9. Windows → DB :5432
```

For example, **if `wg show` does not show `latest handshake`, don't start changing PostgreSQL or DB security groups**. The problem is still between your laptop and the VPN server.

And if the handshake works but:

```powershell
Test-NetConnection <DB_PRIVATE_IP> -Port 5432
```

fails, then we move to **routing / forwarding / NAT / DB SG** troubleshooting.

### Your immediate next step

Do **Steps 1–16 first**, up to:

```bash
wg show
```

on the VPN server.

At that point, if you paste me the output of these four commands:

```bash
ip route
ip addr
systemctl status wg-quick@wg0 --no-pager
wg show
```

I can check your actual VPN server configuration before you activate the Windows client. That is safer than continuing with assumptions about your AWS interface/IPs.

[1]: https://docs.aws.amazon.com/vpc/latest/userguide/work-with-nat-instances.html?utm_source=chatgpt.com "Enable private resources to communicate outside the VPC - Amazon Virtual Private Cloud"
[2]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-security-group.html?utm_source=chatgpt.com "Create a security group for your Amazon EC2 instance - Amazon Elastic Compute Cloud"
[3]: https://docs.aws.amazon.com/vpc/latest/userguide/igw-ingress-routing.html?utm_source=chatgpt.com "Route internet traffic to a single network interface - Amazon Virtual Private Cloud"
