# Session 02: Lab Setup, Switching, and Basic Protocols

## 1. The Essential Tool: PING

The most fundamental command for network troubleshooting is `ping`.

```
ping <ip_address>
# Example: ping 192.168.1.20

```

### How does it work? (ICMP)

Ping uses a protocol called **ICMP** (Internet Control Message Protocol).

> **🌊 Analogy:** Think of ICMP like a **Sonar** on a submarine. It sends a "Pulse" out. If it hits something, the pulse bounces back. It doesn't carry user data (like a file or a photo); it just carries status messages like "Are you there?" (Echo Request) and "Yes, I am" (Echo Reply).

## 2. Lab Topology (Setup)

In this session, we build a physical Local Area Network (LAN) using a Layer 2 Switch.

**The Setup:**

-   **Device A:** Laptop
    
    -   IP: `192.168.1.20`
        
    -   MAC: `AA:AA:AA:AA:AA:AA`
        
-   **Device B:** Raspberry Pi
    
    -   IP: `192.168.1.30`
        
    -   MAC: `BB:BB:BB:BB:BB:BB`
        
-   **Connector:** Ethernet Switch
    

### Network Diagram (Physical View)

```
+----------------+          +----------------+          +----------------+
|     Laptop     |          |    L2 Switch   |          |  Raspberry Pi  |
| IP: 192.168.1.20| <------> | [Port 1]      |          | IP: 192.168.1.30|
| MAC: AA:AA...  |          |      [Port 2] | <------> | MAC: BB:BB...  |
+----------------+          +----------------+          +----------------+
        |                           ^                           |
    Layer 3 (IP)               Layer 2 (Frames)             Layer 3 (IP)

```

### Layer Breakdown

-   **Layer 1 (Physical):** The Ethernet Cables connecting the devices.
    
-   **Layer 2 (Data Link):** The **Ethernet Protocol**.
    
    -   _What is Ethernet?_ It is the standard set of rules for wired connections. It defines how data is packaged into "Frames" and uses MAC addresses to find devices connected to the same cable.
        
-   **Layer 3 (Network):** The IP Addresses used to identify the devices logically.
    

## 3. Inspecting Network Interfaces

### Checking IP Configuration

To see your current IP and MAC address:

-   **Windows:** `ipconfig /all`
    
-   **Linux/Mac:** `ip addr` or `ifconfig`
    

**Example Output (Linux):**

```
2: eth0: <BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:56:91:35 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.20/24 brd 192.168.1.255 scope global eth0

```

### Key Concepts from Output

#### A. Loopback (127.0.0.1)

Often seen as interface `lo`.

> **Description:** This is a virtual interface that points back to **yourself**. If you `ping 127.0.0.1`, the data never leaves your computer. It is used to test if your network driver is working properly or for web developers testing a server on their own machine.

#### B. MAC Address (Physical Address)

Format: `MM:MM:MM:SS:SS:SS` (Example: `00:1A:2B:3C:4D:5E`)

-   **First Half (OUI):** Organizationally Unique Identifier. This tells you the **Manufacturer** (e.g., Apple, Dell, Cisco).
    
-   **Second Half:** Unique Serial Number for that specific card.
    
-   **Layer:** Layer 2.
    

## 4. Configuring IP on Linux

To manually assign an IP address to a specific interface (e.g., `eth0` or `enp0s3`):

```
# Syntax: ip addr add {IP}/{MASK} dev {INTERFACE_NAME}
sudo ip addr add 192.168.1.20/24 dev eth0

```

### What is the `/24`? (Subnet Mask)

It is a shorthand for `255.255.255.0`.

> **🏠 Analogy:** An IP address has two parts: The **Street Name** (Network) and the **House Number** (Host).
> 
> -   `/24` tells the computer: "The first 3 numbers (`192.168.1`) are the Street Name. The last number (`.20`) is the House Number."
>     
> -   Devices must be on the same "Street" to talk directly.
>     

## 5. The "Magic" of Switching (CAM Table)

**The Problem:** We ping an IP address (`192.168.1.30`), but Switches don't understand IPs. They only understand MAC addresses (Layer 2). How does the data get there?

**The Solution:** The **CAM Table** (Content Addressable Memory).

> **📘 Analogy:** The CAM Table is the Switch's **"Internal Phonebook."** It maps Physical Ports to MAC Addresses.
> 
> -   _Example:_ "Port 1 connects to Laptop (MAC A). Port 2 connects to Raspberry Pi (MAC B)."
>     

### How the Switch Learns (The Learning Process)

1.  **Learning:** When a frame enters a port, the switch looks at the **Source MAC**. It writes it down in the CAM table (e.g., "MAC A is on Port 1").
    
2.  **Flooding (Broadcast):** If the switch needs to send data to MAC B, but **doesn't** know where MAC B is, it shouts to everyone! It sends the packet out of **all ports** except the incoming one.
    
3.  **Forwarding:** Once MAC B replies, the switch learns "MAC B is on Port 2." Next time, it sends data _only_ to Port 2 (Unicast).
    

### Aging Timer

The switch doesn't remember forever. If a device is silent for a while (usually 300 seconds / 5 minutes), the switch deletes its entry from the CAM table. This keeps the memory clean.

### Switching Loops (Broadcast Storm)

If you connect switches in a circle (Switch A -> Switch B -> Switch C -> Switch A), a Broadcast packet will spin around forever.

> **🌪️ Analogy:** Imagine shouting "Hello" in a circular canyon with infinite echoes. The noise never stops. This crashes the network. (Protocols like STP prevent this).

## 6. The Missing Link: ARP

How does your Laptop know the Raspberry Pi's MAC address in the first place? You only typed the IP!

**Protocol:** **ARP** (Address Resolution Protocol). **Function:** Maps Layer 3 (IP) to Layer 2 (MAC).

> **🗣️ Analogy:** You are in a crowded room. You know the name "Ali" (IP), but you don't know his face (MAC).
> 
> 1.  **The Request (Broadcast):** You shout: **"Who here is Ali?"**
>     
> 2.  **The Reply (Unicast):** Ali stands up and says: **"I am Ali, here is my face."**
>     
> 3.  Now you can go talk to him directly.
>     

## 7. Packet Analysis (Wireshark Deep Dive)

**Wireshark** is a "Packet Sniffer." It allows us to capture traffic and see the raw data.

### Scenario: First Ping (Cold Start)

We run `ping 192.168.1.30` from the Laptop (`192.168.1.20`). The Laptop does NOT know the Pi's MAC address yet.

**The Packet Flow:**
|No. | Protocol | Source |  Destination| Info |
|--|--| -- | --| -- |
| 1 | **ARP** |Laptop (AA:AA) |Broadcast (FF:FF) | Who has 192.168.1.30? Tell 192.168.1.20|
| 2 | **ARP** | Pi (BB:BB)|Laptop (AA:AA) | 192.168.1.30 is at BB:BB:BB:BB:BB:BB|
| 3 | **ICMP** | 192.168.1.20| 192.168.1.30|Echo (ping) request |
| 4 | **ICMP** |192.168.1.30 | 192.168.1.20| Echo (ping) reply|



### 🔬 Inside the Packets (What Wireshark Shows)

#### Packet 1: ARP Request (The Question)

When you click on this packet, you see the following layers:

-   **Ethernet II Frame:**
    
    -   `Destination: Broadcast (ff:ff:ff:ff:ff:ff)` -> _Send to everyone!_
        
    -   `Source: Laptop (aa:aa:aa:aa:aa:aa)`
        
    -   `Type: ARP (0x0806)`
        
-   **Address Resolution Protocol (request):**
    
    -   `Opcode: request (1)`
        
    -   `Sender MAC: aa:aa:aa:aa:aa:aa`
        
    -   `Sender IP: 192.168.1.20`
        
    -   `Target MAC: 00:00:00:00:00:00` -> _I don't know this yet!_
        
    -   `Target IP: 192.168.1.30`
        

#### Packet 3: ICMP Echo Request (The Ping)

Now that ARP is done, the Ping is sent.

-   **Ethernet II Frame:**
    
    -   `Destination: Pi (bb:bb:bb:bb:bb:bb)` -> _Now we know the address!_
        
    -   `Source: Laptop (aa:aa:aa:aa:aa:aa)`
        
    -   `Type: IPv4 (0x0800)`
        
-   **Internet Protocol Version 4 (IP):**
    
    -   `Source: 192.168.1.20`
        
    -   `Destination: 192.168.1.30`
        
    -   `Protocol: ICMP (1)`
        
-   **Internet Control Message Protocol:**
    
    -   `Type: 8 (Echo (ping) request)`
        
    -   `Code: 0`
        
    -   `Data (32 bytes):` `61 62 63 64 65 66...` (ASCII for 'abcdef...')
