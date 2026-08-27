# Signage Watchtower

**Digital Signage & Mini-PC Fleet Health Monitor** — built around a real
operational problem: mini-PCs driving cinema signage TVs freeze, drop off
Wi-Fi, or get stuck on Windows update screens, and nobody notices until a
customer does.

## How it works

```
[agent.py on every mini-PC] --HTTP heartbeat every 60s--> [Node ingestion server] --WebSocket--> [React dashboard]
                                                                 |
                                                          [persistent store]
```

- **`agent/agent.py`** — lightweight background agent for each mini-PC.
  Collects CPU, RAM, CPU temp, and network latency via `psutil`, grabs a
  throttled screenshot, and POSTs a heartbeat. Never crashes if the server
  is briefly unreachable.
- **`dashboard/server/index.js`** — Express + WebSocket ingestion server.
  Upserts current screen state, appends history, and broadcasts live
  updates. A background sweep flags any screen that goes silent — so a
  dead PC is detected *without* needing a signal from it.
- **`dashboard/src/FleetDashboard.jsx`** — live status grid:
  green/amber/red per screen, CPU/RAM/ping, last-seen time, site filter.
- **`agent/simulate_agent.py`** — simulates a mini-PC as a real separate
  process posting real HTTP heartbeats, for load-testing the pipeline
  without hardware.

## Run it

```bash
# server
cd dashboard/server
npm install
node index.js          # listens on :4000

# simulated fleet (each is a real independent process)
cd ../../agent
pip install requests
python3 simulate_agent.py WAL-SCR-01
python3 simulate_agent.py WAL-SCR-02   # etc.
```

Then `curl http://localhost:4000/api/screens` for live state, or wire up
the React dashboard.

## Tested failure modes

Verified by running 8 independent agent processes against the live server:

- **Silent death** — killed an agent process mid-run; the server's own
  heartbeat-silence sweep flagged it `red / missed_heartbeat` within one
  detection cycle, with no signal from the dead process.
- **Network fault** — an agent with a flaky-network profile reported
  `ping=timeout`; correctly surfaced as a *distinct* failure
  (`network_unreachable`) from the silent-death case.
- **Thermal warning** — an overheat-prone agent tripped the temp
  threshold and was flagged `amber` while still reporting.

## Honest notes

The demo server uses a flat JSON file as its store; the schema is
designed to swap directly to PostgreSQL (`screens` current-state table +
`heartbeats` history table). `simulate_agent.py` generates its stats
randomly; `agent.py` is the production version that reads real hardware
via psutil.
