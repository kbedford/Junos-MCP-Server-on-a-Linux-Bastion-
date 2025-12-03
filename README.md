# Junos-MCP-Server-on-a-Linux-Bastion-
Centralised Junos CLI via MCP + HTTP*


# Junos MCP Server on a Linux Bastion  

**Centralised Junos CLI via MCP + HTTP**

This README describes how to set up the [Junos MCP Server](https://github.com/Juniper/junos-mcp-server) on a Linux host (typically a bastion/jump server) so operations teams can:

- Send **Junos CLI commands** to routers via MCP  
- Use **HTTP + JSON** (or a simple shell helper) to get live output  
- Avoid SSHing to each router individually

This document **does not** cover any editor/IDE integrations — it focuses purely on running everything directly on the Linux host.

---

## 1. Architecture Overview

- **Bastion host** (Linux):
  - Runs the Junos MCP Server (`jmcp.py`) inside a Python virtual environment.
  - Exposes an HTTP MCP endpoint locally on `127.0.0.1:30030`.
  - Provides a shell helper `jmcp_cli` for ops engineers to run Junos CLI commands via MCP.

- **Junos routers**:
  - Reachable from the bastion (e.g. management IPs).
  - Defined in a `devices.json` file with credentials and connection details.

High-level flow:

1. Ops engineer SSHes to the bastion.
2. Runs `jmcp_cli "show bgp summary | no-more"`.
3. Bastion sends an MCP `tools/call` request over HTTP to the MCP server.
4. MCP server connects to the Junos router, runs the CLI command, and returns the output.
5. The helper function prints the CLI text back to the operator.



## 2. Prerequisites

On the bastion host you will need:

- Linux (example below uses Ubuntu 22.04)
- Python **3.11**
- `python3-pip` and `python3.11-venv`
- `git`
- `curl`
- `jq` (used to parse JSON and extract CLI output)

Install core tools (adjust for your environment/repositories as needed):

```bash
apt update
apt install -y \
  git \
  python3.11 \
  python3-pip \
  python3.11-venv \
  curl \
  jq
````

> If your environment uses internal mirrors (e.g. `repo.juniper.net`), ensure `apt` is configured accordingly.

---

## 3. Clone the Junos MCP Server Repository

Choose a directory for the tool, e.g. `/root`:

```bash
cd /root
git clone https://github.com/Juniper/junos-mcp-server.git
cd junos-mcp-server

ls
# CLAUDE.md  Dockerfile  jmcp.py  jmcp_token_manager.py  pyproject.toml
# devices-template.json  requirements.txt  ...
```

---

## 4. Create and Activate a Python Virtual Environment

Create an isolated environment for the MCP server:

```bash
cd /root/junos-mcp-server
python3.11 -m venv .venv
source .venv/bin/activate

which python
# /root/junos-mcp-server/.venv/bin/python
```

Upgrade `pip` (optional but recommended):

```bash
python -m pip install --upgrade pip
```

---

## 5. Install MCP Server Dependencies

With the virtual environment active:

```bash
cd /root/junos-mcp-server
pip install -r requirements.txt
```

If you later see errors like:

```text
ModuleNotFoundError: No module named 'pydantic'
```

it usually means you are not in the virtual environment. Just re-run:

```bash
cd /root/junos-mcp-server
source .venv/bin/activate
```

---

## 6. Configure Devices (`devices.json`)

Start with the provided template:

```bash
cd /root/junos-mcp-server
cp devices-template.json devices.json
```

Edit `devices.json`:

```bash
vi devices.json
```

Example for a single router:

```json
{
  "vmx101": {
    "ip": "10.38.171.252",
    "port": 22,
    "username": "ken",
    "auth": {
      "type": "password",
      "password": "junos123"
    }
  }
}
```

Replace with your own details:

* `"ip"` – management IP/hostname of the Junos device
* `"port"` – SSH port (usually `22`)
* `"username"` / `"password"` – Junos login with appropriate privileges

You can define multiple routers as:

```json
{
  "vmx101": { ... },
  "vmx102": { ... },
  "core-mx1": { ... }
}
```

---

## 7. Start the MCP Server (HTTP / Streamable)

With the venv active:

```bash
cd /root/junos-mcp-server
source .venv/bin/activate

python jmcp.py -f devices.json -t streamable-http -H 127.0.0.1
```

You should see logs similar to:

```text
WARNING - No .tokens file found - server is open to all clients
INFO    - All 1 device(s) validated successfully
INFO    - Successfully loaded and validated 1 device(s)
INFO    - Streamable HTTP server started on http://127.0.0.1:30030
Uvicorn running on http://127.0.0.1:30030 (Press CTRL+C to quit)
```

Leave this process running in that terminal.

> 🔐 **Security note:**
> Without a `.tokens` file, any process that can reach `127.0.0.1:30030` can talk to this MCP server.
> On a single-user bastion, this is usually fine. In shared environments, consider configuring tokens with `jmcp_token_manager.py`.

---

## 8. Initialize MCP Session and Capture the Session ID

Open a **second** SSH session to the same bastion host.

Run the MCP `initialize` call:

```bash
curl -i \
  -X POST "http://127.0.0.1:30030/mcp/" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2024-11-05",
      "capabilities": {},
      "clientInfo": { "name": "curl", "version": "1.0" }
    }
  }'
```

You should see a response like:

```http
HTTP/1.1 200 OK
date: ...
server: uvicorn
content-type: text/event-stream
mcp-session-id: f98a307fce164ec2af8c49e43b1fc664
...

event: message
data: {"jsonrpc":"2.0","id":1,"result":{...}}
```

The key part is the header:

```http
mcp-session-id: f98a307fce164ec2af8c49e43b1fc664
```

Export this into your shell as an environment variable:

```bash
export MCP_SESSION=f98a307fce164ec2af8c49e43b1fc664
echo "$MCP_SESSION"
# f98a307fce164ec2af8c49e43b1fc664
```

> 🔁 **Important:**
> Every time you restart `jmcp.py`, you must:
>
> 1. Call `initialize` again, and
> 2. Update `MCP_SESSION` with the new `mcp-session-id`.

---

## 9. Create a `jmcp_cli` Helper Function

To make the MCP server easy to use for ops teams, add a shell function that:

* Sends MCP `tools/call` requests via `curl`
* Strips the `data:` prefix from Server-Sent Events (SSE)
* Uses `jq` to extract just the Junos CLI text

What this function does

At a high level, jmcp_cli:

1. Takes a Junos CLI command as arguments

  * e.g. jmcp_cli "show bgp summary | no-more"
  * The full string is stored in cmd and passed to the MCP server.

2. Ensures the command and MCP session are valid

   * If you forget to pass a command, it prints:
      Usage: jmcp_cli <Junos command>

   * If MCP_SESSION is not set (you haven’t run the initialize call), it reminds you to do that.

 3. Calls the MCP server over HTTP

   * Sends a JSON-RPC tools/call request to http://127.0.0.1:30030/mcp/.
   * Uses the execute_junos_command tool for the router vmx101 (defined in devices.json).
   * Includes the Mcp-Session-Id: $MCP_SESSION header so the server knows which MCP session to use.

 4. Cleans up the response and prints only CLI text

   * The server responds as Server-Sent Events (SSE) with lines like event: ... and data: {...}.
   * sed strips the leading data: so we’re left with pure JSON.
   * jq picks out .result.content[0].text (the Junos CLI output) and prints it as normal text.

From an operator’s point of view, all of this is hidden. Once MCP_SESSION is set and jmcp_cli is loaded, they just run:
```
jmcp_cli "show chassis routing-engine | no-more"
jmcp_cli "show bgp summary | no-more"
jmcp_cli "show interfaces terse | no-more"
```

Edit your bash config:

```bash
vi ~/.bashrc
```

Append the following:

```bash
jmcp_cli() {
  local cmd="$*"

  if [ -z "$cmd" ]; then
    echo "Usage: jmcp_cli <Junos command>"
    return 1
  fi

  if [ -z "$MCP_SESSION" ]; then
    echo "Error: MCP_SESSION not set. Run the MCP initialize curl and export MCP_SESSION first."
    return 1
  fi

  curl -s "http://127.0.0.1:30030/mcp/" \
    -H "Content-Type: application/json" \
    -H "Accept: application/json, text/event-stream" \
    -H "Mcp-Session-Id: $MCP_SESSION" \
    -d "{
      \"jsonrpc\": \"2.0\",
      \"id\": 10,
      \"method\": \"tools/call\",
      \"params\": {
        \"name\": \"execute_junos_command\",
        \"arguments\": {
          \"router_name\": \"vmx101\",
          \"command\": \"${cmd}\"
        }
      }
    }" \
    | sed -n 's/^data: //p' \
    | jq -r '.result.content[0].text // .data.result.content[0].text'
}
```
Example output below 

```
root@ansible:~# curl -i \
  -X POST "http://127.0.0.1:30030/mcp/" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2024-11-05",
      "capabilities": {},
      "clientInfo": { "name": "curl", "version": "1.0" }
    }
  }'
HTTP/1.1 200 OK
date: Wed, 03 Dec 2025 16:04:58 GMT
server: uvicorn
cache-control: no-cache, no-transform
connection: keep-alive
content-type: text/event-stream
mcp-session-id: f98a307fce164ec2af8c49e43b1fc664
x-accel-buffering: no
Transfer-Encoding: chunked

event: message
data: {"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2024-11-05","capabilities":{"experimental":{},"prompts":{"listChanged":false},"resources":{"subscribe":false,"listChanged":false},"tools":{"listChanged":false}},"serverInfo":{"name":"jmcp-server","version":"1.0.0"}}}


```

Reload your shell configuration:

```bash
source ~/.bashrc
type jmcp_cli
# jmcp_cli is a function
```

> 📝 The example above is hard-coded to `router_name: "vmx101"`.
> If you have multiple routers, you can create a `jmcp_cli_r <router> "<cmd>"` variant later.

---

## 10. Using `jmcp_cli` to Run Junos CLI Commands

Once:

* `jmcp.py` is running on the bastion
* `MCP_SESSION` is set
* `jmcp_cli` is loaded in your shell

…you can run commands like this:

Example output:

```bash
root@ansible:~# jmcp_cli "show chassis routing-engine | no-more"

Routing Engine status:
  Slot 0:
    Current state                  Master
    Election priority              Master (default)
    DRAM                      4042 MB (4096 MB installed)
    Memory utilization          11 percent
    5 sec CPU utilization:
      User                       1 percent
      Background                 0 percent
      Kernel                     1 percent
      Interrupt                  1 percent
      Idle                      97 percent
    1 min CPU utilization:
      User                       1 percent
      Background                 0 percent
      Kernel                     2 percent
      Interrupt                  1 percent
      Idle                      96 percent
    5 min CPU utilization:
      User                       1 percent
      Background                 0 percent
      Kernel                     2 percent
      Interrupt                  1 percent
      Idle                      96 percent
    15 min CPU utilization:
      User                       1 percent
      Background                 0 percent
      Kernel                     2 percent
      Interrupt                  1 percent
      Idle                      96 percent
    Model                          RE-VMX
    Serial ID                      907311d8-cf
    Start time                     2025-12-02 06:49:05 PST
    Uptime                         1 day, 1 hour, 33 minutes, 3 seconds
    Last reboot reason             Router rebooted after a normal shutdown.
    Load averages:                 1 minute   5 minute  15 minute
                                       0.06       0.17       0.16

root@ansible:~# jmcp_cli "show bgp summary "

Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 1 Peers: 6 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0               
                      36         36          0          0          0          0
bgp.l3vpn.0          
                      12         12          0          0          0          0
bgp.l2vpn.0          
                       4          4          0          0          0          0
bgp.evpn.0           
                       0          0          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
11.0.0.102               11       3413       3412       0       0  1d 1:56:26 Establ
  inet.0: 6/6/6/0
  bgp.l3vpn.0: 4/4/4/0
  bgp.l2vpn.0: 0/0/0/0
  bgp.evpn.0: 0/0/0/0
  vpn_0.inet.0: 4/4/4/0
  l2vpn_3.l2vpn.0: 0/0/0/0
  vpn_200.l2vpn.0: 0/0/0/0
11.0.0.103               11       3385       3382       0       0  1d 1:42:55 Establ
  inet.0: 6/6/6/0
  bgp.l3vpn.0: 4/4/4/0
  bgp.l2vpn.0: 1/1/1/0
  bgp.evpn.0: 0/0/0/0
  vpn_0.inet.0: 4/4/4/0
  l2vpn_3.l2vpn.0: 0/0/0/0
  vpn_200.l2vpn.0: 1/1/1/0
11.0.0.104               11       3385       3381       0       0  1d 1:42:37 Establ
  inet.0: 6/6/6/0
  bgp.l3vpn.0: 4/4/4/0
  bgp.l2vpn.0: 2/2/2/0
  bgp.evpn.0: 0/0/0/0
  vpn_0.inet.0: 4/4/4/0
  l2vpn_3.l2vpn.0: 1/1/1/0
  vpn_200.l2vpn.0: 1/1/1/0
11.0.0.105               11       3411       3413       0       0  1d 1:57:31 Establ
  inet.0: 6/6/6/0
  bgp.l2vpn.0: 0/0/0/0
  bgp.evpn.0: 0/0/0/0
  l2vpn_3.l2vpn.0: 0/0/0/0
  vpn_200.l2vpn.0: 0/0/0/0
11.0.0.106               11       3377       3377       0       0  1d 1:41:19 Establ
  inet.0: 6/6/6/0
  bgp.l2vpn.0: 1/1/1/0
  bgp.evpn.0: 0/0/0/0
  l2vpn_3.l2vpn.0: 0/0/0/0
  vpn_200.l2vpn.0: 1/1/1/0
11.0.0.107               11       3377       3379       0       0  1d 1:42:05 Establ
  inet.0: 6/6/6/0
  bgp.l3vpn.0: 0/0/0/0
  bgp.l2vpn.0: 0/0/0/0
  l2vpn_3.l2vpn.0: 0/0/0/0
  vpn_200.l2vpn.0: 0/0/0/0


```

From the operator’s point of view, they simply:

1. SSH to the bastion
2. Ensure `MCP_SESSION` is set (or run the wrapper that does it)
3. Use `jmcp_cli "<Junos command>"` to interact with the router

---

---

## 11. Summary

This setup turns a Linux bastion into a **central MCP gateway for Junos**:

* **Single SSH entry point** for ops teams
* **HTTP + JSON** northbound interface (MCP) to the Junos estate
* Simple `jmcp_cli` shell helper so operators can:

  * Avoid direct SSH to every router
  * Run standard show commands
  * Build small scripts/health checks on top of MCP

This README intentionally stops at the CLI + HTTP layer.
The same MCP server can later be integrated with IDEs, chat agents, or other tooling, but the core pattern of “bastion MCP server + `jmcp_cli`” already provides a powerful operational workflow.

```
::contentReference[oaicite:0]{index=0}
```
