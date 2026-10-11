# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`hex_zmq_servers` is a distributed device control framework built on ZeroMQ for HEXFELLOW robotics devices. It provides a client-server architecture where device servers handle hardware communication while clients send commands and receive state updates over TCP.

## Development Setup

### Installation

```bash
# Full installation with all device drivers
./venv.sh

# Minimal installation (no extra device drivers)
./venv.sh --min

# Package only (no examples)
./venv.sh --pkg-only
```

This uses `uv` package manager to create a Python 3.11 virtual environment in `.venv/` and installs the package in editable mode.

### Running Examples

Examples follow a launcher pattern:

```bash
# Run a basic example
python examples/basic/robot_dummy/launch.py

# Run an advanced multi-device example
python examples/adv/gello_real/launch.py
```

Each example has:
- `launch.py`: Orchestrates multiple processes (servers and clients)
- `cli.py`: Client implementation that connects to device servers
- `cli.json` / `srv.json`: Configuration files

## Architecture

### Core Components

1. **ZMQ Base Layer** (`zmq_base.py`):
   - `HexZMQClientBase`: REQ socket client with automatic reconnection
   - `HexZMQServerBase`: ROUTER-DEALER proxy with worker pool (4 workers default)
   - `HexSafeValue`: Thread-safe value container for passing data between device and server threads
   - `HexRate`: Precise rate-limiting for control loops
   - Communication uses JSON headers + numpy buffer (multipart messages)

2. **Device Base Layer** (`device_base.py`):
   - `HexDeviceBase`: Abstract base for hardware devices
   - Devices run `work_loop()` in separate thread, communicating via `HexSafeValue` objects
   - Subclasses: `HexRobotBase`, `HexCamBase`, `HexMujocoBase`

3. **Server/Client Pattern**:
   - Each device type has three components:
     - Device class (e.g., `HexRobotDummy`): Hardware abstraction
     - Server class (e.g., `HexRobotDummyServer`): ZMQ server wrapping device
     - Client class (e.g., `HexRobotDummyClient`): ZMQ client for remote control

4. **Launch System** (`hex_launch.py`):
   - `HexLaunch`: Process orchestrator that manages multiple device nodes
   - Handles logging, respawning, signal handling, terminal restoration
   - Nodes configured via list of dicts with `node_path`, `cfg_path`, `venv`, etc.

### Device Types

- **Robot** (`robot/`): Robotic arms (dummy, gello, hexarm)
  - Commands: `get_dofs`, `get_limits`, `get_states`, `set_cmds`
  - State includes position, velocity, torque per joint

- **Camera** (`cam/`): RGB/depth cameras (dummy, berxel)
  - Commands: `get_rgb`, `get_depth`
  - Images returned as numpy arrays

- **Mujoco** (`mujoco/`): Physics simulations (archer_d6y, e3_desktop)
  - Combines robot and camera interfaces
  - Additional commands: `reset`, `get_obj_pose`, `get_intri`

### Communication Protocol

All requests/responses use this structure:
- **Header** (JSON): `{"cmd": str, "ts": dict, "args": Any, "dtype": str, "shape": tuple}`,
  plus `"host_ts": float` (the frame's monotonic receipt stamp) when the handler supplies one
- **Buffer** (bytes): C-contiguous numpy array

Sequence numbers prevent stale data (using modulo `1e12`).

### Configuration System

Configs merge in this order:
1. Default values in server classes
2. JSON config file (if `cfg_path` provided)
3. Runtime overrides (via `cfg` dict in node config)

The `__dict_update` method in `hex_launch.py` recursively merges nested dicts.

## Key Design Patterns

### Thread Safety
- Device hardware runs in main thread via `work_loop()`
- ZMQ server runs in worker threads
- Communication via `HexSafeValue` with locks and events

### Error Handling
- Commands return `_ok` or `_failed` suffix
- Clients check response header `cmd` field
- Server catches exceptions and returns error header with message

### Position Limits
- Robots use `_rads_normalize()` to wrap angles to [-π, π]
- `_apply_pos_limits()` clamps to joint limits, choosing nearest bound

## Common Tasks

### Adding a New Device Type

1. Create device class inheriting from `HexRobotBase`/`HexCamBase`/`HexMujocoBase`
2. Implement `work_loop(hex_values)` and `close()`
3. Create server class inheriting from corresponding server base
4. Implement `_process_request(recv_hdr, recv_buf)`
5. Create client class inheriting from corresponding client base
6. Add convenience methods for device-specific commands
7. Register in `__init__.py` and add config paths

### Modifying the Launch System

- Nodes are defined as dicts with keys: `name`, `node_path`, `cfg_path`, `venv`, `cwd`, `env`, `respawn`
- Logs go to `logs/info/` and `logs/err/` with timestamps
- Set `min_level` in `HexLaunch` to filter log verbosity (0=info, 1=warn, 2=err)

### Testing Device Communication

Use the `zmq_dummy` example to test basic client-server communication without hardware.

## Important Notes

- All servers run as `__main__` and expect `--cfg` argument with JSON string
- Use `hex_server_helper(cfg, ServerClass)` utility for consistent server startup
- Network configs default to localhost:12345 but can be overridden per device
- The framework uses process groups (`os.setsid`) for clean subprocess termination
- Terminal attributes are saved/restored to handle SIGINT properly
