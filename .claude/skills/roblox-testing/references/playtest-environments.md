# Playtest Environments

Network, device, Team Test and real-player checks. Belongs to the roblox-testing skill.

## Network and device

**GA per the announcement.** The new Device Simulator toolbar and Network Simulator
left beta on 2026-10-16 (DevForum 4861695). The docs snapshot of 2026-10-29 still
shows a beta alert on both pages, which conflicts. If the toolbar is missing, look
under File > Beta Features > New Device Simulator.

- Open **Device Simulator** from the Test menu, then the **Network** pill. Set
  profile or inbound/outbound latency, jitter and packet loss.
- Values **add to** the real connection, which matters in Team Test. Local Studio
  playtests only, never live players.
- Device Simulator covers screen size, scaling and input. It does **not** simulate
  device performance or memory; the old memory setting was a placeholder.
- Run every server-confirmed action under a bad-mobile profile: is the UI stuck?
  Can it double-fire? Server Authority games need this most (see
  roblox-server-authority). A question on whether `Player:GetNetworkPing` reflects
  simulated latency was unanswered in that thread; don't rely on it.

## Real hardware

Emulation doesn't reproduce frame rate, memory pressure, thermal throttling or
input latency (`performance-optimization/test-on-hardware`).

- Android is roughly 65% of a typical player base, and about 60% of those devices
  have 2–4 GB of RAM. Pick at least one low-end baseline device.
- On device: F9 console **Memory** tab, MicroProfiler (Ctrl+Alt+F6 / Cmd+Opt+F6),
  and Performance Stats.
- Run 10–15 minutes of active play. A steady frame rate decline suggests thermal
  throttling.
- Confirm network behavior over real cellular or constrained Wi-Fi.

## Team Test and the debugger

Team Test runs on a real Roblox server, one session at a time.

| Feature | Status | Notes |
| --- | --- | --- |
| Breakpoint halts all Luau threads | Live (706) | Timers and other coroutines stop too, so state stays consistent. Debugger can't be disabled (709) |
| Server output in Team Test | Live fix (713) | |
| **Remote Breakpoints** | Studio Beta: File > Beta Features > Remote Breakpoints | Server scripts in Team Test; only for the user who started the session |
| **Server Inspection** | Studio Beta (2026-10-30): File > Beta Features > Server Inspection | Live server viewport, instances, properties, scripts, breakpoints |

Server Inspection limits (DevForum 4966700): cloned, asset-loaded and runtime-made
scripts can show empty, with a fix for some announced; shown source is from session
start; edits are accepted but discarded when the session stops; one debugger per
server; breakpoints in server startup code may not fire; only ServerScriptService
and ServerStorage among server-only services. Never conclude a script is empty or a
fix took effect from this view alone.

## Real players, before launch

- **Playtest audience and early testers** (DevForum 4930524, 2026-10-12, rolling
  out over "the coming weeks"; check that your account has it): up to 10 friends,
  or 10 Group members for Group games. Collaborate > Early testers in Studio. An
  invite can be blocked by age check, trust, parent settings or region; check those
  before debugging.
- **Experiments** answer "is B better?", not "does it work?". Config-driven via
  `ConfigService:GetConfigForPlayerAsync`, 14–60 days, up to two variants plus a
  control, one matchmaking experiment at a time. Under 1,000 daily users, useful
  data is unlikely. Run full duration; don't launch on early harm results or a
  Sample Ratio Mismatch.

Last verified: 2026-11-01 (Studio 0.745, release notes 745). Sources are listed in `../SKILL.md`.
