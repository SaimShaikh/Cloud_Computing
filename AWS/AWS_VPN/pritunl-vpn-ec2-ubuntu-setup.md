# Pritunl VPN on EC2 Ubuntu — Access Private EC2 Instances (No NAT)

Pritunl is a self-hosted, open-source VPN server with a web console. This guide sets up a **Point-to-Site VPN**: your laptop connects to Pritunl, gets a VPN IP, and reaches **private EC2 instances** in the same VPC directly, using its own VPN IP end to end. **No NAT is used anywhere.**

Everything is done through the AWS Console, the Pritunl web console, and the Linux shell.

---

## Part 1 — Architecture

<img width="1167" height="1347" alt="image" src="https://github.com/user-attachments/assets/1c55735c-3670-4bfa-91dc-ac45977505a1" />

### Addressing used in this guide

| Item | Value |
|---|---|
| VPC CIDR | `10.0.0.0/16` |
| Pritunl EC2 (public subnet) | private IP `10.0.1.10` + Elastic IP |
| Target private EC2 (private subnet) | `10.0.2.20` |
| VPN client network (Pritunl) | `192.168.250.0/24` |

The VPN client network must **not overlap** with your VPC CIDR or with your home/office LAN. `192.168.0.0/24` and `192.168.1.0/24` are avoided on purpose: they are the most common home router ranges and would break routing on your laptop.

### What each piece does

| Piece | Role |
|---|---|
| **EC2 instance** | Runs Pritunl and its MongoDB database on one machine. |
| **MongoDB** | Stores organizations, users, servers, and certificates. Pritunl does not work without it. |
| **Pritunl** | Web console (admin) plus the VPN server process (terminates client tunnels). |
| **Elastic IP** | Fixed public IP. It is embedded in every client `.ovpn` profile, so it must not change. |
| **Organization** | A group of users. A server allows only users from its attached organizations. |
| **User** | A person/device with its own certificate and `.ovpn` profile. |
| **Server** | The VPN listener: protocol, port, and client IP range. |
| **Route** | Tells the client which networks to send through the tunnel. |

### How a packet travels (no NAT)

```
Laptop (VPN IP 192.168.250.2)        Pritunl EC2 (10.0.1.10)        Private EC2 (10.0.2.20)
        │                                    │                              │
        │ src=192.168.250.2 dst=10.0.2.20    │                              │
        ├────── encrypted tunnel ───────────▶│                              │
        │                                    │ forwarded unchanged          │
        │                                    ├─────── VPC network ─────────▶│
        │                                    │                              │
        │                                    │◀── reply: dst=192.168.250.2 ─┤
        │                                    │   (VPC route table sends it  │
        │                                    │    back to Pritunl)          │
        │◀────── encrypted tunnel ───────────┤                              │
```

Because the source IP is **not rewritten**, three things must be true. Each one has its own step below, and each is a common cause of failure:

1. Pritunl's route has **NAT turned off**.
2. The Pritunl instance has **Source/destination check disabled**, or AWS drops the forwarded packets.
3. The **VPC route table** has a route for `192.168.250.0/24` pointing to the Pritunl instance, or replies from the private EC2 never find their way back.

---

## Part 2 — Prerequisites

1. An AWS account that can launch EC2, allocate Elastic IPs, and edit security groups and route tables.
2. **Ubuntu 24.04 LTS** AMI. The repository lines below use the `noble` codename and only work on 24.04.
3. Instance type **`t3.small` or larger**. `t2.micro` is tight with MongoDB running alongside Pritunl.
4. A key pair to SSH into the instance.
5. An existing **private EC2 instance** in the same VPC to test against.
6. Pritunl's docs state that RHEL-based distros (Amazon Linux, AlmaLinux, Oracle Linux) get dedicated builds and testing, while Ubuntu is supported as-is. It works today, but a future OS upgrade is more likely to break on Ubuntu.

---

## Part 3 — Setup

### Step 1 — Launch the Pritunl EC2 instance

1. **EC2 → Launch instance**.
2. Name: `pritunl-vpn`.
3. AMI: **Ubuntu Server 24.04 LTS**.
4. Instance type: `t3.small`.
5. Key pair: select yours.
6. Network: **the same VPC** as your private EC2, in a **public subnet** (route to an Internet Gateway, auto-assign public IP enabled).
7. Launch.

The instance must be in the same VPC as your targets and reachable from the internet, because VPN clients connect to it from anywhere.

### Step 2 — Allocate and attach an Elastic IP

1. **EC2 → Elastic IPs → Allocate Elastic IP address**.
2. Select it → **Actions → Associate Elastic IP address** → choose `pritunl-vpn`.

Do this before downloading any client profile. The public IP is baked into the profile.

### Step 3 — Security group for the Pritunl instance

Edit the security group attached to `pritunl-vpn` and add these inbound rules:

| Type | Port | Source | Why |
|---|---|---|---|
| SSH | 22 | Your IP | Management only. Never `0.0.0.0/0`. |
| Custom TCP | 443 | Your IP | Pritunl web console. |
| Custom TCP | 80 | `0.0.0.0/0` | HTTP redirect and Let's Encrypt validation (only needed if you add a real SSL certificate later). |
| Custom UDP | 1194 | `0.0.0.0/0` | The VPN tunnel. Clients can connect from any public IP, so this must be open. Security comes from certificate authentication. |

The UDP port here must exactly match the port you set on the Pritunl server in Step 17.

### Step 4 — SSH in and install prerequisites

```bash
ssh -i your-key.pem ubuntu@<your-elastic-ip>
```
```bash
sudo apt --assume-yes install gnupg
```

`gnupg` is needed to import the repository signing keys.

### Step 5 — Add the MongoDB repository

```bash
sudo tee /etc/apt/sources.list.d/mongodb-org.list << EOF
deb [ signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse
EOF
```
```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor --yes
```

Each repository gets its own keyring file (`signed-by=...`), so its packages are verified only against its own key. This replaces the deprecated `apt-key`.

### Step 6 — Add the OpenVPN repository

Pritunl's docs use a dedicated OpenVPN build instead of Ubuntu's older package. Using the distro package is a known cause of client authentication failures.

```bash
sudo tee /etc/apt/sources.list.d/openvpn.list << EOF
deb [ signed-by=/usr/share/keyrings/openvpn-repo.gpg ] https://build.openvpn.net/debian/openvpn/stable noble main
EOF
```
```bash
curl -fsSL https://swupdate.openvpn.net/repos/repo-public.gpg | sudo gpg -o /usr/share/keyrings/openvpn-repo.gpg --dearmor --yes
```

### Step 7 — Add the Pritunl repository

```bash
sudo tee /etc/apt/sources.list.d/pritunl.list << EOF
deb [ signed-by=/usr/share/keyrings/pritunl.gpg ] https://repo.pritunl.com/stable/apt noble main
EOF
```
```bash
curl -fsSL https://raw.githubusercontent.com/pritunl/pgp/master/pritunl_repo_pub.asc | sudo gpg -o /usr/share/keyrings/pritunl.gpg --dearmor --yes
```

### Step 8 — Install everything

```bash
sudo apt update
```
```bash
sudo apt --assume-yes install pritunl openvpn mongodb-org wireguard wireguard-tools
```

WireGuard is installed so you can enable it later from the console without further package changes. This guide uses OpenVPN.

### Step 9 — Disable ufw

```bash
sudo ufw disable
```

Pritunl manages its own `iptables` rules. Running `ufw` at the same time can conflict with them. This is safe because the **security group** is your network firewall.

### Step 10 — Start and enable services

```bash
sudo systemctl start mongod
```
```bash
sudo systemctl enable mongod
```
```bash
sudo systemctl start pritunl
```
```bash
sudo systemctl enable pritunl
```

### Step 11 — Disable Source/destination check (required)

Without NAT, Pritunl forwards packets whose source IP (`192.168.250.x`) is not the instance's own IP. AWS drops such packets unless this check is off.

1. **EC2 → select `pritunl-vpn` → Actions → Networking → Change source/destination check**.
2. Tick **Stop** and save.

Do not skip this. With it enabled, the tunnel connects and everything looks fine, but no traffic reaches your private instances.

### Step 12 — Get the setup key

```bash
sudo pritunl setup-key
```

Copy the key it prints.

### Step 13 — Open the web console and finish database setup

1. Open `https://<your-elastic-ip>`. Accept the self-signed certificate warning (expected).
2. Paste the **setup key**.
3. Leave the **MongoDB URI** at its default. It points to the local MongoDB.
4. Save.

### Step 14 — Log in and secure the admin account

1. Username: `pritunl`.
2. Get the default password:
   ```bash
   sudo pritunl default-password
   ```
3. Log in, then change the username and password in the setup dialog. The public address is auto-detected and normally needs no change.

Do this before creating any users. This console is internet-facing.

### Step 15 — Create an organization

**Users → Organizations → Add Organization**. Name it `default-org`.

### Step 16 — Create a user

1. **Users → Add User**.
2. Organization: `default-org`. Name: your name or device label.
3. Save.

Pritunl generates a unique certificate for this user. That certificate authenticates the VPN connection.

### Step 17 — Create the server

1. **Servers → Add Server**.
2. Name: `main-server`.
3. Protocol: **UDP**.
4. Port: **1194**. Pritunl pre-fills a random port, so change it to match the security group from Step 3.
5. Network: **`192.168.250.0/24`**. Pritunl pre-fills a random CIDR, so change this too. It must not overlap your VPC, your home LAN, or any other network you connect from.
6. Save.

### Step 18 — Configure the route (NAT off)

This step decides what the client can reach.

1. Open `main-server` → **Routes** tab.
2. **Remove** the default `0.0.0.0/0` route. Leaving it would send all of the client's internet traffic through the tunnel, and without NAT that traffic has no working path out.
3. Click **Add Route**:
   - Network: your VPC CIDR, `10.0.0.0/16`.
   - **NAT Route: OFF (unchecked).** The dialog may have it enabled by default. Untick it.
4. Save the route.

Only traffic to the VPC CIDR goes through the tunnel. Your normal internet browsing stays on your own connection.

**DNS:** leave Pritunl's default DNS as is. Only if you need to reach instances by internal hostname, set the DNS server to your VPC resolver (VPC CIDR base + 2, e.g. `10.0.0.2`). Be aware that this pushes **all** of the client's DNS lookups through the tunnel, which makes normal browsing feel slower. Connecting by private IP avoids the problem entirely.

### Step 19 — Attach the organization and start the server

1. On `main-server`, click **Attach Organization** → select `default-org`.
2. Click **Start Server**.

### Step 20 — Add the return route in the VPC route table (required)

The private EC2 replies to `192.168.250.x`. The VPC knows nothing about that network, so you must tell it to send those replies to the Pritunl instance.

1. **VPC → Route tables**.
2. Find the route table associated with the **subnet of your private EC2** (check the instance's *Networking* tab for its subnet, then the subnet's route table).
3. **Routes → Edit routes → Add route**:
   - Destination: `192.168.250.0/24`
   - Target: **Instance** → select `pritunl-vpn`
4. Save.

Repeat for **every route table** whose subnets contain instances you want to reach. If several private subnets use different route tables, each one needs this route. If the Pritunl instance's own subnet also holds targets, its route table needs it as well.

### Step 21 — Security group on the private EC2

The target instance sees the real VPN client IP, so allow the VPN CIDR, not the Pritunl instance.

Edit the security group of the private EC2 and add inbound rules:

| Type | Port | Source |
|---|---|---|
| SSH | 22 | `192.168.250.0/24` |
| All ICMP - IPv4 | all | `192.168.250.0/24` |

Add other ports (HTTP, RDP, database ports, etc.) as needed, always with source `192.168.250.0/24`.

### Step 22 — Download the profile and connect

1. In Pritunl **Users**, click the **download** icon next to your user to get the `.ovpn` profile.
2. On your laptop, install the **Pritunl Client** from Pritunl's official client download page.
3. Import the profile and click **Connect**.

---

## Part 4 — Verify end to end

### 1. Confirm the tunnel is up (on your laptop)

```bash
ip addr show tun0
```

You should see an address from `192.168.250.0/24`. On Windows, check the Pritunl client for the assigned IP.

### 2. Confirm the route is installed (on your laptop)

```bash
ip route | grep 10.0.0.0
```

You should see `10.0.0.0/16` going through the VPN interface. If it is missing, recheck Step 18.

### 3. Reach the private EC2

```bash
ping 10.0.2.20
```
```bash
ssh -i your-key.pem ubuntu@10.0.2.20
```

Both should work, with no public IP on the target.

### 4. Prove no NAT is happening (on the private EC2)

```bash
sudo tcpdump -i any icmp
```

While pinging from your laptop, the **source IP must be your VPN IP** (`192.168.250.x`), not the Pritunl instance's `10.0.1.10`. If you see `10.0.1.10`, NAT is still on. Recheck Step 18.

---

### Final checklist

- [ ] Pritunl EC2 is in a public subnet of the same VPC, with an Elastic IP
- [ ] Security group: 22 and 443 from your IP only, UDP 1194 open, port matches the Pritunl server
- [ ] Source/destination check **disabled** on `pritunl-vpn`
- [ ] Pritunl server: UDP 1194, network `192.168.250.0/24`, organization attached, started
- [ ] Route: only the VPC (or subnet) CIDR, **NAT Route off**, no `0.0.0.0/0`
- [ ] VPC route table(s): `192.168.250.0/24` → Instance `pritunl-vpn`
- [ ] Private EC2 security group allows `192.168.250.0/24`
- [ ] `ip route get 8.8.8.8` avoids `tun0`; `ip route get 10.0.2.20` uses `tun0`

---

## Part 5 — Keep your personal internet fast while connected (split tunnel)

Goal: only traffic to your private EC2 goes through the VPN. Everything else (YouTube, browsing, calls, downloads) uses your normal connection, so there is no added lag.

This guide already does this. Step 18 removes the `0.0.0.0/0` route and adds only the VPC CIDR. Four settings keep it that way:

1. **No `0.0.0.0/0` route** on the server. If it exists, all client traffic goes through the tunnel and your speed is limited by the EC2 instance's bandwidth and distance.
2. **Route as little as needed.** Instead of the whole VPC (`10.0.0.0/16`), you can add only the subnet you use (e.g. `10.0.2.0/24`). Fewer routes means less chance of overlap with your local network.
3. **Avoid overlap with your local network.** If your home or office LAN also uses `10.0.x.x`, traffic to your own router or printer would be pushed into the tunnel and break. Check your local IP range before choosing the routed CIDR.
4. **Do not change the DNS to the VPC resolver** unless you need hostnames (see the DNS note in Step 18).

Also use **UDP** (Step 17). TCP inside a tunnel is slower and less stable.

### Verify that personal traffic bypasses the VPN

On your laptop, while connected:

```bash
ip route get 8.8.8.8
```

The output must show your normal interface (Wi-Fi/Ethernet, e.g. `wlan0`/`eth0`), **not** `tun0`. Then:

```bash
ip route get 10.0.2.20
```

This one must show `tun0`. On Windows, use `tracert 8.8.8.8` (first hop should be your home router, not `192.168.250.1`) and `tracert 10.0.2.20` (should go via the VPN).

Also compare a speed test with the VPN off and on. The results should be about the same.

**Limit:** the VPN only avoids slowing your normal traffic. Traffic to the private EC2 itself still depends on the distance to the AWS region and on the instance size. `t3.small` is enough for SSH and light use, but bulk transfers will be limited by its network bandwidth.

---

## Part 6 — Troubleshooting

| Symptom | Likely cause |
|---|---|
| Can't open `https://<elastic-ip>` | Missing 443 rule in the security group, or Elastic IP not associated. |
| Setup key rejected | Extra whitespace when copying, or `setup-key` was re-run. Re-run and copy carefully. |
| Client times out connecting | UDP port in the security group doesn't match the server port (Step 3 vs 17). |
| Client connects, ping to private EC2 gets no reply, **no packets arrive** at the target | Source/destination check still enabled (Step 11), target security group doesn't allow `192.168.250.0/24` (Step 21), or the route in Step 18 is missing. |
| Client connects, packets **arrive** at the target but ping still times out | Missing or wrong VPC route for `192.168.250.0/24` (Step 20), or it's on the wrong route table / points at the wrong instance. |
| Packets never arrive at the target even though every AWS setting looks right | A custom Network ACL on the subnet blocks `192.168.250.0/24`, or the target's own OS firewall (`iptables`/`firewalld`/Windows Firewall) drops it. |
| Works to one private subnet but not another | The second subnet uses a different route table without the Step 20 route. |
| Target sees source `10.0.1.10` instead of your VPN IP | NAT Route is still ticked on the route in Step 18. |
| Client loses normal internet or becomes slow after connecting | The `0.0.0.0/0` route was not removed (Step 18), or DNS was pointed at the VPC resolver. See Part 5. |
| Routes fail after the laptop moves networks | VPN CIDR overlaps the local LAN. Choose a different CIDR in Step 17. |
| Hostname doesn't resolve | Pritunl DNS is still the public default. Set the VPC resolver only if you need hostnames (Step 18 DNS note). |
| `mongod` won't start | Check `sudo systemctl status mongod`. Often too little RAM on `t2.micro`. |
| OpenVPN auth errors on newer clients | The dedicated OpenVPN repo from Step 6 wasn't used. |

---

## Part 7 — Cleanup

1. Pritunl console → stop and delete the server (optional).
2. Terminate the `pritunl-vpn` EC2 instance.
3. **Release the Elastic IP.** AWS charges hourly for every public IPv4 address, including one attached to a running instance, so a forgotten Elastic IP keeps billing.
4. Remove the `192.168.250.0/24` route from each route table. After termination it shows as a **blackhole** route.
5. Remove the `192.168.250.0/24` rules from the private EC2's security group.
6. Delete the Pritunl security group if it isn't used elsewhere.
