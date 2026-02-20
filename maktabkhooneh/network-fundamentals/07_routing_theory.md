
# Session 07: Routing Theory, ISPs, and Protocols

## 1. What is an ISP?

**ISP** stands for **Internet Service Provider** (e.g., Comcast, Vodafone, Shatel, Irancell). Simply put, an ISP is a company that has a massive, high-speed connection to the global Internet and sells smaller pieces of that connection to regular users.

## 2. The "Naive" ISP Setup (And Why It Fails)

Imagine you want to start your own ISP. The simplest (but worst) way to do it is to take a computer, connect it to the internet, run a DHCP server (e.g., handing out `1.1.1.x` IPs), and run long Ethernet cables out of your window to your neighbors' houses.

### ANSI Diagram: The Naive ISP

```
        [ The Global Internet ]
                  |
         (Public IP: 8.8.4.4)
     +--------------------------+
     |   Your PC (Naive ISP)    |
     |   DHCP Server: 1.1.1.1   |
     +--------------------------+
                  |  (Ethernet Cables out the window)
      +-----------+-----------+----------------+
      |                       |                |
(Cable: 30 meters)    (Cable: 150 meters!) (Cable: 10m)
      |                       |                |
 [Neighbor A]           [Neighbor B]     [Neighbor C]
 IP: 1.1.1.2            IP: 1.1.1.3      IP: 1.1.1.4

```

### Why is this a bad idea?

1.  **Distance Limitations:** Standard Ethernet cables (Cat5e/Cat6) lose their signal after **100 meters**. Neighbor B won't get any connection.
    
2.  **Security & Privacy:** Because everyone is plugged into the same simple switch/DHCP pool, they are all on the **same local network (Broadcast Domain)**. Neighbor A can directly ping, scan, or even hack Neighbor C.
    
3.  **Scalability:** You can't run millions of cables from one room.
    

## 3. The Real ISP Architecture

To solve the problems of distance and security, real ISPs use a multi-tiered architecture utilizing phone lines, fiber optics, and advanced routing.

### ANSI Diagram: The Real ISP

```
               [ The Global Internet ]
                         |
+---------------------------------------------------+
|                  ISP DATACENTER                   |
|                                                   |
|  +---------------------------------------------+  |
|  |                EDGE ROUTER                  |  | <-- Connects ISP to the World
|  +---------------------------------------------+  |
|                         |                         |
|  +---------------------------------------------+  |
|  |                 FIREWALL                    |  | <-- Blocks attacks, filters traffic
|  +---------------------------------------------+  |
|                         |                         |
|  +---------------------------------------------+  |
|  |           CENTRAL ROUTING CORE              |  | <-- Manages traffic across the city
|  +---------------------------------------------+  |
+---------------------------------------------------+
                         |
          (Fiber Optic cables across the city)
                         |
+---------------------------------------------------+
|          TELECOM CENTER / DSLAM (Local)           | <-- Neighborhood distribution point
+---------------------------------------------------+
                         |
           (Telephone Line / Copper / Fiber)
                         |
+---------------------------------------------------+
|               HOME MODEM / CPE                    | <-- Your House
+---------------------------------------------------+

```

### Definitions for the Architecture:

-   **Edge Router:** The massive, powerful router at the very edge of the ISP's network. It is the gateway between the ISP and the rest of the world. It holds maps (routes) to the entire global internet.
    
-   **Firewall:** Protects the ISP's internal network from global cyberattacks and filters restricted IPs.
    
-   **Telecom Center:** Converts the ISP's massive digital data into signals that can travel over your home's telephone line (DSL) or local fiber network.
    
-   **Hop:** Every time data jumps from one router to the next (e.g., Modem -> Telecom Center -> Central Core -> Edge Router), it is called one **Hop**.
    

## 4. IP Allocation and CGNAT

Because IPv4 addresses are exhausted and expensive, ISPs do not give every single customer a Public IP. Instead, they use a tiered system:

1.  **Your Home Network (LAN):** Your home router hands out private IPs to your phone and laptop (e.g., `192.168.1.10`).
    
2.  **The Modem's WAN IP:** The ISP gives your modem a "Carrier-Grade Private IP" (e.g., `10.1.1.100`).
    
3.  **The Edge Router:** The ISP's Edge Router finally translates all of its customers' `10.x.x.x` IPs into a few shared **Public IPs** to talk to the global internet.
    

## 5. CPE (Customer-Premises Equipment)

**CPE** is any networking equipment located at the customer's site (your house or office).

-   Examples: Your ADSL Modem, a MikroTik router, or an ONT (Fiber box).
    
-   **Ports:** CPEs typically have a **WAN port** (Wide Area Network - connects to the ISP) and multiple **LAN ports** (Local Area Network - connects to your personal devices).
    
-   They automatically manage the complex routing between your safe internal network and the outside world.
    

## 6. Dynamic Routing Protocols: RIP

In previous sessions, we typed `ip route add...` manually. This is **Static Routing**. But what if a cable breaks? The network goes down. **Dynamic Routing Protocols** allow routers to talk to each other, share maps, and automatically find the best path.

One of the oldest protocols is **RIP (Routing Information Protocol)**.

### How RIP Works

-   Every router has a table mapping IPs to specific ports.
    
-   In **RIP version 1**, a router shouts its _entire_ routing table out of all its ports every **30 seconds**.
    
-   Neighboring routers hear this, learn new routes, and update their own tables.
    
-   **Metric:** RIP uses **Hop Count** to find the best path. If Path A takes 2 hops to reach Google, and Path B takes 5 hops, RIP chooses Path A.
    

### The Limitations of RIP

1.  **Network Congestion:** Broadcasting the entire table every 30 seconds creates a massive amount of "chatter" (noise) on the network, wasting bandwidth.
    
2.  **The 15-Hop Limit:** RIP was designed for small networks. It can only track up to 15 hops. If a destination is **16 hops** away, RIP considers it "Infinite" or "Unreachable." The internet is much larger than 15 hops!
    

### Security Issues with RIPv1

RIPv1 is inherently insecure because it operates on **blind trust**:

-   **No Authentication:** RIPv1 does not use passwords.
    
-   **Route Poisoning / Blackholing:** A malicious user can plug a laptop into the network, run a routing software, and broadcast: _"Hey everyone! The fastest way to reach the Central Server is through ME!"_
    
-   Because RIPv1 has no authentication, the other routers will believe the hacker. All traffic meant for the server will be sent to the hacker's laptop, allowing them to steal data (Man-in-the-Middle attack) or simply drop the data, taking the network offline (Blackhole attack).
