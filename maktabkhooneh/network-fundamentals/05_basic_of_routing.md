# Session 05: Introduction to Routing (Part 1)

## 1. The Routing Problem

In previous sessions, we connected devices on the _same_ network (e.g., `192.168.1.x`). This is easy because they are "neighbors."

**But what if we want to talk to a stranger?** Imagine you are on network `10.1.1.0/24` and you want to ping `5.5.5.1`.

### The Device's Logic

Before sending a single bit of data, your computer asks a critical question:

1.  **"Is the destination IP in my subnet?"**
    
    -   **YES:** Great! I'll broadcast ARP, find their MAC, and send it directly.
        
    -   **NO:** Uh oh. I need a map (Route) to find where to send this.
        

If your computer checks its map (Routing Table) and finds **no path** to `5.5.5.1`, it won't even try. It gives up immediately with "Network Unreachable."

---

## 2. The Scenario Setup

Let's simulate two isolated networks connected by a wire (or virtual switch), but configured with different subnets.

-   **PC A (Source):**
    
    -   IP: `10.1.1.10/24`
        
    -   Interface: `eth0`
        
-   **PC B (Destination):**
    
    -   IP: `5.5.5.1/24`
        
    -   Interface: `eth0`
        

**Goal:** Ping from PC A to PC B.

---

## 3. manipulating the Routing Table (Linux)

### Viewing the Table

To see the current map, use:

```
ip route

```

**Output (PC A):**

```
10.1.1.0/24 dev eth0 proto kernel scope link src 10.1.1.10

```

_Translation:_ "I only know about the `10.1.1.0` network because I am physically connected to it."

### Adding a Route (The Outbound Path)

We need to teach PC A how to reach `5.5.5.0`.

```
# Syntax: ip route add {target_network} dev {exit_interface}
sudo ip route add 5.5.5.0/24 dev eth0

```

_Translation:_ "Hey PC A, if you want to talk to anyone in `5.5.5.x`, just throw the packet out of interface `eth0`."

## 4. The "One-Way" Trap (Why Ping Fails)

You added the route on PC A. You type `ping 5.5.5.1`. **Result:** No response. (Request Timed Out).

**WHY?** Routing is a **Two-Way Street**.

1.  **PC A** sends the packet. It knows the way. -> **Packet arrives at PC B.**
    
2.  **PC B** gets the packet ("Hello!").
    
3.  **PC B** tries to reply to `10.1.1.10`.
    
4.  **PC B** checks its routing table: "Do I know `10.1.1.10`? No. Do I have a Gateway? No."
    
5.  **PC B drops the reply.**
    
---

### 🔬 Wireshark Analysis

If you look at Wireshark on PC B, you will see a tragedy:

|No. | Protocol | Source | Destination | Info|
|--- | --- | --- | --- | --- | 
|1 |**ICMP** | `10.1.1.10` | `5.5.5.1` | **Echo (ping) request**|
|... | ... | ... |... |_...Silence..._|

-   **Analysis:** The _Request_ made it! The valid green line proves the outbound route works. The silence proves the **Return Path** is broken.
    
--- 

## 5. The Solution: The Return Route

We must go to **PC B** and teach it how to reach PC A's network.

```
# On PC B (5.5.5.1)
sudo ip route add 10.1.1.0/24 dev eth0

```

Now, when PC B tries to reply to `10.1.1.10`, it looks at its table, sees the new rule, and sends the reply out of `eth0`.

---

## 6. Visualizing the Path

Here is the packet flow before and after fixing the route.

### ❌ Before (Broken Return Path)

```
      PC A (10.1.1.10)                        PC B (5.5.5.1)
      +--------------+                        +--------------+
      |              |                        |              |
(1)   | Ping Request | ---------------------> | Received!    |
      | (Has Route)  |                        |              |
      |              |                        | (2) Wants to reply to 10.x
      |              |                        | "I don't know 10.x!"
      |   (Waiting)  |      X <-------------- | *DROP* |
      +--------------+                        +--------------+

```

### ✅ After (Symmetric Routing)

```
      PC A (10.1.1.10)                        PC B (5.5.5.1)
      +--------------+                        +--------------+
      |              |                        |              |
(1)   | Ping Request | ---------------------> | Received!    |
      |              |                        |              |
      |              |                        | (2) Reply to 10.x
      |              |                        | "Ah, I have a route for 10.x!"
(3)   | Ping Reply   | <--------------------- | Sends Reply  |
      | (Success!)   |                        |              |
      +--------------+                        +--------------+

```

--- 

## 7. Key Takeaways

1.  **Routes are Local:** Adding a route on one computer does not magically tell the other computer the path. You must configure **both sides** (or use a Gateway).
    
2.  **Ping is a Round Trip:** For a ping to succeed, you need a path from A -> B **AND** a path from B -> A.
    
3.  **Default Gateway:** In real networks (like the Internet), we don't add 1000 manual routes. We add one "Default Gateway" (0.0.0.0/0) that says "For everything else, send it to the Router." (We will cover this next session).
