# Packet Tracer MCP Server

An MCP (Model Context Protocol) server that lets any LLM (GitHub Copilot, Claude, etc.) create, configure, validate, and **deploy in real time** complete network topologies to Cisco Packet Tracer.

Tell your LLM _"create a network with 3 routers, DHCP and OSPF"_ and the server plans the topology, validates everything, generates the scripts and configs, and deploys it directly to Packet Tracer in real time.

**Python 3.11+ · Pydantic 2.0+ · FastMCP · Streamable HTTP · v0.5.0**

---

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [MCP Client Configuration](#mcp-client-configuration)
- [Live Deploy](#live-deploy)
- [MCP Tools (35)](#mcp-tools-35)
- [MCP Resources (5)](#mcp-resources-5)
- [Supported Devices](#supported-devices)
- [IP Addressing](#ip-addressing)
- [Routing Protocols](#routing-protocols)
- [Topology Templates](#topology-templates)
- [Scenario Presets](#scenario-presets)
- [IOS Config Templates](#ios-config-templates)
- [Architecture](#architecture)
- [PTBuilder Extension](#ptbuilder-extension)
- [Testing](#testing)
- [Environment Variables](#environment-variables)

---

## Installation

```bash
git clone https://github.com/deiviidsito/mcp_packet_tracer
cd mcp_packet_tracer

# Production
pip install -e .

# Development (includes pytest, ruff, mypy)
pip install -e ".[dev]"
```

---

## Quick Start

### 1. Start the server

```bash
python -m src.packet_tracer_mcp
```

This starts two services automatically:

| Service | Address | Purpose |
|---------|---------|---------|
| **MCP Server** | `http://127.0.0.1:39000/mcp` | Receives LLM tool requests (streamable-http) |
| **HTTP Bridge** | `http://127.0.0.1:54321` | Sends commands to Packet Tracer in real time |

> For stdio transport (debug/legacy clients): `python -m src.packet_tracer_mcp --stdio`

### 2. Configure your MCP client

See [MCP Client Configuration](#mcp-client-configuration) below.

### 3. Ask the LLM to create a network

```
"Create a network with 2 routers, 2 switches, 4 PCs, DHCP, and static routing"

→ pt_full_build generates:
  - 8 devices: R1, R2, SW1, SW2, PC1–PC4
  - 7 links: R1↔R2 (crossover), R1↔SW1, R2↔SW2, SW1↔PC1/PC2, SW2↔PC3/PC4
  - IPs: LAN1 192.168.0.0/24, LAN2 192.168.1.0/24, inter-router 10.0.0.0/30
  - DHCP pools on R1 and R2
  - Bidirectional static routes
  - 23 JavaScript commands sent directly to Packet Tracer
```

---

## MCP Client Configuration

**VS Code** (`.vscode/mcp.json`):

```json
{
  "servers": {
    "packet-tracer": {
      "url": "http://127.0.0.1:39000/mcp"
    }
  }
}
```

**Claude Desktop** (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "packet-tracer": {
      "url": "http://127.0.0.1:39000/mcp"
    }
  }
}
```

**Claude Code** (`.mcp.json` in project root):

```json
{
  "mcpServers": {
    "packet-tracer": {
      "type": "http",
      "url": "http://127.0.0.1:39000/mcp"
    }
  }
}
```

---

## Live Deploy

The main feature: send commands directly to Packet Tracer without copy-pasting anything.

```
┌─────────┐         ┌──────────────┐   HTTP    ┌──────────────┐  $se()  ┌──────────────┐
│   LLM   │  MCP    │  MCP Server  │  :54321   │  PTBuilder   │  IPC   │ Packet Tracer│
│(Copilot)│ ──────► │  (:39000)    │ ────────► │  (WebView)   │ ─────► │   (Engine)   │
└─────────┘         └──────────────┘           └──────────────┘        └──────────────┘
```

### Setup (once per Packet Tracer session)

1. Open **Packet Tracer 8.2+**
2. Open **Builder Code Editor** (`Extensions > Builder Code Editor`)
3. Paste the bootstrap script below and click **Run**:

```javascript
/* PT-MCP Bridge */ window.webview.evaluateJavaScriptAsync("setInterval(function(){var x=new XMLHttpRequest();x.open('GET','http://127.0.0.1:54321/next',true);x.onload=function(){if(x.status===200&&x.responseText){$se('runCode',x.responseText)}};x.onerror=function(){};x.send()},500)");
```

This makes PTBuilder poll the bridge every 500 ms. When the LLM generates commands, the MCP Server queues them on the bridge and Packet Tracer executes them in real time.

> **Technical note:** The bootstrap injects a `setInterval` into the webview that polls HTTP. `$se('runCode', ...)` bridges from the webview to the PT Script Engine. `/* */` comments are used instead of `//` because PTBuilder's `executeCode()` strips newlines.

### Permanent setup (optional)

To have the bridge start automatically when Builder Code Editor opens:

1. In PT: `Extensions > Scripting Interface`
2. Select the Builder module
3. Replace `main.js` and `interface.js` with the modified versions in `PTBuilder/source/`
4. Save and restart the module

---

## MCP Tools (35)

### Catalog

| Tool | Description |
|------|-------------|
| `pt_list_devices` | List all available devices with ports |
| `pt_list_templates` | List available topology templates |
| `pt_get_device_details` | Full port/interface details for a device model |

### Estimation

| Tool | Description |
|------|-------------|
| `pt_estimate_plan` | Dry-run: estimate device/link counts without generating a full plan |

### Planning

| Tool | Description |
|------|-------------|
| `pt_plan_topology` | Generate a complete plan (devices, links, IPs, DHCP, routes) |

### Validation

| Tool | Description |
|------|-------------|
| `pt_validate_plan` | Validate a plan with 15 typed error codes |
| `pt_fix_plan` | Auto-correct common errors (cables, models, ports) |
| `pt_explain_plan` | Natural language explanation of every plan decision |
| `pt_validate_config` | Validate IOS config lines against the topology (IP conflicts, missing `no shutdown`, ACL mismatches) |
| `pt_validate_topology` | Deep topology checks (orphaned devices, loops without STP, OSPF mismatches) |

### Generation

| Tool | Description |
|------|-------------|
| `pt_generate_script` | Generate the PTBuilder JavaScript script |
| `pt_generate_configs` | Generate per-device IOS CLI configurations |

### Full Pipeline

| Tool | Description |
|------|-------------|
| `pt_full_build` | Plan + validate + generate + deploy in one call |

### Live Deploy & Bridge

| Tool | Description |
|------|-------------|
| `pt_live_deploy` | Send commands directly to PT in real time via HTTP bridge |
| `pt_deploy` | Copy script to clipboard with step-by-step instructions |
| `pt_bridge_status` | Check whether the bridge is active |
| `pt_ping_bridge` | Health check — returns bridge_up, pt_connected, url |
| `pt_undo_last_action` | Undo the last command sent to PT |
| `pt_load_last_plan` | Reload the last successfully deployed plan from disk |

### Topology Interaction

| Tool | Description |
|------|-------------|
| `pt_query_topology` | Query which devices currently exist in PT |
| `pt_delete_device` | Delete a device and its links from PT |
| `pt_rename_device` | Rename a device in the active topology |
| `pt_move_device` | Move a device to new canvas coordinates |
| `pt_delete_link` | Delete the link on a specific interface |
| `pt_send_raw` | Send arbitrary JavaScript to the PT Script Engine |

### Intelligence

| Tool | Description |
|------|-------------|
| `pt_analyze_topology` | Parse a natural language topology description into a structured plan |
| `pt_suggest_improvements` | Analyze a plan and suggest redundancy, security, and best-practice improvements |
| `pt_calculate_addressing` | Auto-generate IPv4/IPv6 dual-stack addressing for multiple sites |

### IOS Config Templates

| Tool | Description |
|------|-------------|
| `pt_list_config_templates` | List all available Jinja2 IOS config templates |
| `pt_apply_template` | Render a template with context and get ready-to-paste CLI commands |

### Scenario Presets

| Tool | Description |
|------|-------------|
| `pt_list_presets` | List all available scenario presets with descriptions |
| `pt_load_preset` | Load a preset and generate a complete build plan |

### Export & Projects

| Tool | Description |
|------|-------------|
| `pt_export` | Export plan + scripts + configs to files (JS, CLI, JSON) |
| `pt_export_documentation` | Generate full documentation: addressing table, topology description, verification commands |
| `pt_list_projects` | List saved projects |
| `pt_load_project` | Load a saved project |

---

## MCP Resources (5)

| URI | Description |
|-----|-------------|
| `pt://catalog/devices` | All devices with ports |
| `pt://catalog/cables` | Cable types |
| `pt://catalog/aliases` | Model aliases |
| `pt://catalog/templates` | Topology templates |
| `pt://capabilities` | Server capabilities |

---

## Supported Devices

### Routers

| Model | Interfaces |
|-------|-----------|
| 1941 | GigabitEthernet0/0, GigabitEthernet0/1 |
| 2901 | GigabitEthernet0/0, GigabitEthernet0/1 |
| 2911 | GigabitEthernet0/0, GigabitEthernet0/1, GigabitEthernet0/2 |
| ISR4321 | GigabitEthernet0/0/0, GigabitEthernet0/0/1 |

> **Note:** No router has serial ports by default. Serial interfaces require HWIC modules.

### Switches

| Model | Interfaces |
|-------|-----------|
| 2960-24TT | FastEthernet0/1–24, GigabitEthernet0/1–2 |
| 3560-24PS | FastEthernet0/1–24, GigabitEthernet0/1–2 |

### End Devices

| Model | Interface |
|-------|----------|
| PC-PT | FastEthernet0 |
| Server-PT | FastEthernet0 |
| Laptop-PT | FastEthernet0 |

### Other

| Model | Type |
|-------|------|
| Cloud-PT | WAN Cloud |
| AccessPoint-PT | Wireless AP |

---

## Cable Types

| Cable | Typical Use |
|-------|-------------|
| `straight` | Switch ↔ Router, Switch ↔ PC |
| `cross` | Router ↔ Router, Switch ↔ Switch, PC ↔ PC |
| `serial` | Router serial ↔ Router serial (WAN) |
| `fiber` | Fiber-optic connections |
| `auto` | Auto-detection |

---

## IP Addressing

- **LANs:** `192.168.0.0/16` base, `/24` prefixes — gateway at `.1`, PCs from `.2`
- **Inter-router links:** `10.0.0.0/16` base, `/30` prefixes — 2 hosts per link
- **DHCP:** Automatic pool per LAN with gateway exclusion
- **IPv6 dual-stack:** Available via `pt_calculate_addressing` (`fd00::/48` ULA base)

---

## Routing Protocols

| Protocol | Status | Generates |
|----------|--------|-----------|
| `static` | Complete | `ip route` commands; supports floating routes with AD=254 |
| `ospf` | Complete | `router ospf` with router-id and area support |
| `rip` | Complete | `router rip` v2 with `no auto-summary` |
| `eigrp` | Complete | `router eigrp` with wildcard masks and AS number |
| `none` | Complete | No routing configured |

---

## Topology Templates

| Template | Description |
|----------|-------------|
| `single_lan` | 1 router + 1 switch + PCs |
| `multi_lan` | N routers interconnected, each with its own LAN |
| `multi_lan_wan` | Multi-LAN with WAN cloud |
| `star` | Central router with satellite routers |
| `hub_spoke` | Hub-and-spoke |
| `branch_office` | Branch offices |
| `router_on_a_stick` | Inter-VLAN routing |
| `three_router_triangle` | 3 routers in a triangle |
| `custom` | Custom |

---

## Scenario Presets

Ready-made topologies that generate complete, wired, and configured plans in one call.

| Preset | Description |
|--------|-------------|
| `small_office` | 1 router, 1 switch, 5 PCs with DHCP |
| `branch_hq` | 2 sites connected via WAN, OSPF, DHCP |
| `dmz_network` | Router as firewall with DMZ server zone and internal LAN |
| `redundant_core` | Dual core with floating static routes |
| `full_enterprise` | HQ + 2 branches, OSPF, servers |
| `ccna_lab_1` | Classic CCNA exam topology |
| `ccnp_switch_lab` | Multi-switch lab, OSPF |
| `ipv6_dual_stack` | IPv4 + IPv6 dual-stack |

---

## IOS Config Templates

Jinja2-based templates that generate ready-to-paste IOS CLI commands.

| Template | Description |
|----------|-------------|
| `ospf_basic` | OSPF with areas and passive interfaces |
| `eigrp_named` | EIGRP named mode |
| `vlan_trunk` | VLAN creation + trunk ports |
| `hsrp_pair` | HSRP active/standby pair |
| `nat_overload` | NAT overload (PAT) |
| `acl_dmz` | Extended ACL for DMZ |
| `dhcp_server` | DHCP pool with exclusions |
| `stp_rapid` | Rapid PVST+ with root bridge |

---

## Architecture

```
src/packet_tracer_mcp/
├── adapters/mcp/              # MCP protocol layer
│   ├── tools/                 # Split by domain concern (9 modules)
│   │   ├── catalog_tools.py        # pt_list_devices, pt_list_templates, pt_get_device_details
│   │   ├── planning_tools.py       # pt_estimate_plan, pt_plan_topology
│   │   ├── validation_tools.py     # pt_validate_plan, pt_fix_plan, pt_explain_plan,
│   │   │                           # pt_validate_config, pt_validate_topology
│   │   ├── generation_tools.py     # pt_generate_script, pt_generate_configs, pt_full_build
│   │   ├── deploy_tools.py         # pt_export, pt_deploy, pt_list_projects, pt_load_project,
│   │   │                           # pt_export_documentation
│   │   ├── bridge_tools.py         # pt_live_deploy, pt_bridge_status, pt_ping_bridge,
│   │   │                           # pt_undo_last_action, pt_load_last_plan,
│   │   │                           # pt_query/delete/rename/move/send_raw
│   │   ├── topology_tools.py       # pt_analyze_topology, pt_suggest_improvements,
│   │   │                           # pt_calculate_addressing
│   │   ├── preset_tools.py         # pt_list_presets, pt_load_preset
│   │   └── template_tools.py       # pt_list_config_templates, pt_apply_template
│   ├── tool_registry.py       # Coordinator — delegates to tools/
│   └── resource_registry.py   # 5 MCP resources
├── application/               # Use cases + DTOs (requests/responses)
├── domain/                    # Core business logic
│   ├── models/               # TopologyPlan, DevicePlan, LinkPlan, errors, TopologyAnalysis
│   ├── services/             # Orchestrator, IPPlanner, Validator, AutoFixer, Explainer,
│   │                         # Estimator, TopologyAnalyzer, TemplateEngine, Presets
│   └── rules/                # Validation rules (devices, cables, IPs)
├── infrastructure/
│   ├── catalog/              # Device catalog, cable types, templates, aliases
│   ├── generator/            # PTBuilder JS + CLI config generators
│   ├── execution/            # HTTP bridge + live executor + deploy + manual export
│   └── persistence/          # Project save/load
├── shared/
│   └── templates/            # 8 Jinja2 IOS config templates (.j2)
├── server.py                  # MCP server entry point
├── settings.py                # Version + config (v0.5.0)
└── __main__.py                # python -m entry point
```

### Data Flow

```
TopologyRequest → Orchestrator → IPPlanner → Validator → AutoFixer
                                                              ↓
                                              TopologyPlan (validated)
                                                              ↓
                                    ┌─────────────────────────┼──────────────────┐
                                    ↓                         ↓                  ↓
                            PTBuilder Script            CLI Configs         Live Deploy
                           (addDevice/addLink)      (hostname, IPs,     (HTTP bridge
                                                     DHCP, routing)      → PT real-time)
```

### Why port 39000?

The server uses **streamable-http** instead of stdio. This means it runs once as a persistent HTTP process and MCP clients connect to it over the network.

**Advantages over stdio:**
- **Persistence** — the server stays running, not restarted with each editor session
- **Multiple clients** — VS Code, Claude Desktop, and other clients can connect to the same server simultaneously
- **Shared state** — the HTTP bridge to Packet Tracer stays active between requests
- **Easier debugging** — you can `curl` the server and see logs in the terminal
- **Decoupling** — the server is independent of the editor lifecycle

Port 39000 was chosen to avoid collisions with common ports (3000, 5000, 8000, 8080) and the internal PT bridge port (54321).

---

## PTBuilder Extension

The `PTBuilder/` directory contains the source of the "Builder Code Editor" Script Module:

| File | Purpose |
|------|---------|
| `source/main.js` | Entry point — creates menu and webview |
| `source/runcode.js` | `runCode(scriptText)` — executes JS in the Script Engine |
| `source/userfunctions.js` | `addDevice()`, `addLink()`, `configureIosDevice()`, `configurePcIp()`, `queryTopology()`, `deleteDevice()`, `renameDevice()`, `moveDevice()`, `deleteLink()` |
| `source/devices.js` | Model → PT numeric type mapping |
| `source/links.js` | Cable type → PT numeric ID mapping |
| `source/modules.js` | Hardware module mapping |
| `source/window.js` | Webview window management (QWebEngine) |
| `source/interface/` | HTML + JS for the web editor (status panel + real-time logging) |
| `Builder.pts` | Compiled extension package (binary) |

### PTBuilder API

```javascript
// Create a device at coordinates (x, y)
addDevice("R1", "2911", 100, 200);

// Create a link between two devices
addLink("R1", "GigabitEthernet0/1", "S1", "GigabitEthernet0/1", "straight");

// Configure a router/switch with IOS CLI commands
configureIosDevice("R1", [
  "enable",
  "configure terminal",
  "hostname R1",
  "interface GigabitEthernet0/0",
  "ip address 192.168.0.1 255.255.255.0",
  "no shutdown"
].join("\n"));

// Configure a PC with a static IP
configurePcIp("PC1", false, "192.168.0.2", "255.255.255.0", "192.168.0.1");

// Configure a PC for DHCP
configurePcIp("PC1", true);
```

---

## Testing

```bash
# All tests (129 tests across 15 files)
python -m pytest tests/ -v

# Single test file
python -m pytest tests/test_full_build.py -v

# Specific test
python -m pytest tests/test_full_build.py::TestFullBuild::test_basic_2_routers -v

# With coverage
python -m pytest tests/ --cov=src/packet_tracer_mcp --cov-report=term-missing
```

Test coverage includes: IP planning, validation, auto-fix, explanation, estimation, code generation, full-build integration (8 scenarios), RIP and EIGRP routing (14 tests), project persistence, resource/catalog validity, topology intelligence (20 tests), Jinja2 config templates (11 tests), scenario presets, validation upgrades (12 tests), and bridge recovery (4 tests).

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PT_MCP_PORT` | `39000` | MCP server HTTP port |
| `PT_MCP_LOG_LEVEL` | `INFO` | Logging level (`DEBUG`, `INFO`, `WARNING`, `ERROR`) |

---

## Requirements

- Python 3.11+
- Cisco Packet Tracer 8.2+ (for live deploy)
- PTBuilder extension installed in PT (included in `PTBuilder/`)

---

## License

MIT
