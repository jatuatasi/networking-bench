# Networking Basics: Following One Request

Study notes · October 17, 2026

## How to use these notes

Networking is not 20 separate things to memorize. It is one request taking a trip, and each topic solves one problem along the way.

These notes follow the Cloud X Berry video [I Bet Nobody Has Ever Explained the Networking Topics Like This Before](https://www.youtube.com/watch?v=TR0RU2pmMeA) (Oct 13, 2026). The story: a passenger opens a rideshare app called RideNow and taps **Request Ride**. We follow that tap from the phone, through the home router, across the internet, and into RideNow's servers.

![One ride request, start to finish: phone and home router, internet routers, then RideNow's firewall, load balancer, reverse proxy, app server and database](journey.svg)

Want to click through it step by step? Open [journey.html](journey.html).

Each topic has the same parts:

- **What's really going on** – the plain-English explanation
- **Analogy** – an everyday comparison
- **In the rideshare story** – where it fits in the trip
- **See it on your computer** – a Windows command to try
- **When it goes wrong** – what a help desk person would see
- **A+ tip** – what tends to show up on the exam

**To try the commands:** click Start, type **cmd**, and open **Command Prompt**. For commands marked PowerShell, type **powershell** instead.

## Part 1: Inside the passenger's home

### Step 1: IP address

**What's really going on:** Every device needs an address so data can reach it and replies can come back. An **IPv4** address is four numbers from 0 to 255 with dots, like `192.168.1.15`. There are only about 4 billion of them, which isn't enough. An **IPv6** address is longer, uses letters and numbers with colons, like `2001:db8::1`, and the supply basically never runs out. An IP isn't permanent: it means "where you can be reached right now."

**Analogy:** Your home mailing address. It changes if you move, like your phone getting a new IP on cell data.

**In the rideshare story:** The phone needs an address so RideNow's reply can find its way back.

**See it on your computer:** `ipconfig` – look for IPv4 Address, Subnet Mask and Default Gateway (your router).

**When it goes wrong:** An address starting with **169.254** means the computer couldn't get an address from the router and made one up. No internet.

**A+ tip:** Know what IPv4 and IPv6 look like. `169.254.x.x` = **APIPA** = automatic addressing failed.

### Step 2: Private vs public IP

**What's really going on:** Some address ranges are private, reused in every home and office, never used on the open internet:

- `10.x.x.x`
- `172.16.x.x` through `172.31.x.x`
- `192.168.x.x`

Everything else (with a few exceptions) is public. Your internet company gives your home **one public IP** that every device shares. Special case: `127.0.0.1` (**localhost** or **loopback**) always means "this same computer."

**Analogy:** "Apartment 12" only makes sense inside the building. The post office needs the building's street address.

**In the rideshare story:** The phone has a private IP. RideNow's public parts can be reached from the internet; its database stays private.

**See it on your computer:** `ipconfig` shows your private IP. A site like whatismyip.com shows your public IP. They're different.

**When it goes wrong:** Giving a friend `192.168.1.20` to join your game server won't work; it's private. You'd need your public IP plus **port forwarding** on the router.

**A+ tip:** Memorize the three private ranges and that `127.0.0.1` = loopback.

### Step 3: DNS (names into numbers)

**What's really going on:** The app knows a name like `api.ridenow.com`, but data needs a number. DNS finds it:

1. Check the computer's own saved answers. If found and not expired, done.
2. Ask the DNS server it was given (often the router, the internet company's server, or a public one like `8.8.8.8`).
3. If that server doesn't know, it asks a **root server** ("who handles .com?"), then the **.com server** ("who handles ridenow.com?"), then **RideNow's own DNS server**.
4. The answer is saved for a set time called the **TTL** (time to live).

Record types: **A** (name to IPv4), **AAAA** (name to IPv6), **CNAME** (nickname pointing to another name), **MX** (which server handles email).

**Analogy:** Your phone's contacts list. Tap "Mom" and it finds her number.

**In the rideshare story:** The app looks up RideNow's IP from its name.

**See it on your computer:** `nslookup google.com` · `ipconfig /displaydns` (saved answers) · `ipconfig /flushdns` (clear them, a common fix).

**When it goes wrong:** User can't open websites by name, but `ping 8.8.8.8` works. Internet is fine, DNS is the problem.

**A+ tip:** DNS uses **port 53**. Know A, AAAA, CNAME, MX. "Reaches an IP but not a name" = DNS issue.

### Step 4: Ports

**What's really going on:** The IP gets data to the right **machine**. The **port** gets it to the right **program** on that machine. Ports run 0 to 65535; 0–1023 are **well-known ports** for common services. Both sides use a port: the server listens on a known one (443), your device picks a random high one (like 52317) for its side. IP + port together is a **socket**, like `203.0.113.5:443`.

**Analogy:** The IP is the building, the port is the apartment number.

**In the rideshare story:** The request is aimed at the ride service, not some other program on the server.

**See it on your computer:** `netstat -an` – the number after the colon is the port. You'll see lots of `:443`.

**When it goes wrong:** A program is running but unreachable. It may only be listening on `127.0.0.1` (localhost), or a firewall is blocking the port. Running is not the same as reachable.

**A+ tip:** The port numbers are heavily tested (see the cheat sheet at the end).

### Step 5: MAC address

**What's really going on:** Every network card has a **MAC address** set at the factory: 12 characters using 0–9 and A–F, like `3C-52-82-1A-2B-9F`. The first half names the maker; the second half is that card's own number. IP gets data **across networks**. MAC delivers it **inside one local network**. Switches keep a list of which MAC is on which port.

**Analogy:** IP is the street address; MAC is the name on the mailbox inside the building.

**In the rideshare story:** To hand data to the home router over Wi-Fi, the phone needs the router's MAC.

**See it on your computer:** `ipconfig /all` (look for Physical Address) or `getmac`.

**When it goes wrong:** Some offices use **MAC filtering**, so a new laptop can't connect until IT approves its MAC. Newer phones use a random MAC for privacy, which can confuse that.

**A+ tip:** MAC = hardware address, 48 bits, 12 hex characters. Switches use MAC; routers use IP.

### Step 6: ARP (matching an IP to a MAC)

**What's really going on:** RideNow isn't on your home network, so the phone doesn't need RideNow's MAC. It needs the **router's** MAC, because the router is the exit door.

1. The phone checks its ARP list for the router's MAC.
2. If missing, it **broadcasts** to the whole home network: "Who has 192.168.1.1?"
3. Only the router answers with its MAC.
4. The phone saves the answer and sends the data.

**Analogy:** Calling out in a classroom, "Who's in seat 14?" and someone raises their hand.

**In the rideshare story:** The phone finds the router's MAC so it can send the ride request out.

**See it on your computer:** `arp -a` – lists IPs on your network with their MACs.

**When it goes wrong:** An **IP conflict** (two devices with the same IP) makes things flaky. **ARP spoofing** is an attacker falsely claiming to be the router (a Security+ topic).

**A+ tip:** ARP turns an **IP into a MAC**, local network only. For anything outside, you ARP for the router's MAC.

### Part 1 recap

The phone already has a private IP, uses DNS to find RideNow's IP, aims at port 443, sees RideNow is outside the house, uses ARP to find the router's MAC, and sends the request over Wi-Fi to the router.

## Part 2: Leaving the house and crossing the internet

### Step 7: Routing

**What's really going on:** No single device knows the whole path to RideNow. Each router has a **routing table**: "traffic for this range of addresses goes to that next router." Each handoff is a **hop**; a cross-country trip may be 10 to 20 hops. Your home router mostly has one rule, the **default route**: anything not in this house goes to the internet company. Each packet also carries a counter that drops by one per hop and is thrown away at zero, so lost data can't loop forever. (This counter is also called TTL, but it's not the DNS TTL.)

**Analogy:** Driving cross-country using only road signs. At each junction you just follow the sign to the next turn.

**In the rideshare story:** The request hops across several networks to reach RideNow.

**See it on your computer:** `tracert google.com` (every hop and its time) · `route print` (your routing table) · `ping google.com` (can you reach it, how fast).

**When it goes wrong:** In `tracert`, a big jump in time or `* * *` from some hop onward points to where the slowdown or break is. The website itself may be fine. (Some routers just don't answer tracert, so a few stars aren't always a problem.)

**A+ tip:** Routers connect **different networks** using **IP**. Switches connect devices **within one network** using **MAC**. Know `ping` and `tracert`.

### Step 8: NAT (Network Address Translation)

**What's really going on:** Private IPs aren't allowed on the internet, so as the request leaves, the router:

1. Replaces the phone's **private IP** with the house's **public IP**.
2. Often swaps the port for one of its own.
3. Writes it in a **NAT table**: "outside port 40001 = the phone at 192.168.1.15, port 52317."
4. When the reply arrives on port 40001, it hands it to the phone.

The port-swapping version is technically **PAT** (Port Address Translation), and nearly every home router does it. Side benefit: outsiders can't start a conversation with your devices, since only replies to things you started have a table entry.

**Analogy:** An office receptionist. All calls go out on the main number, and callbacks get transferred to the right desk.

**In the rideshare story:** The home router sends the request out under the house's public IP and passes RideNow's reply back to the phone.

**See it on your computer:** Compare `ipconfig` (private IP) with whatismyip.com (public IP). NAT sits between them.

**When it goes wrong:** Hosting a game server or reaching a home camera from outside won't work until you set **port forwarding**: a router rule that sends traffic on a certain port to one inside device.

**A+ tip:** NAT lets private addresses reach the internet through one public IP. Port forwarding lets outside traffic reach a specific inside device.

## Part 3: How networks get organized and set up

### Step 9: Subnetting

**What's really going on:** An IP address has a **network part** (which neighborhood) and a **host part** (which house). The **subnet mask** shows where one ends and the other begins. The common home mask is `255.255.255.0`: the 255s are the network part, the 0 is the host part.

Example: `192.168.1.15` with mask `255.255.255.0` → network `192.168.1`, host `15`. Everything from `192.168.1.1` to `192.168.1.254` is the same neighborhood (254 usable addresses; the first and last are reserved). Shorthand: `192.168.1.0/24`, called **CIDR notation**. `/24` = `255.255.255.0`.

Every time a device sends something, it uses the mask to decide:

- **Same neighborhood?** Send directly (ARP for that device's MAC).
- **Different neighborhood?** Send to the default gateway (the router).

Companies split networks into many subnets so they can put rules between sections.

**Analogy:** An office building with keycard floors. Visitors get the lobby, staff get their floor, only a few get the server room.

**In the rideshare story:** RideNow keeps public-facing parts, app servers, and the database in separate sections. Only app servers can reach the database.

**See it on your computer:** `ipconfig` – look at Subnet Mask (likely `255.255.255.0`).

**When it goes wrong:** A wrong mask or gateway typed in by hand makes a computer reach some devices but not others, or no internet at all.

**A+ tip:** `255.255.255.0` = `/24`. Know what a subnet mask and a default gateway do. Deep subnet math is Network+ and CCNA territory.

### Step 10: DHCP (automatic network settings)

**What's really going on:** A device joining a network needs an **IP address**, **subnet mask**, **default gateway** and **DNS server**. DHCP hands them out in four messages, **DORA**:

1. **Discover** – device shouts, "Any DHCP server out there?"
2. **Offer** – server: "You can have 192.168.1.15."
3. **Request** – device: "I'll take it."
4. **Acknowledge** – server: "Confirmed."

The address is a **lease** with an expiration time, renewed before it runs out. In offices the server has a **scope** (the range it hands out) and **reservations** (a printer always gets the same address).

**Analogy:** Hotel check-in. You get a room, a key card that expires at checkout, and directions to the exits.

**In the rideshare story:** The phone got its settings from the home router when it joined Wi-Fi. RideNow's new servers get theirs automatically too.

**See it on your computer:** `ipconfig /all` (DHCP Enabled, DHCP Server, Lease Obtained, Lease Expires) · `ipconfig /release` then `ipconfig /renew` (one of the most common help desk fixes).

**When it goes wrong:** If nobody answers Discover, the device gives itself a `169.254.x.x` address (APIPA). Causes: unplugged cable, weak Wi-Fi, DHCP server down, or the scope ran out of addresses.

**A+ tip:** DORA, leases, ports **67 (server) and 68 (client)**, and `169.254` = DHCP failed.

## Part 4: Making the connection

### Step 11: TCP

**What's really going on:** Data travels in small pieces (**segments** inside **packets**) that can take different paths and arrive out of order. TCP makes sure the receiver gets everything, put together correctly:

- **Numbering** – every piece gets a sequence number so it can be reordered.
- **Receipts** – the receiver sends **ACKs**: "got everything up to piece 500."
- **Resending** – no receipt in time, the piece is sent again.
- **Flow control** – the receiver says how much it can take at once (**windowing**).
- **Congestion control** – lots of lost pieces means a jammed network, so TCP slows down.

**Analogy:** Shipping a 20-box furniture set with tracking. Every box numbered, signed for, and missing boxes resent.

**In the rideshare story:** A ride request must arrive complete, so it uses TCP.

**See it on your computer:** `netstat -an` – TCP connections marked **ESTABLISHED** are active.

**When it goes wrong:** Weak Wi-Fi loses pieces, TCP keeps resending, and the user just sees **everything getting slow**.

**A+ tip:** TCP = **connection-based, reliable, ordered**. Used for web, email, file transfer and remote access.

### Step 12: The three-way handshake

**What's really going on:** Before real data moves, TCP sets up the connection:

1. **SYN** – phone: "I'd like to connect. My starting number is X."
2. **SYN-ACK** – server: "Got it. My starting number is Y."
3. **ACK** – phone: "Got yours. We're connected."

When finished, each side sends **FIN** to close politely. Opening a connection takes a moment, so apps reuse connections instead of opening a new one for every small request.

**Analogy:** "Can you hear me?" "Yes, can you hear me?" "Yep." Then the real conversation. At the end, "Bye." "Bye."

**In the rideshare story:** Phone and server greet each other before sending the ride request.

**See it on your computer (PowerShell):** `Test-NetConnection google.com -Port 443` – **TcpTestSucceeded: True** means the handshake worked.

**When it goes wrong (very useful):**

- **"Connection timed out"** – SYN went out and **nothing came back**. Usually a firewall silently dropping it, the server being off, or a routing problem.
- **"Connection refused"** – the server **answered but said no**. The machine is reachable, but **nothing is listening on that port** (app stopped or on a different port).

**A+ tip:** SYN, SYN-ACK, ACK. Timed out = no response. Refused = reached the machine, nothing listening.

### Step 13: HTTPS and TLS

**What's really going on:** Right after the TCP handshake comes the TLS handshake:

1. Phone: "I want a secure connection. Here's the encryption I support."
2. Server sends its **certificate**, a digital ID card.
3. Phone checks it: issued by a trusted **Certificate Authority**? Name matches the site? Still in date?
4. If it checks out, both agree on a secret key and everything after is scrambled.

TLS gives three protections: **encryption** (nobody can read it), **integrity** (changes are detected), **authentication** (it's really RideNow). **HTTPS** is web traffic inside TLS on **port 443**. Plain **HTTP** is **port 80** and not encrypted. In big setups the reverse proxy often unlocks the encryption (**TLS termination**) instead of every app server doing it.

**Analogy:** A locked box with a tamper seal, handed only to someone who shows official ID. Not a postcard.

**In the rideshare story:** Location, account and payment info stay private on the way there.

**See it on your computer:** Click the **padlock** next to a web address and view the certificate: who it's for, who issued it, when it expires.

**When it goes wrong:** "Your connection is not private" on **lots** of sites at once → check the computer's **date and time**. A wrong clock makes every certificate look invalid. On **one** site only → that site's certificate may really be expired.

**A+ tip:** HTTPS = 443, HTTP = 80. Certificates prove identity. Wrong system time causes certificate errors.

### Step 14: HTTP (the actual request)

**What's really going on:** An HTTP request has a **method**, an **address** (like `/rides`), **headers** (extra info such as your login token) and a **body** (the data, like pickup and drop-off).

- **GET** – give me something
- **POST** – here's something new (request a ride)
- **PUT** – update this
- **DELETE** – remove this

The server replies with a **status code**:

| Code | Meaning |
| --- | --- |
| 200 | OK, it worked |
| 301 | Moved, go to this other address |
| 401 | You need to log in |
| 403 | Forbidden, not allowed |
| 404 | Not found |
| 500 | The server itself broke |
| 502 | Bad gateway (proxy got a bad answer from the server behind it) |
| 503 | Server too busy or down for maintenance |
| 504 | Gateway timeout (server behind the proxy didn't answer) |

Rule of thumb: **400s = problem with the request, 500s = problem on the server.**

**Analogy:** Ordering at a restaurant: what you want, which item, special notes, details. The waiter brings food or says "we're out of that" (404).

**In the rideshare story:** The app sends a POST saying "I need a ride from here to there." RideNow checks your account, finds a driver, and replies 200 OK.

**See it on your computer:** In Chrome or Edge press **F12** → **Network** tab → reload. You'll see each request's method and status code.

**When it goes wrong:** 404 = bad link or removed page. 403 = permissions. 500/503 = the website's side, not the user's computer.

**A+ tip:** HTTP is the web's request/response language. Recognize the common status codes, especially 404.

## Part 5: Inside RideNow's front door

### Step 15: Load balancer

**What's really going on:** One server would collapse under millions of riders. RideNow runs many identical servers with a load balancer in front that picks one for each request:

- **Round robin** – take turns: 1, 2, 3, 1, 2, 3...
- **Least connections** – send to the least busy server.
- **Sticky sessions** – keep a user on the same server when that matters.

It runs **health checks** on each server. If one stops answering, it gets no traffic until it recovers. Two benefits: **scalability** (add servers as traffic grows) and **availability** (stay up when one fails).

**Analogy:** A restaurant host spreading guests across waiters, skipping any waiter who went home sick. Or the Costco worker pointing you to an open lane.

**In the rideshare story:** Millions of ride requests get spread across many app servers.

**When it goes wrong:** A problem that happens **sometimes but not every time** (refresh works, refresh again fails) can mean one server behind the load balancer is broken.

**A+ tip:** Light coverage. Know it spreads traffic for scalability and availability.

### Step 16: Reverse proxy

**What's really going on:** The company's smart front door. Every outside request hits it first. Its jobs:

- **Routing** – `/rides` to the ride service, `/payments` to payments, `/drivers` to drivers.
- **TLS termination** – unlocks the encryption so app servers don't have to.
- **Caching** – keeps copies of common answers to reply instantly.
- **Compression** – shrinks responses so they travel faster.
- **Hiding the inside** – outsiders only see the proxy.

**Forward vs reverse proxy:** A **forward proxy** sits in front of **users** (a company proxy that filters employees' web browsing). A **reverse proxy** sits in front of **servers**. Load balancers and reverse proxies overlap and are often used together.

**Analogy:** A hotel concierge. Guests ask for everything there and never need to know who handles it behind the scenes.

**In the rideshare story:** It sends the ride request to the ride service, not the driver or payment service.

**When it goes wrong:** **502 Bad Gateway** or **504 Gateway Timeout** usually means the proxy is fine but the server behind it didn't answer properly.

**A+ tip:** A+ covers **proxy servers** generally. Know a proxy is a middleman that can filter, cache and hide traffic.

### Step 17: Firewall

**What's really going on:** A firewall checks traffic against **rules** based on source IP, destination IP, port, and TCP or UDP, then **allows** or **denies** it. Rules are checked top to bottom; anything not allowed is blocked by default (**implicit deny**). Most firewalls are **stateful**: they remember conversations you started and let the replies back in.

- **Network firewall** – a device protecting a whole network.
- **Host-based firewall** – software on one computer, like **Windows Defender Firewall**.

RideNow's rules might be:

| From | To | Port | Action |
| --- | --- | --- | --- |
| Internet | Front door | 443 | Allow |
| App servers | Database | Database port | Allow |
| Internet | Database | Any | Deny |

**Analogy:** A gate guard with a guest list. Not on the list, not getting in.

**In the rideshare story:** Riders can reach the public app, never the database directly.

**See it on your computer:** Type `wf.msc` in Start to open Windows Defender Firewall with Advanced Security (inbound and outbound rules).

**When it goes wrong:** Firewalls usually block **silently**, so the user sees a **timeout**. If the server is definitely up, check firewalls on both ends. In PowerShell: `Test-NetConnection server -Port number`.

**A+ tip:** Host-based vs network firewalls, allowing an app through Windows Defender Firewall, filtering by IP, port and protocol.

## Part 6: Live updates and speed

### Step 18: WebSockets

**What's really going on:** Regular HTTP only lets the phone ask and the server answer; the server can't speak first. The old workaround is **polling**: asking "any update?" every few seconds, which is wasteful and a bit late. A WebSocket:

1. Starts as a normal HTTP request asking to **upgrade** to a WebSocket.
2. The server agrees.
3. The connection stays open and **either side can send anytime**.

Secure WebSockets start with `wss://` (like `https://`) and use port 443, so they pass most firewalls.

**Analogy:** Texting "Where are you now?" every 30 seconds vs. staying on a call where the driver tells you as they go.

**In the rideshare story:** Once a driver accepts, RideNow pushes each location update to the passenger's map.

**When it goes wrong:** Strict company firewalls or proxies can cut long-lasting connections. Live chat or tracking freezes or keeps reconnecting while normal pages load fine.

**A+ tip:** Not a major A+ topic. Good general knowledge for web and developer roles.

### Step 19: UDP

**What's really going on:** UDP is TCP's opposite: **no handshake, no receipts, no resending, no ordering.** That makes it **fast and light**. For live things, waiting to resend old data makes it worse; a tiny glitch beats a frozen call.

| Service | Port | Why UDP |
| --- | --- | --- |
| DNS | 53 | Quick little questions |
| DHCP | 67/68 | Device has no address yet |
| TFTP | 69 | Simple file transfer |
| SNMP | 161 | Network device monitoring |
| Voice/video calls, gaming, live streaming | Varies | Live beats perfect |

**TCP or UDP?** Need every piece → **TCP**. Need it fast and live → **UDP**. Neither is better; they solve different problems.

**Analogy:** TCP is certified mail. UDP is a live radio broadcast; if you miss a word, the DJ doesn't repeat it.

**In the rideshare story:** The ride request stays on TCP. A voice call with the driver would likely use UDP.

**When it goes wrong:** Choppy calls, robotic voices, or game lag point to lost UDP packets, often weak Wi-Fi or an overloaded connection.

**A+ tip:** Very commonly tested. **TCP = reliable, connection-based. UDP = fast, connectionless.** Know which services use which.

## Part 7: The OSI model and troubleshooting

### Step 20: The OSI model

**What's really going on:** The OSI model splits networking into seven layers. Each has its own job and relies on the one below it.

| # | Layer | What it handles | Examples from the trip |
| --- | --- | --- | --- |
| 7 | Application | What apps use | HTTP, HTTPS, DNS, DHCP |
| 6 | Presentation | Formatting and encryption | TLS encryption |
| 5 | Session | Keeping conversations going | WebSocket staying open |
| 4 | Transport | Delivery method and ports | TCP, UDP, ports |
| 3 | Network | Addresses across networks | IP addresses, routers |
| 2 | Data Link | Delivery inside a local network | MAC addresses, switches, ARP |
| 1 | Physical | Cables and signals | Ethernet cables, Wi-Fi radio |

**Memory tricks:** bottom to top, **P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way. Top to bottom, **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing. (The **TCP/IP model** squeezes this into four layers; A+ focuses on OSI.)

**Analogy:** Fixing a lamp. Is it plugged in? Is the outlet working? Is the switch on? Is the bulb good? Start at the basics and work up.

**In the rideshare story:** When RideNow can't connect to something, check each layer instead of saying "the network is broken."

### Troubleshooting from the bottom up

| Layer | What to check | How |
| --- | --- | --- |
| 1 Physical | Plugged in? Wi-Fi connected? Lights on? | Look at it |
| 2 Data Link | On the right network? | `arp -a`, check Wi-Fi network name |
| 3 Network | Good IP? Reach the router and beyond? | `ipconfig`, `ping` the gateway, `ping 8.8.8.8`, `tracert` |
| 4 Transport | Connect to the right port? | `Test-NetConnection`, check firewall |
| 7 Application | Do names work? Does the site respond? | `nslookup`, browser, error codes |

**Real example: "I can't get to the internet."**

1. Cable plugged in or Wi-Fi connected? Yes → Layer 1 OK.
2. `ipconfig` shows `169.254.x.x`? → DHCP problem. Run `ipconfig /release` and `ipconfig /renew`.
3. Good IP, can `ping` the router? Can `ping 8.8.8.8`? Yes → Layer 3 OK.
4. Can't open google.com but `ping 8.8.8.8` works? → DNS problem. Run `ipconfig /flushdns`.

**A+ tip:** Know all seven layers in order, which devices and protocols belong to each (switch = 2, router = 3, TCP/UDP = 4), and the bottom-up approach.

## The full trip, start to finish

**At home**

1. When the phone joined Wi-Fi, **DHCP** gave it an IP address, subnet mask, gateway and DNS server (DORA).
2. Its address is **private** (`192.168.1.15`), so it only works inside the house.
3. The passenger taps Request Ride.
4. The phone uses **DNS** to turn `api.ridenow.com` into an IP address.
5. It aims for **port 443** (secure web).
6. The **subnet mask** shows RideNow is outside the home network, so the request goes to the **router**.
7. **ARP** finds the router's **MAC address**, and the data goes out over Wi-Fi.

**Leaving the house**

8. The router uses **NAT** to swap the private IP for the house's **public IP** and notes it so replies come back to the right phone.

**Across the internet**

9. **Routers** pass the data hop by hop toward RideNow.

**Connecting**

10. The **TCP three-way handshake**: SYN, SYN-ACK, ACK.
11. The **TLS handshake**: RideNow shows its certificate and they agree on encryption. Now it's **HTTPS**.
12. The phone sends an **HTTP POST**: "I need a ride from here to there."

**Inside RideNow**

13. The **firewall** checks its rules: port 443 from the internet is allowed.
14. The **load balancer** picks a healthy app server.
15. The **reverse proxy** unlocks the encryption and sends the request to the ride service.
16. The ride service checks the **database**, in a private **subnet** only app servers can reach.
17. A driver is found. The server replies **200 OK** with the driver's info.

**The trip back**

18. The response travels back through the proxy and across the internet's routers to the house's public IP.
19. The home router checks its **NAT table** and hands it to the right phone.
20. **TCP** makes sure every piece arrived in order.

**Live tracking**

21. A **WebSocket** stays open so RideNow can push the driver's location.
22. A voice call with the driver would likely use **UDP** for speed.

**When something breaks**

23. Use the **OSI model** from the bottom up: cable, IP, router, port, DNS, app.

## Cheat sheet

### Ports to memorize

| Port | Service | TCP or UDP |
| --- | --- | --- |
| 20/21 | FTP (file transfer) | TCP |
| 22 | SSH (secure remote command line) | TCP |
| 23 | Telnet (old, insecure remote access) | TCP |
| 25 | SMTP (sending email) | TCP |
| 53 | DNS | UDP (TCP for big transfers) |
| 67/68 | DHCP (server/client) | UDP |
| 69 | TFTP (simple file transfer) | UDP |
| 80 | HTTP (regular web) | TCP |
| 110 | POP3 (receiving email) | TCP |
| 143 | IMAP (receiving email) | TCP |
| 161 | SNMP (device monitoring) | UDP |
| 443 | HTTPS (secure web) | TCP |
| 445 | SMB (Windows file sharing) | TCP |
| 3389 | RDP (Remote Desktop) | TCP |

### Windows commands

| Command | What it shows or does |
| --- | --- |
| `ipconfig` | Your IP, subnet mask, default gateway |
| `ipconfig /all` | Adds MAC (Physical Address), DHCP and DNS server details |
| `ipconfig /release` + `/renew` | Drop and request a new IP from DHCP |
| `ipconfig /flushdns` | Clear saved DNS answers |
| `ipconfig /displaydns` | Show saved DNS answers |
| `nslookup name` | Look up a name's IP address |
| `ping address` | Can you reach it, and how fast |
| `tracert address` | Every hop on the way there |
| `route print` | Your computer's routing table |
| `arp -a` | IP-to-MAC list on your network |
| `getmac` | Your MAC address |
| `netstat -an` | Active connections and ports |
| `Test-NetConnection name -Port n` (PowerShell) | Can you connect to that port |
| `wf.msc` | Windows Defender Firewall rules |

### Quick clues

| You see | It usually means |
| --- | --- |
| IP starts with 169.254 | DHCP failed (APIPA) |
| Ping 8.8.8.8 works, names don't | DNS problem |
| Connection timed out | Firewall dropping it, server off, or routing |
| Connection refused | Machine reached, nothing listening on that port |
| Certificate errors on many sites | Wrong date/time on the computer |
| Works sometimes, fails sometimes | One server behind a load balancer may be broken |
| 404 / 403 / 500 | Not found / not allowed / server broke |
