# iTarget Cube Controller

Python reimplementation of the iTarget Pro Android app protocol for controlling iTarget training cubes.

## Overview

This project reverse-engineers the iTarget Pro v1.5.1 Android app to provide a Python-based controller for iTarget cube training targets. The protocol has **no authentication** - any device on the same WiFi network can control the cubes.

## Files

- **`itarget_cube.py`** - Main Python controller script
- **`PROTOCOL.md`** - Complete protocol documentation from reverse engineering
- **`app/src/main/`** - Decompiled Android app source code (JADX output)

## Requirements

- Python 3.7+
- iTarget cubes on the same WiFi network
- No additional dependencies (uses only standard library)

## Quick Start

### 1. Discover Cubes

```bash
python itarget_cube.py --discover
```

This will:
- Scan your local /24 subnet for cubes
- Show discovered cube IDs and IPs
- Wait for TCP connections to establish

### 2. Run a Game

#### Sequential Drill (cubes flash in order)
```bash
python itarget_cube.py --game sequential
```

#### Random Drill (random order with delays)
```bash
python itarget_cube.py --game random
```

#### Clearing Drill (all cubes at once)
```bash
python itarget_cube.py --game clearing
```

#### Random Loop (continuous, Ctrl+C to stop)
```bash
python itarget_cube.py --game loop
```

### 3. Interactive Mode

Run without arguments to choose game type interactively:

```bash
python itarget_cube.py
```

## Advanced Usage

### Enable Debug Logging

See all packets sent/received and protocol details:

```bash
python itarget_cube.py --game sequential --debug
```

Debug mode shows:
- UDP/TCP packet contents (sent/received)
- Cube discovery process
- Connection establishment
- Heartbeat messages
- Hit detection and timing

### Custom Game Settings

```bash
python itarget_cube.py --game clearing \
  --countdown 5 \
  --timeout 15 \
  --delay 2
```

Options:
- `--countdown N` - Seconds before game starts (default: 3)
- `--timeout N` - Seconds before cube times out (default: 10)
- `--delay N` - Delay between shots in seconds (default: 1)

## Game Modes

| Mode | Description |
|------|-------------|
| **Sequential** | Cubes flash in order, one at a time. Predictable pattern. |
| **Random** | Random cube order with 0.5-3.5s random delays between shots. |
| **Clearing** | All cubes flash simultaneously. Hit them all before timeout. |
| **Loop** | Continuous random mode. Runs until you press Ctrl+C. |

## How It Works

### Protocol Overview

The iTarget system uses three communication channels:

```
┌──────────────┐         WiFi          ┌──────────────┐
│   Your PC    │◄────────────────────►│  iTarget Cube│
│              │                        │              │
│  UDP :4587   │◄─ IPGET, POWER ───────│  UDP :7042   │
│  (listen)    │─ INVITATION, ALREADY ─►│  (listen)    │
│              │                        │              │
│  TCP :4586   │◄─ cube connects ──────│              │
│  (server)    │─ FLASH, OFF, OVER ────►│              │
│              │◄─ HIT, POWER ──────────│              │
└──────────────┘                        └──────────────┘
```

### Discovery & Connection Flow

1. **Broadcast INVITATION** (UDP → 7042) to all IPs on subnet
2. **Cubes respond with IPGET** (UDP → 4587) containing their ID
3. **Send ALREADY 5×** (UDP → 7042) - triggers cube to connect
4. **Cube opens TCP connection** to controller (→ 4586)
5. **Periodic POWER heartbeats** (TCP) maintain connection

### Game Flow

1. Send `TYPE=OFF,` 2× to reset all cubes (TCP)
2. Countdown (3 seconds default)
3. Send `TYPE=FLASH,COUNT=2,` to activate target (TCP)
4. Wait for `TYPE=HIT,ID=...,MIS=...` from cube (TCP)
5. Send `TYPE=SUCCESS,` acknowledgment (TCP)
6. Repeat for next cube or end game

## Protocol Details

See **`PROTOCOL.md`** for complete protocol documentation including:
- All message types and formats
- Timing constants and reconnection logic
- TCP connection lifecycle
- Security analysis (spoiler: there is none)
- Python implementation notes

## Security Warning

⚠️ **The iTarget protocol has ZERO security**:
- No authentication or encryption
- No password protection
- Anyone on your WiFi can control your cubes
- WiFi passwords sent in plaintext during provisioning

**Recommendations**:
- Use on isolated/guest WiFi network
- Don't use on public WiFi
- Assume anyone on the network can interfere with your training

## Troubleshooting

### No cubes found

- Ensure cubes are powered on (two lights flashing)
- Verify cubes are on the same WiFi network
- Check your PC's firewall allows UDP 4587 and TCP 4586
- Try discovery with `--debug` to see network traffic

### Cubes discovered but won't connect

- Wait 10-15 seconds for TCP connections to establish
- Check cube battery level (low battery = slow connection)
- Restart cubes by power cycling
- Use `--debug` to see if ALREADY packets are being sent

### Game starts but cubes don't flash

- Verify TCP connections are established (see connection count)
- Check if cubes respond to POWER heartbeats in debug mode
- Ensure cubes aren't already in a game (send OFF to reset)

### Cubes timeout immediately

- Increase timeout: `--timeout 20`
- Check cube battery level
- Verify cubes are actually flashing (LED should be on)
- Use `--debug` to see if HIT messages are being received

## Development

### Modifying the Script

The code is well-commented and follows the protocol exactly. Key classes:

- **`ITargetController`** - Main controller, handles discovery and games
- **`Cube`** - Represents a single cube with state
- **`ProtocolMessage`** - Message builder/parser
- **`GameType`** - Enum for game modes

### Adding New Game Modes

1. Add new `GameType` enum value
2. Implement `_run_<mode>()` method in `ITargetController`
3. Add to game type map in `main()`

### Testing Without Cubes

You can test discovery and protocol flow by:
1. Running with `--debug` to see all network traffic
2. Using Wireshark to capture packets
3. Creating a mock cube responder (listens on 7042, responds to INVITATION)

## Credits

- Protocol reverse-engineered from iTarget Pro v1.5.1 Android app
- Decompiled with JADX
- Python reimplementation by analyzing Java source code

## License

This is reverse-engineered research code. Use at your own risk.

No warranty. No support. No affiliation with iTarget.
