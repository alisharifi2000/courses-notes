# Session 06: Routing Part 2 (The Default Gateway)

## 1. The Impossible Problem

In the previous session, we manually added a route for a specific network (`ip route add 5.5.5.0/24...`).

**The Problem:** The Internet has millions of networks.

-   Google is `8.8.8.0`
    
-   Cloudflare is `1.1.1.0`
    
-   Facebook is `157.240.0.0`
    
-   ...and millions more.
    

It is **impossible** to manually add millions of lines of `ip route add` to your computer.

### The Solution: The Default Gateway

Instead of memorizing every map in the world, your computer just needs to know **one** thing:

> "If I don't know where this packet goes, I will send it to the **Router** (Gateway), and let him handle it."

This "Catch-All" rule is called the **Default Route**.

-   **Network IP:** `0.0.0.0`
    
-   **Netmask:** `0.0.0.0` (or `/0`)
    

**Command:**

```
# Syntax: ip route add default via {Router_IP}
sudo ip route add default via 192.168.88.1

```

---

## 2. Lab Scenario: The Chain of Gateways

Let's analyze a real-world setup involving multiple jumps (Hops) to get to the internet.

**The Setup:**

1.  **Laptop:** Connected via Wi-Fi.
    
2.  **Raspberry Pi:** Connected via Cable.
    
3.  **Mikrotik Router:** The central router for the home/lab.
    
4.  **WiMAX Modem:** Providing the actual Internet connection.
    

### Getting IPs (DHCP)

-   **Laptop:** Connects to Wi-Fi -> DHCP Request -> Gets IP `192.168.88.253`. Gateway: `192.168.88.1`.
    
-   **Raspberry Pi:** Connects Cable -> Runs `dhclient eth0` -> Gets IP `192.168.88.254`. Gateway: `192.168.88.1`.

 ---

## 3. The Packet Journey (Hop-by-Hop)

Imagine you are on the Laptop and you run `ping 4.2.2.4`. Your laptop checks its routing table. It does **not** have a route for `4.2.2.4`. So, it uses the **Default Gateway**.

![Hop-by-hop](./assets/"Hop by Hop Gateway Chain.jpg")

### Step 1: Laptop -> Mikrotik

-   **Source:** `192.168.88.253`
    
-   **Destination:** `4.2.2.4`
    
-   **Action:** Laptop sends packet to **Mikrotik** (`192.168.88.1`).
    

### Step 2: Mikrotik -> WiMAX Modem

-   The Mikrotik receives the packet. It checks _its_ routing table.
    
-   It doesn't know `4.2.2.4` either.
    
-   It pushes the packet to **its** Default Gateway (The WiMAX Modem, e.g., `192.168.1.1`).
    

### Step 3: WiMAX Modem -> ISP

-   The Modem receives the packet.
    
-   It pushes the packet to **its** Default Gateway (The ISP's Router Tower).
    

### Step 4: ISP -> Internet Backbone

-   The ISP has huge routing tables (BGP) and knows exactly which direction `4.2.2.4` is located physically in the world.
    

> **Key Concept:** The Internet is just a massive chain of routers passing the "Hot Potato" (Packet) to the next Default Gateway until it reaches someone who knows the way.

---

## 4. Tools: Traceroute & TTL

### What is TTL (Time To Live)?

If a routing loop happens (Router A sends to B, B sends to A), a packet could spin forever, clogging the internet. **TTL** is a safety mechanism. It is a number (e.g., 64) in the IP Packet.

-   Every time a packet passes through a Router (Hop), **TTL is minus 1**.
    
-   If TTL hits **0**, the packet dies, and an error is sent back ("Time to Live Exceeded").
    

### Traceroute

We can use this TTL mechanism to "Trace" the path our packet takes.

-   **Linux:** `traceroute 4.2.2.4`
    
-   **Windows:** `tracert 4.2.2.4`
    

**Example Output:**

```
traceroute to 4.2.2.4 (4.2.2.4), 30 hops max
 1  192.168.88.1 (Mikrotik)  2.123 ms   <-- Hop 1 (Your Router)
 2  192.168.1.1 (Modem)      4.051 ms   <-- Hop 2 (Your Modem)
 3  10.20.30.1 (ISP Tower)   15.221 ms  <-- Hop 3 (ISP)
 4  ...                      ...
 5  4.2.2.4 (Destination)    45.102 ms

```

_This confirms our "Chain of Gateways" theory!_

---

## 5. Review Quiz (Concepts)

**Q1: What is the main job of a Router?** **A:** To route data between **different** networks (Layer 3) using IP Addresses. It connects isolated networks together.

**Q2: Interface IP Assignment**

-   **Router Interface:** **YES**, it must have an IP to act as a Gateway.
    
-   **Switch Interface:** **NO** (generally), L2 switches pass traffic based on MAC, they don't need an IP for traffic flow (only for management).
    

**Q3: The Subnet Mask Test** Imagine two computers connected to a switch:

-   PC A: `150.70.10.10/8`
    
-   PC B: `150.90.4.1/8` **Do they Ping?**
    
-   **Answer:** **YES.**
    
-   **Why?** The mask is `/8` (`255.0.0.0`).
    
    -   PC A Network: `150.x.x.x`
        
    -   PC B Network: `150.x.x.x`
        
    -   Since the Network ID is the same (`150`), they are on the same "Street". They can talk directly without a router.
        

**Q4: Connecting to Home Wi-Fi** When you connect to Wi-Fi to access the internet, what actually happens?

-   **Answer:** DHCP gives you an IP **AND** sets a **Default Route** (Default Gateway) pointing to the Wi-Fi Router. This allows you to exit your house and enter the internet.
