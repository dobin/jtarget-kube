# iTarget Cube Communication Protocol

> Reverse-engineered from the iTarget Pro Android app v1.5.1 (`com.iTargetPro.newcube`)  
> Decompiled with JADX. All findings below are from static analysis of the Java source.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Network Topology](#network-topology)
3. [Ports & Transports](#ports--transports)
4. [Message Format](#message-format)
5. [Discovery & Connection Flow](#discovery--connection-flow)
6. [TCP Connection Lifecycle](#tcp-connection-lifecycle)
7. [Game Protocol](#game-protocol)
8. [Message Reference](#message-reference)
9. [Timing & Reconnection](#timing--reconnection)
10. [Settings & Configuration](#settings--configuration)
11. [Authentication & Security](#authentication--security)
12. [WiFi Provisioning (Initial Setup)](#wifi-provisioning-initial-setup)
13. [Source Code Reference](#source-code-reference)
14. [Notes for Python Re-implementation](#notes-for-python-re-implementation)

---

## Architecture Overview

The system consists of:
- **Phone/App** (Android): Acts as the **server** for both UDP and TCP
- **iTarget Cubes**: WiFi-connected hardware targets that act as **clients**

The phone discovers cubes, commands them, and receives hit events. There is **zero authentication** — anyone on the same WiFi network can control any cube.

```
┌──────────────┐         WiFi (same subnet)         ┌──────────────┐
│   Phone/App  │◄──────────────────────────────────►│  iTarget Cube│
│              │                                     │              │
│  UDP :4587   │◄─── cube responses (IPGET,POWER) ──│  UDP :7042   │
│  (listen)    │                                     │  (listen)    │
│              │──── commands (INVITATION,ALREADY) ──►              │
│              │                                     │              │
│  TCP :4586   │◄─── cube connects as TCP client ───│              │
│  (server)    │──── game commands (FLASH,OFF,...) ──►              │
│              │◄─── events (HIT,POWER) ────────────│              │
└──────────────┘                                     └──────────────┘
```

---

## Network Topology

- All devices must be on the **same WiFi network** and **same /24 subnet**
- The app determines its own IP via `WifiManager.getConnectionInfo().getIpAddress()`
- Discovery scans IPs `x.x.x.2` through `x.x.x.254` (skipping its own IP and already-connected cube IPs)

---

## Ports & Transports

| Port | Transport | Direction | Purpose |
|------|-----------|-----------|---------|
| **7042** | UDP | App → Cube | Send commands to cubes (INVITATION, ALREADY, WIFI-INFO) |
| **4587** | UDP | Cube → App | App listens for cube broadcast responses (IPGET, POWER, FINDWIFI) |
| **4586** | TCP | Cube → App (cube initiates) | App runs TCP server; cubes connect after receiving ALREADY. Used for game commands and events |

### Port Details

- **UDP 7042**: The cube listens on this port. The app sends discovery and control packets here.
  - Note: The app dynamically updates `SOCKET_UDP_CUBE_PORT` from the source port of received UDP packets, but it's initialized to 7042.
- **UDP 4587**: The app binds a `DatagramSocket` on this port to receive cube responses.
- **TCP 4586**: The app creates a `ServerSocket` on this port. Cubes initiate TCP connections to the app after receiving the `ALREADY` message.

---

## Message Format

All messages are **UTF-8 encoded strings** using a simple comma-separated key=value format:

```
KEY1=VALUE1,KEY2=VALUE2,KEY3=VALUE3,
```

### Parsing Rules

The parser (`Tools.stringToMap()`) works as follows:

1. Strip `\r\n` or truncate at first `\r`
2. Split on `,`
3. For each token, split on `=`
4. Trim whitespace from key and value
5. Store in `HashMap<String, String>`

### Important Notes
- Messages typically end with a trailing comma (e.g., `TYPE=ALREADY,`)
- Values can be quoted in some cases (e.g., `IP="192.168.1.100"`)
- The `TYPE` field is always present and determines the message type
- Messages are sent as raw bytes via `str.getBytes()` (platform default encoding, effectively UTF-8)

---

## Discovery & Connection Flow

This is the complete flow assuming cubes are already on the correct WiFi network.

### Step 1: UDP Broadcast — INVITATION (App → Cubes)

The app broadcasts an INVITATION to every IP on the local /24 subnet (except its own IP and already-connected cubes):

```
DEVICE=CUBE,TYPE=INVITATION,IP=<app_local_ip>
```

**Destination**: Each IP individually, UDP port 7042  
**Source**: `UDPBuild.sendInvitation(context)`  
**Note**: Sends one UDP packet per IP with 5ms sleep between packets

### Step 2: Cube Response — IPGET (Cube → App)

Each cube that receives the INVITATION responds with:

```
TYPE=IPGET,ID=iTarget<digits>
```

**Destination**: App UDP port 4587  
**Fields**:
- `ID`: Cube identifier, format `iTarget` followed by 7+ digits (validated by regex `iTarget[0-9]{7}` or `iTarget[0-9]*`)

The app processes this in `ITargetHomeActivity.MyHandler` (message.what == 200):
- Creates a `Cube` object with the IP (from packet source) and ID (from message)
- Adds the cube to the game's cube list
- Saves to persistent storage (SharedPreferences via Gson serialization)
- Fires an EventBus event to refresh the UI

### Step 3: Targeted INVITATION (App → Cube)

After receiving IPGET, the app sends a targeted INVITATION back to the specific cube:

```
TYPE=INVITATION,IP="<app_local_ip>"
```

**Destination**: Cube IP, UDP port 7042  
**Note**: The IP value is **quoted** here (with double quotes), unlike the broadcast version. Sent 1× with 500ms sleep after.

### Step 4: ALREADY — Trigger TCP Connection (App → Cube)

The ALREADY message tells the cube to initiate a TCP connection to the app:

```
TYPE=ALREADY,
```

**Destination**: Cube IP, UDP port 7042  
**Source**: `UDPBuild.sendALREADY(ip)`

**Sending pattern (CONFIG_TYPE == 0 / AP config mode)**:
- After the INVITATION, sleep 1000ms
- Send ALREADY 4× with 500ms sleep between sends

**Sending pattern (game fragment, on connection)**:
- Send 5× per cube with 100ms sleep between sends (per cube, iterating all cubes)

**This is the critical message**: Without ALREADY, the cube will NOT connect via TCP.

### Step 5: TCP Connection (Cube → App)

Upon receiving ALREADY, the cube opens a TCP connection to the app on port 4586.

The app's `MySocketServer` accepts the connection:
```java
Socket socket = server.accept();
Constants.TCPClients.put(socket.getInetAddress().getHostAddress(), socket);
```

The TCP socket is stored in `Constants.TCPClients` (a `HashMap<String, Socket>` keyed by IP).

### Step 6: Heartbeat — POWER (Cube → App, via TCP)

Once connected, the cube sends periodic POWER messages over TCP:

```
TYPE=POWER,ID=<cube_id>,VALUE=<battery_level>
```

**Fields**:
- `ID`: Cube identifier (e.g., `iTarget1234567`)
- `VALUE`: Battery power level (numeric, e.g., `050` for 50%)

The app uses POWER messages to:
- Update battery level display
- Update the cube's IP (from TCP socket source)
- Reset the heartbeat timestamp (`cube.setHeartbeat()`)
- Mark the cube as connected (`cube.setConnected(true)`)

---

## TCP Connection Lifecycle

### Connection State

Each `Cube` object tracks:
- `connected` (boolean): Whether the cube is reachable
- `Heartbeat` (Date): Timestamp of last POWER message
- `getConnected()`: Returns `true` only if `connected == true` AND heartbeat is within `GAME_CONNECTION_TIMEOUT` (20 seconds)

### Connection Timeout

```java
public static int GAME_CONNECTION_TIMEOUT = 20; // seconds
```

If no POWER heartbeat is received for 20 seconds, `getConnected()` returns `false` and marks the cube as disconnected.

### TCP Client Storage

```java
// Constants.java
public static Map<String, Socket> TCPClients = new HashMap();  // IP → Socket
```

When the TCP server accepts a connection, it stores it: `TCPClients.put(ip, socket)`.  
Messages are sent to a specific cube by looking up its IP in this map.

### Reconnection

The `GameITargetFragment.assistantStart()` runs a background loop every 1 second that:
1. Calls `invUnConnected()` — iterates all cubes, calling `getConnected()` which triggers timeout checks
2. Calls `missChecked()` — checks for game timeout conditions
3. Checks if TCP server is closed and restarts if needed

The `ITargetHomeActivity.assistantStart()` runs a separate reconnection loop:
- Every 3 seconds, sends ALREADY 3× to each disconnected cube (message 300)
- Runs for up to 10 iterations then stops

---

## Game Protocol

### Game Types

| Type | Name | Description |
|------|------|-------------|
| 0 | Sequential Drill | Cubes flash in order, one at a time |
| 1 | Random Drill | One random cube flashes at a time |
| 2 | Clearing Drill | ALL cubes flash simultaneously, hit them all |
| 3 | Random Loop | Continuous random, repeats indefinitely |

### Game Start Sequence

When the user presses GO and selects a game type:

#### 1. Send OFF to all cubes (TCP)

```
TYPE=OFF,
```

Sent **twice** via `sendMultiTCPMessage()` (to all connected cubes via TCP).  
This resets all cubes to their idle state.

#### 2. Countdown

A visual countdown runs on the app (configurable delay, default 3 seconds).  
After countdown, TTS says "go".

#### 3. Game begins — FLASH (TCP)

The first cube(s) are selected and sent:

```
TYPE=FLASH,COUNT=2,
```

**Transport**: TCP via `sendSingleTCPMessage(cube.Ip, "TYPE=FLASH,COUNT=2,")`  
**Fields**:
- `COUNT`: Always `2` in all observed game modes (may control flash pattern/intensity)

**Cube selection by game type**:
- **Type 0 (Sequential)**: First unplayed connected cube in list order
- **Type 1 (Random)**: Random unplayed connected cube
- **Type 2 (Clearing)**: ALL connected cubes simultaneously (each gets FLASH individually)
- **Type 3 (Loop)**: Random connected cube, game resets and repeats

#### 4. Cube Hit — HIT (Cube → App, via TCP)

When a cube detects a hit, it sends:

```
TYPE=HIT,ID=<cube_id>,VALUE=<battery_level>,MIS=<reaction_time_ms>
```

**Fields**:
- `ID`: Cube identifier
- `VALUE`: Battery level
- `MIS`: Reaction time in milliseconds (time from FLASH to hit)

The app processes this in `OnParserTCPComplete()`:
- Updates cube power/connection status
- Records the hit time (`MIS / 1000.0` = seconds)
- TTS speaks the reaction time
- Triggers next cube selection

#### 5. Success Acknowledgment — SUCCESS (TCP)

After processing a HIT, the app sends:

```
TYPE=SUCCESS,
```

**Transport**: TCP to the cube that was hit  
**Note**: Found in the TCP callback handler (decompiled method was partially corrupted but this message type is referenced)

#### 6. Next Cube

After a hit, the app waits a configurable delay (`startDelayTime`, default 1 second) then flashes the next cube. For Random Drill (Type 1), an additional random delay of 0.5-3.5 seconds is added.

#### 7. Timeout — OVER (TCP)

If a cube is not hit within the timeout period (`outTime`, default 10 seconds):

```
TYPE=OVER,
```

Sent **twice** via TCP to the timed-out cube.  
TTS says "TimeOut".  
The cube's status is set to 3 (disconnected/failed).

#### 8. Game End

When all cubes have been hit or timed out (`gameIsOver()`):
- `isStartGame` is set to `false`
- Game history is saved
- UI returns to GO button state

For Loop mode (Type 3): The game never truly ends — it resets and continues until the user presses STOP.

### Cube Status Values

| Status | Meaning |
|--------|---------|
| 0 | Ready / Not yet played |
| 1 | Active / Flashing (waiting for hit) |
| 2 | Hit / Completed |
| 3 | Disconnected / Timed out |

---

## Message Reference

### App → Cube (UDP port 7042)

| Message | Format | When |
|---------|--------|------|
| INVITATION (broadcast) | `DEVICE=CUBE,TYPE=INVITATION,IP=<ip>` | Discovery scan, every IP on /24 |
| INVITATION (targeted) | `TYPE=INVITATION,IP="<ip>"` | After receiving IPGET from a cube |
| ALREADY | `TYPE=ALREADY,` | Triggers cube TCP connection |
| WIFI-INFO | `TYPE=WIFI-INFO,SSID=<ssid>,PWD=<password>,AIP=<app_ip>,SIP=<saved_ip>` | WiFi provisioning (initial setup) |
| FLASH (unused UDP path) | `TYPE=FLASH,COUNT=<n>` | Dead code path — game uses TCP instead |

### Cube → App (UDP port 4587)

| Message | Format | When |
|---------|--------|------|
| IPGET | `TYPE=IPGET,ID=<cube_id>` | Response to INVITATION |
| FINDWIFI | `TYPE=FINDWIFI` | Cube looking for WiFi config |
| POWER | `TYPE=POWER,ID=<id>,VALUE=<battery>` | Also received via UDP during discovery |

### App → Cube (TCP port 4586)

| Message | Format | When |
|---------|--------|------|
| FLASH | `TYPE=FLASH,COUNT=2,` | Activate cube's target light |
| OFF | `TYPE=OFF,` | Reset cube to idle (sent 2× at game start) |
| OVER | `TYPE=OVER,` | Cancel active flash / timeout (sent 2×) |
| SUCCESS | `TYPE=SUCCESS,` | Acknowledge hit received |
| SET | `TYPE=SET,VALUE=<sensitivity>` | Set cube sensitivity (0-? from SeekBar) |
| RELINK | `TYPE=RELINK,` | Tell cubes to reconnect (sent before WiFi reconfiguration) |

### Cube → App (TCP port 4586)

| Message | Format | When |
|---------|--------|------|
| HIT | `TYPE=HIT,ID=<cube_id>,VALUE=<battery>,MIS=<ms>` | Cube was hit |
| POWER | `TYPE=POWER,ID=<cube_id>,VALUE=<battery>` | Heartbeat / keep-alive |

---

## Timing & Reconnection

### Key Timing Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `GAME_CONNECTION_TIMEOUT` | 20 seconds | No heartbeat → cube marked disconnected |
| `GAME_INITIAL_TIME` | 10000 | Initial "time" value for unplayed cubes |
| `GAME_OUT_TIME` | 10001 | Sentinel value for timed-out cubes |
| Discovery sleep | 5ms | Delay between UDP packets to each IP |
| ALREADY send count | 4-5× | Number of ALREADY packets sent per connection attempt |
| ALREADY interval | 100-500ms | Sleep between ALREADY sends |
| Reconnection interval | 3 seconds | Background loop checks disconnected cubes |
| Reconnection ALREADY | 3× per cube | ALREADY packets sent during reconnection |
| TCP message post-sleep | 100ms | Sleep after each TCP write |
| assistantStart loop | 1 second | Game fragment monitoring loop interval |

### Reconnection Flow

1. `assistantStart()` in `GameITargetFragment` runs every 1 second
2. Calls `getConnected()` on every cube — if heartbeat is older than 20s, marks disconnected
3. `ITargetHomeActivity.assistantStart()` checks reconnection flag every 3s
4. For each disconnected cube, sends ALREADY 3× via UDP to port 7042
5. If cube responds, it reconnects via TCP

---

## Settings & Configuration

### Configurable Game Parameters

| Setting | Default | Stored In | Description |
|---------|---------|-----------|-------------|
| Delay (countdown) | 3 | SharedPreferences "Delay" | Countdown seconds before game start |
| OutTime | 10 | SharedPreferences "OutTime" | Seconds before a cube times out |
| StartDelayTime | 1 | SharedPreferences "StartDelayTime" | Delay between shots (seconds) |
| MagazineSize | 0 | SharedPreferences "MagazineSize" | Shots per magazine in Loop mode (0 = unlimited) |
| Sensitivity | SeekBar value | Sent via TCP SET | Cube hit sensitivity |

### SET Command

When settings are saved, the app sends to all cubes:

```
TYPE=SET,VALUE=<seekbar_value>
```

Sent via `sendMultiTCPMessage()` (TCP to all connected cubes).

---

## Authentication & Security

### **There is NO authentication whatsoever.**

- No passwords, tokens, or keys in any message
- No TLS/SSL on any connection
- No MAC address verification
- No challenge-response
- No encryption of any kind
- Cube IDs are just identifiers (`iTarget` + digits), not secrets
- WiFi passwords are sent **in plaintext** via UDP during provisioning

### Security Implications

Anyone on the same WiFi network can:
1. **Discover all cubes** by sending INVITATION broadcasts
2. **Take control of any cube** by sending ALREADY + TCP commands
3. **Interfere with games** by sending FLASH/OFF/OVER to cubes
4. **Sniff WiFi credentials** if they capture WIFI-INFO packets
5. **Spoof cube responses** (HIT, POWER) to the app

---

## WiFi Provisioning (Initial Setup)

> This section covers the initial cube WiFi setup, NOT the normal game protocol.  
> You specified to assume cubes are already on WiFi, but this is documented for completeness.

### Method 1: AP Config (CONFIG_TYPE == 0)

1. Cube creates its own WiFi hotspot (SSID: `AI-THINKER-XXXXXX`)
2. User connects phone to cube's hotspot
3. App scans for `AI-THINKER` SSIDs in WiFi scan results
4. App sends WIFI-INFO via UDP to the cube's gateway (x.x.x.1):
   ```
   TYPE=WIFI-INFO,SSID=<home_ssid>,PWD=<home_password>,AIP=<app_ip>,SIP=<saved_ip>
   ```
5. Cube joins the home WiFi network
6. User switches phone back to home WiFi
7. Normal discovery flow begins

### Method 2: ESP-Touch / SmartConfig (CONFIG_TYPE == 1)

Uses the Espressif ESP-Touch protocol (SmartConfig) to provision the cube:
1. App broadcasts WiFi credentials via ESP-Touch multicast
2. Cube (ESP8266/ESP32 based) captures and joins the network
3. Normal discovery flow begins

### Method 3: Bluetooth Config

Uses Realtek BT configuration (`RTK BTConfig`) as an alternative provisioning method.

---

## Source Code Reference

### Key Classes

| Class | Purpose |
|-------|---------|
| `UDPBuild` | Singleton. UDP socket on port 4587. Sends INVITATION, ALREADY, FLASH (unused), WIFI-INFO |
| `UUdp` | Static helper. Sends UDP packets to one or more IPs |
| `MySocketServer` | Singleton. TCP server on port 4586. Manages TCP client connections |
| `Constants` | Static config. Ports, timeouts, TCPClients map |
| `Tools` | Message parser (`stringToMap()`), validation, EventBus helpers |
| `Game` | Game logic. Cube selection, hit processing, game modes |
| `Cube` | Cube model. IP, ID, status, heartbeat, battery, connection state |
| `AppContext` | Singleton. App state, cube list, settings accessors |
| `GameITargetFragment` | Main game UI. TCP callback handler, game flow, reconnection loop |
| `ITargetHomeActivity` | Main activity. UDP handler, discovery, IPGET processing |
| `NetWorkUtil` | WiFi IP address utilities |
| `SettingFragment` | Settings UI. Sends SET command to cubes |

### Key Methods

| Method | Class | Purpose |
|--------|-------|---------|
| `sendInvitation(context)` | UDPBuild | Broadcast discovery to /24 subnet |
| `sendInvitation(context, ip)` | UDPBuild | Targeted invitation to specific cube |
| `sendALREADY(ip)` | UDPBuild | Trigger TCP connection from cube |
| `startUDPSocket()` | UDPBuild | Bind UDP port 4587 and start listener |
| `startServerAsync()` | MySocketServer | Start TCP server on port 4586 |
| `sendSingleTCPMessage(ip, msg)` | MySocketServer | Send TCP message to one cube by IP |
| `sendMultiTCPMessage(msg)` | MySocketServer | Send TCP message to all connected cubes |
| `stringToMap(str)` | Tools | Parse `KEY=VAL,KEY=VAL,` format |
| `OnParserTCPComplete(ip, map)` | GameITargetFragment | Handle incoming TCP messages (HIT, POWER) |
| `nextCube(type)` / `nextCube0-3()` | Game | Select and flash next cube based on game type |
| `handleMessage(200)` | ITargetHomeActivity.MyHandler | Process IPGET — add cube, send ALREADY |

---

## Notes for Python Re-implementation

### Minimal Connection Sequence

```python
# 1. Start UDP listener on port 4587
# 2. Start TCP server on port 4586
# 3. Broadcast INVITATION to subnet
# 4. Wait for IPGET responses on UDP 4587
# 5. Send targeted INVITATION to each cube
# 6. Send ALREADY 4-5× to each cube (triggers TCP connection)
# 7. Accept TCP connections on 4586
# 8. Handle POWER heartbeats on TCP to track connectivity
```

### Message Construction

```python
def build_message(**kwargs):
    """Build an iTarget protocol message."""
    return ",".join(f"{k}={v}" for k, v in kwargs.items()) + ","

def parse_message(data: bytes) -> dict:
    """Parse an iTarget protocol message."""
    text = data.decode("utf-8").strip().rstrip("\r\n")
    result = {}
    for token in text.split(","):
        if "=" in token:
            key, value = token.split("=", 1)
            result[key.strip()] = value.strip().strip('"')
    return result
```

### Key Implementation Details

1. **UDP is fire-and-forget**: Each `sendMessage` in `UUdp` creates a **new** `DatagramSocket()` per packet (no reuse). The app's listening socket on 4587 is separate.

2. **TCP server must stay running**: Cubes may reconnect at any time. The `ServerSocket.accept()` loop runs continuously.

3. **TCP messages are raw strings**: No length prefix, no framing. Just `socket.getOutputStream().write(str.getBytes())`. The read side uses a fixed 4096-byte buffer. **Multiple messages may arrive in one read, or one message may span multiple reads.**

4. **ALREADY is the critical handshake**: Without it, cubes won't initiate TCP. Send it multiple times for reliability.

5. **Heartbeat timeout is 20 seconds**: If you don't receive POWER within 20s, consider the cube disconnected.

6. **Game FLASH is TCP-only**: Despite a UDP `sendFlash()` method existing in `UDPBuild`, all game modes use TCP `sendSingleTCPMessage()`.

7. **Send OFF twice before starting a game**: The app always sends `TYPE=OFF,` twice to all cubes via TCP before beginning.

8. **Send OVER twice on timeout**: The app sends `TYPE=OVER,` twice to a timed-out cube.

9. **IPGET cube IDs follow pattern `iTarget[0-9]+`**: The app validates this with regex before accepting a cube.

10. **The app skips its own IP and connected cube IPs during discovery**: When scanning, it builds a "skip list" of its own IP and all connected cube IPs.

### Suggested Python Architecture

```
┌─────────────────────────────────────┐
│           Main Controller           │
├─────────────────────────────────────┤
│  ┌─────────┐  ┌──────────────────┐  │
│  │UDP Send  │  │ UDP Listener     │  │
│  │(any port)│  │ (port 4587)      │  │
│  └─────────┘  └──────────────────┘  │
│  ┌──────────────────────────────┐   │
│  │ TCP Server (port 4586)       │   │
│  │  - accept loop               │   │
│  │  - per-client read loop      │   │
│  │  - client map {ip: socket}   │   │
│  └──────────────────────────────┘   │
│  ┌──────────────────────────────┐   │
│  │ Cube State Manager           │   │
│  │  - id, ip, connected, power  │   │
│  │  - last_heartbeat timestamp  │   │
│  └──────────────────────────────┘   │
│  ┌──────────────────────────────┐   │
│  │ Game Engine                  │   │
│  │  - sequential/random/clear   │   │
│  │  - flash → wait hit → next   │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
```

### Example: Discover and Connect One Cube

```python
import socket
import time

LOCAL_IP = "192.168.1.100"  # Your IP
SUBNET = "192.168.1"
UDP_CUBE_PORT = 7042
UDP_LISTEN_PORT = 4587
TCP_PORT = 4586

# Start UDP listener
udp_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
udp_sock.bind(("0.0.0.0", UDP_LISTEN_PORT))
udp_sock.settimeout(5.0)

# Broadcast INVITATION
for i in range(2, 255):
    target = f"{SUBNET}.{i}"
    if target == LOCAL_IP:
        continue
    msg = f"DEVICE=CUBE,TYPE=INVITATION,IP={LOCAL_IP}"
    send_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    send_sock.sendto(msg.encode(), (target, UDP_CUBE_PORT))
    send_sock.close()
    time.sleep(0.005)

# Wait for IPGET
while True:
    data, addr = udp_sock.recvfrom(1024)
    msg = parse_message(data)
    if msg.get("TYPE") == "IPGET":
        cube_ip = addr[0]
        cube_id = msg.get("ID")
        print(f"Found cube: {cube_id} at {cube_ip}")
        break

# Send targeted INVITATION
send_udp(cube_ip, UDP_CUBE_PORT, f'TYPE=INVITATION,IP="{LOCAL_IP}"')

# Send ALREADY to trigger TCP
for _ in range(5):
    send_udp(cube_ip, UDP_CUBE_PORT, "TYPE=ALREADY,")
    time.sleep(0.1)

# Accept TCP connection
tcp_server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
tcp_server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
tcp_server.bind(("0.0.0.0", TCP_PORT))
tcp_server.listen(5)
tcp_server.settimeout(10.0)

client_sock, client_addr = tcp_server.accept()
print(f"TCP connection from {client_addr}")

# Read POWER heartbeat
data = client_sock.recv(4096)
msg = parse_message(data)
if msg.get("TYPE") == "POWER":
    print(f"Cube {msg.get('ID')} battery: {msg.get('VALUE')}%")

# Flash the cube!
client_sock.send(b"TYPE=FLASH,COUNT=2,")

# Wait for HIT
data = client_sock.recv(4096)
msg = parse_message(data)
if msg.get("TYPE") == "HIT":
    reaction_ms = msg.get("MIS")
    print(f"HIT! Reaction time: {reaction_ms}ms")
    client_sock.send(b"TYPE=SUCCESS,")
```

---

## Protocol Summary Diagram

```
    App (Phone)                                     Cube
    ───────────                                     ────
         │                                            │
         │──── INVITATION (UDP:7042, broadcast) ─────►│
         │                                            │
         │◄──── IPGET (UDP:4587) ────────────────────│
         │                                            │
         │──── INVITATION (UDP:7042, targeted) ──────►│
         │                                            │
         │──── ALREADY ×5 (UDP:7042) ────────────────►│
         │                                            │
         │◄════ TCP connect (→ port 4586) ═══════════│
         │                                            │
         │◄──── POWER heartbeat (TCP) ───────────────│
         │        ... periodic ...                    │
         │                                            │
    [GAME START]                                      │
         │                                            │
         │──── OFF ×2 (TCP, all cubes) ──────────────►│
         │                                            │
    [COUNTDOWN]                                       │
         │                                            │
         │──── FLASH,COUNT=2 (TCP) ──────────────────►│
         │                                            │
         │◄──── HIT,ID=..,MIS=.. (TCP) ─────────────│
         │                                            │
         │──── SUCCESS (TCP) ────────────────────────►│
         │                                            │
    [NEXT CUBE or GAME END]                           │
         │                                            │
         │──── OVER ×2 (TCP, if timeout) ────────────►│
         │                                            │
```
