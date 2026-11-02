# 🌐 Linux Network Namespaces & Enterprise Routing Lab

**🛡️ Network & Network Security Projects, Part 1 (Proof of Concept)**
👨‍💻 Author: Dharm Mavani

This lab builds a two-site enterprise network (HQ and Branch) inside a single Linux machine. It uses **network namespaces** as routers and clients, **veth pairs** as virtual cables, **static routing** between the sites and an **iptables firewall rule** that blocks and restores ping traffic.

---

## 1. 🗺️ Topology

```text
┌───────────────┐   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│   client-hq   ├───┤   router-hq   ├───┤ router-branch ├───┤ client-branch │
├───────────────┤   ├───────────────┤   ├───────────────┤   ├───────────────┤
│ 192.168.1.10  │   │ 192.168.1.1   │   │ 10.0.0.2      │   │ 192.168.2.10  │
│               │   │ 10.0.0.1      │   │ 192.168.2.1   │   │               │
└───────────────┘   └───────────────┘   └───────────────┘   └───────────────┘
  HQ LAN: 192.168.1.0/24   |   WAN: 10.0.0.0/30   |   Branch LAN: 192.168.2.0/24
```

- **Namespace**: an isolated network stack that acts as a host or a router.
- **veth pair**: a virtual Ethernet cable with one end in each namespace.
- **IP forwarding**: the kernel setting that lets a namespace route packets.

## 2. 🔢 IP addressing plan

| Namespace | Interface | IP address | Connected to |
|---|---|---|---|
| client-hq | veth-c-hq | 192.168.1.10/24 | router-hq (gateway 192.168.1.1) |
| router-hq | veth-r-hq | 192.168.1.1/24 | client-hq |
| router-hq | veth-hq-wan | 10.0.0.1/30 | router-branch (WAN) |
| router-branch | veth-branch-wan | 10.0.0.2/30 | router-hq (WAN) |
| router-branch | veth-r-branch | 192.168.2.1/24 | client-branch |
| client-branch | veth-c-branch | 192.168.2.10/24 | router-branch (gateway 192.168.2.1) |

## 3. 🧰 Requirements

- A Linux machine (this lab was done on **Kali Linux** running in VirtualBox)
- A user with `sudo` rights
- Packages: `iproute2`, `iptables`, `iputils-ping`

---

## 📦 Step 0: Install the tools

```bash
sudo apt install iproute2 iptables iputils-ping -y
```

## 🧱 Step 1: Create the namespaces

Four namespaces represent two routers and two clients.

```bash
sudo ip netns add router-hq
sudo ip netns add client-hq
sudo ip netns add router-branch
sudo ip netns add client-branch
```

Check that all four exist:

```bash
sudo ip netns list
```

## 🔌 Step 2: Connect namespaces with veth pairs

Each pair is a virtual cable. Create the pair, then move each end into its namespace.

```bash
# Client HQ <-> Router HQ
sudo ip link add veth-c-hq type veth peer name veth-r-hq
sudo ip link set veth-c-hq netns client-hq
sudo ip link set veth-r-hq netns router-hq

# Router HQ <-> Router Branch (WAN simulation)
sudo ip link add veth-hq-wan type veth peer name veth-branch-wan
sudo ip link set veth-hq-wan netns router-hq
sudo ip link set veth-branch-wan netns router-branch

# Client Branch <-> Router Branch
sudo ip link add veth-c-branch type veth peer name veth-r-branch
sudo ip link set veth-c-branch netns client-branch
sudo ip link set veth-r-branch netns router-branch
```

## 🏷️ Step 3: Assign IP addresses and bring interfaces up

Every interface needs an address and must be set to `up`.

```bash
# Client HQ and Router HQ (LAN side)
sudo ip netns exec client-hq ip addr add 192.168.1.10/24 dev veth-c-hq
sudo ip netns exec client-hq ip link set veth-c-hq up
sudo ip netns exec client-hq ip link set lo up
sudo ip netns exec router-hq ip addr add 192.168.1.1/24 dev veth-r-hq
sudo ip netns exec router-hq ip link set veth-r-hq up

# WAN link between the routers
sudo ip netns exec router-hq ip addr add 10.0.0.1/30 dev veth-hq-wan
sudo ip netns exec router-hq ip link set veth-hq-wan up
sudo ip netns exec router-branch ip addr add 10.0.0.2/30 dev veth-branch-wan
sudo ip netns exec router-branch ip link set veth-branch-wan up

# Client Branch and Router Branch (LAN side)
sudo ip netns exec client-branch ip addr add 192.168.2.10/24 dev veth-c-branch
sudo ip netns exec client-branch ip link set veth-c-branch up
sudo ip netns exec client-branch ip link set lo up
sudo ip netns exec router-branch ip addr add 192.168.2.1/24 dev veth-r-branch
sudo ip netns exec router-branch ip link set veth-r-branch up
```

Verify the addresses. Every interface should show `state UP` and the planned IP:

```bash
sudo ip netns exec router-hq ip addr
sudo ip netns exec router-branch ip addr
```

> 💡 The `lo` interface shows DOWN inside the routers. This lab does not need it.

## 🛣️ Step 4: Enable IP forwarding and configure routes

```bash
# Enable IP forwarding in both routers
sudo ip netns exec router-hq sysctl -w net.ipv4.ip_forward=1
sudo ip netns exec router-branch sysctl -w net.ipv4.ip_forward=1

# Default gateways on the clients
sudo ip netns exec client-hq ip route add default via 192.168.1.1
sudo ip netns exec client-branch ip route add default via 192.168.2.1

# Static routes between the routers
sudo ip netns exec router-hq ip route add 192.168.2.0/24 via 10.0.0.2
sudo ip netns exec router-branch ip route add 192.168.1.0/24 via 10.0.0.1
```

Check the routing table of `router-hq`:

```bash
sudo ip netns exec router-hq ip route
```

Expected output:

```
10.0.0.0/30 dev veth-hq-wan proto kernel scope link src 10.0.0.1
192.168.1.0/24 dev veth-r-hq proto kernel scope link src 192.168.1.1
192.168.2.0/24 via 10.0.0.2 dev veth-hq-wan
```

## 📡 Step 5: Test connectivity

```bash
# Gateway of HQ
sudo ip netns exec client-hq ping -c 4 192.168.1.1

# WAN link
sudo ip netns exec router-hq ping -c 4 10.0.0.2

# HQ client to Branch client (crosses both routers)
sudo ip netns exec client-hq ping -c 4 192.168.2.10

# Branch client to HQ client
sudo ip netns exec client-branch ping -c 4 192.168.1.10
```

✅ All pings should show `0% packet loss`. Pings that cross both routers show `ttl=62`, because each router subtracts 1 from the starting TTL of 64.

## 🔥 Step 6: Firewall with iptables

### 6.1 Baseline: no rules yet

```bash
sudo ip netns exec router-hq iptables -L -n -v
```

INPUT, FORWARD and OUTPUT all show `policy ACCEPT` with no rules.

### 6.2 Add a rule that blocks ping from HQ to Branch

```bash
sudo ip netns exec router-hq iptables -A FORWARD -s 192.168.1.0/24 -d 192.168.2.0/24 -p icmp --icmp-type echo-request -j DROP
```

### 6.3 Test: the ping is blocked 🚫

```bash
sudo ip netns exec client-hq ping -c 4 192.168.2.10
```

Result: `4 packets transmitted, 0 received, 100% packet loss`.

### 6.4 Delete the rule: the ping works again ✅

```bash
sudo ip netns exec router-hq iptables -D FORWARD -s 192.168.1.0/24 -d 192.168.2.0/24 -p icmp --icmp-type echo-request -j DROP
sudo ip netns exec client-hq ping -c 4 192.168.2.10
```

Result: `4 received, 0% packet loss`.

### Rule explained

| Option | Meaning |
|---|---|
| `-A FORWARD` | Append to the FORWARD chain (traffic passing through the router) |
| `-s 192.168.1.0/24` | Source: the HQ LAN |
| `-d 192.168.2.0/24` | Destination: the Branch LAN |
| `-p icmp` | Protocol: ICMP, which ping uses |
| `--icmp-type echo-request` | Only ping requests, not the replies |
| `-j DROP` | Action: discard the packet silently |

Use `-D` instead of `-A` with the same options to delete the rule.

---

## 📊 Results summary

| # | Test | Purpose | Result |
|---|---|---|---|
| 1 | router-hq to 10.0.0.2 | WAN link works | 4 of 4 replies, 0% loss |
| 2 | client-hq to 192.168.1.1 | Gateway reachable | 4 of 4 replies, 0% loss |
| 3 | client-hq to 192.168.2.10 | HQ reaches Branch | 4 of 4 replies, TTL 62 |
| 4 | client-branch to 192.168.1.10 | Branch reaches HQ | 4 of 4 replies, TTL 62 |
| 5 | client-hq to 192.168.2.10 with DROP rule | Firewall blocks ping | Blocked: 100% packet loss |
| 6 | Same ping after deleting the rule | Connectivity restored | 4 of 4 replies, 0% loss |

## 🧹 Cleanup

Deleting a namespace also removes its veth interfaces and its iptables rules.

```bash
sudo ip netns del router-hq
sudo ip netns del client-hq
sudo ip netns del router-branch
sudo ip netns del client-branch
```

> ⚠️ Everything in this lab is lost on reboot, so rerun the steps from Step 1 after restarting the machine.

## 🛠️ Troubleshooting

| Problem | What to check |
|---|---|
| `Cannot open network namespace` or permission error | Run the command with `sudo` |
| `File exists` when creating a namespace or veth | It already exists. Run the Cleanup commands and start again |
| Ping fails | Check `ip addr` (state UP and correct IP), `ip route` and `ip_forward` in both routers |
| Ping to the other office fails but the gateway works | Check the static routes on both routers and the default routes on the clients |
| Ping is blocked unexpectedly | List the rules with `sudo ip netns exec router-hq iptables -L -n -v` and delete the DROP rule |

## 🎓 What I learned

- Namespaces turn one Linux machine into several isolated hosts and routers.
- veth pairs and static routes join them like a real two-site network.
- iptables on the router controls which traffic may cross between sites.

### 🚀 Possible extensions

- Add NAT on `router-hq` to simulate internet access
- Filter specific ports or protocols instead of only ping
- Capture packets with `tcpdump` to watch the TTL change
