Yes. Let's do **only the DB server first**. Don't install/configure WireGuard yet.

We'll install PostgreSQL, create a test database/user, make PostgreSQL listen on the private interface, and verify that PostgreSQL is working locally. Then we can move to the VPN server.

Assuming your DB server is **Ubuntu 24.04**.

## 1. Connect to DB server through SSM

AWS Console:

```text
EC2
→ Instances
→ private-db-server
→ Connect
→ Session Manager
→ Connect
```

Then:

```bash
sudo -i
```

---

## 2. Update the server

```bash
apt update
```

Then:

```bash
apt upgrade -y
```

Check Ubuntu version:

```bash
cat /etc/os-release
```

---

## 3. Install PostgreSQL

Run:

```bash
apt install postgresql postgresql-contrib -y
```

Check the service:

```bash
systemctl status postgresql
```

You should see:

```text
Active: active (exited)
```

On Ubuntu, that's normal for the PostgreSQL umbrella service.

Check the actual cluster:

```bash
pg_lsclusters
```

You should get something similar to:

```text
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main ...
```

Your PostgreSQL version might be different. **Use whatever version `pg_lsclusters` shows.**

---

# 4. Check PostgreSQL is listening

Run:

```bash
ss -lntp | grep 5432
```

Initially you will probably see something like:

```text
127.0.0.1:5432
```

That means PostgreSQL is currently accepting connections only from the DB server itself.

We'll change this later so the VPN can reach it.

---

# 5. Test PostgreSQL locally

Switch to the PostgreSQL user:

```bash
sudo -u postgres psql
```

You should get:

```text
psql (16.x)
Type "help" for help.

postgres=#
```

Run:

```sql
SELECT version();
```

Then:

```sql
SELECT current_database();
```

Exit:

```sql
\q
```

---

# 6. Create a test database

Let's create a database specifically for this VPN practical.

Run:

```bash
sudo -u postgres psql
```

Then:

```sql
CREATE DATABASE vpn_lab;
```

Verify:

```sql
\l
```

You should see:

```text
vpn_lab
```

---

# 7. Create a PostgreSQL user

Inside `psql`:

```sql
CREATE USER vpnuser WITH PASSWORD 'ChangeThisPassword123!';
```

Then give the user access to the database:

```sql
GRANT ALL PRIVILEGES ON DATABASE vpn_lab TO vpnuser;
```

Exit:

```sql
\q
```

### Important

For a real environment, use a strong unique password rather than the example above.

---


# 9. Find your DB server private IP

Run:

```bash
hostname -I
```

For example:

```text
172.31.20.10
```

Also run:

```bash
ip route
```

You should see something like:

```text
default via 172.31.20.1 dev ens5
172.31.0.0/16 dev ens5 proto kernel scope link src 172.31.20.10
```

Write down your actual private IP.

For the rest of the example I'll use:

```text
172.31.20.10
```

**Don't configure anything using that example IP if your actual IP is different.**

---

# 10. Configure PostgreSQL to accept VPC connections

First find your PostgreSQL version:

```bash
pg_lsclusters
```

Suppose it says:

```text
16 main 5432 online
```

Then the configuration directory is:

```text
/etc/postgresql/16/main/
```

If yours says version 15, use:

```text
/etc/postgresql/15/main/
```

You can avoid manually typing the version by finding the config file:

```bash
sudo -u postgres psql -c "SHOW config_file;"
```

You'll get something like:

```text
              config_file
-----------------------------------------
 /etc/postgresql/16/main/postgresql.conf
```

---

# 11. Change `listen_addresses`

Open the configuration:

```bash
nano /etc/postgresql/16/main/postgresql.conf
```

**Replace `16` with your actual PostgreSQL version.**

Find:

```text
#listen_addresses = 'localhost'
```

Change it to:

```text
listen_addresses = '*'
```

Save:

```text
Ctrl + O
Enter
Ctrl + X
```

---

# 12. Configure PostgreSQL authentication

Find your `pg_hba.conf`:

```bash
sudo -u postgres psql -c "SHOW hba_file;"
```

You'll get something like:

```text
/etc/postgresql/16/main/pg_hba.conf
```

Open it:

```bash
nano /etc/postgresql/16/main/pg_hba.conf
```

At the bottom, add:

```text
host    vpn_lab    vpnuser    172.31.0.0/16    scram-sha-256
```

This means:

```text
Database: vpn_lab
User:     vpnuser
Source:   AWS VPC 172.31.0.0/16
Auth:     SCRAM password authentication
```

For this lab, that's fine.

---

# 13. Restart PostgreSQL

Run:

```bash
systemctl restart postgresql
```

Check:

```bash
systemctl status postgresql
```

Then:

```bash
pg_lsclusters
```

It should say:

```text
online
```

---

# 14. Verify PostgreSQL is now listening

Run:

```bash
ss -lntp | grep 5432
```

You should now see something similar to:

```text
LISTEN 0 244 0.0.0.0:5432 0.0.0.0:*
```

or:

```text
LISTEN 0 244 [::]:5432 [::]:*
```

This is important.

Previously:

```text
127.0.0.1:5432
```

Now:

```text
0.0.0.0:5432
```

means PostgreSQL can accept connections arriving through the DB server's network interface.

---

# 15. Test the DB using its private IP

From the **DB server itself**, run:

```bash
psql -h 172.31.20.10 -U vpnuser -d vpn_lab
```

Replace `172.31.20.10` with your actual private IP.

Enter the password.

You should get:

```text
vpn_lab=>
```

Then:

```sql
SELECT inet_server_addr();
```

You should see:

```text
172.31.20.10
```

Exit:

```sql
\q
```

---

# 16. Configure the AWS DB Security Group

Now go to:

```text
AWS Console
→ EC2
→ Security Groups
→ private-db-sg
```

Inbound rules should eventually contain:

```text
Type: PostgreSQL
Protocol: TCP
Port: 5432
Source: VPN Server Security Group
```

For now, if your VPN server hasn't been created/configured yet, **don't open 5432 to the Internet**.

Do NOT use:

```text
0.0.0.0/0
```

for PostgreSQL.

We'll add the VPN server SG after creating the VPN server.

---

# 17. Final DB-server state before moving to WireGuard

Run these commands and make sure they work:

### PostgreSQL service

```bash
systemctl status postgresql
```

### Cluster

```bash
pg_lsclusters
```

Should show:

```text
online
```

### Port

```bash
ss -lntp | grep 5432
```

Should show PostgreSQL listening beyond localhost.

### Private IP

```bash
hostname -I
```

### Local database test

```bash
psql -h <DB_PRIVATE_IP> -U vpnuser -d vpn_lab
```

Then:

```sql
SELECT current_user;
SELECT current_database();
SELECT inet_server_addr();
```

---

## Your DB server is ready when this works

```text
DB SERVER
Private IP: 172.31.20.10
        |
        | PostgreSQL :5432
        |
        v
vpn_lab
        |
        v
vpnuser
```

Then we'll build the second half:

```text
YOUR LAPTOP
     |
     | WireGuard
     |
     v
VPN SERVER
     |
     | AWS VPC
     |
     v
PRIVATE DB SERVER
172.31.20.10:5432
```

**Stop after completing these DB steps.** Once `psql -h <DB_PRIVATE_IP> -U vpnuser -d vpn_lab` works, the next step is the **VPN server installation and configuration**, where we'll set up WireGuard, IP forwarding, routing, NAT, AWS security groups, and then finally the Windows WireGuard client.
