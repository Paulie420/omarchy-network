# Omarchy Network (self-verifying)

A clone of [Omarchy](https://omarchy.org)'s built-in Wi-Fi/network bar widget
that **verifies** a "Limited internet access" verdict instead of just
repainting it — so a VPN killswitch's false alarm shows completely normal,
not a warning color, while a real outage still shows red.

## The bug this fixes

NetworkManager decides "limited vs full" by fetching
`http://ping.archlinux.org/nm-check.txt` **bound directly to the physical
device** (`wlp170s0` here), independent of whatever route is actually
carrying your traffic. A VPN killswitch that uses policy routing to capture
traffic sourced from that device's IP — PIA does this — catches
NetworkManager's own probe along with everything else and refuses it. So
NetworkManager's device-bound check genuinely fails, and the stock widget
just mirrors that faithfully — even though every other request the machine
makes goes out fine through the tunnel.

Reproducible directly:

```sh
# Fails while a killswitch VPN is connected — this is what NetworkManager's own probe does
curl --interface wlp170s0 http://ping.archlinux.org/nm-check.txt

# Works fine — this is what everything else on the machine actually gets
curl http://ping.archlinux.org/nm-check.txt
```

A second, independent bug compounds it: **`nmcli networking connectivity`
(without the `check` subcommand) only reads NetworkManager's cached
property — it does not force a fresh probe.** NetworkManager itself only
re-checks on device activation or certain internal trigger events, then
caches the result indefinitely. In testing this sat on a stale "full" for
20+ minutes through a full VPN disconnect/reconnect cycle, because nothing
happened to trigger a recheck. Only `nmcli networking connectivity check`
(or the equivalent D-Bus `CheckConnectivity()` call) forces a real probe.

## What this clone does differently

**First version of this fix inferred the cause** ("NetworkManager says
limited, and PIA happens to be connected, so it's probably that") and just
recolored the same red to amber. That's a guess, not an answer — it still
couldn't tell a real problem from a fake one if the guess was wrong, and it
only knew about PIA specifically.

**Current version verifies instead of guessing.** When NetworkManager
reports restricted, the widget fires its own unbound request to the exact
same URL/expected body NetworkManager itself checks — no `--interface`
binding, so it follows whatever route is actually active (through a VPN
tunnel or not, doesn't matter). If that confirms real connectivity works,
the widget shows **fully normal — no color, no label, nothing** — because
nothing is actually wrong. It only falls back to NetworkManager's red
"restricted" verdict when the independent check can't disprove it.

This generalizes past PIA to any VPN killswitch, any policy-routing setup,
anything that makes NetworkManager's device-bound probe lie about real
reachability — because it's checking the actual fact, not pattern-matching
on which app is running. It also no longer needs to know anything about
PIA, `piactl`, or any specific VPN client.

Second change, independent of the above: **it actively re-probes on a short
timer regardless of current state** (10s while NetworkManager already
reports restricted, 20s otherwise), instead of only polling while already
flagged restricted. The stock widget's poll only ever helps detect
*recovery* from a known-bad state; it never notices the initial onset faster
than NetworkManager's own infrequent internal triggers do.

Everything else — Wi-Fi scanning, connecting, the speed test, QR sharing,
captive portal handling — is untouched, straight from `omarchy.network`.

### How the verification works

| Property | What it is |
|---|---|
| `restricted` | NetworkManager's raw, unmodified verdict (`hasCaptivePortal \|\| connectivity === "limited"`) — untouched from upstream, still drives the active re-probing of NetworkManager itself |
| `ownProbeResult` | `"unknown"` / `"ok"` / `"fail"` — result of *this widget's own* unbound curl to the same check URL, run only while `restricted` is true |
| `realConnectivityOk` | `ownProbeResult === "ok"` |
| `displayRestricted` | `restricted && !realConnectivityOk` — the only thing that drives the icon color, tooltip, and panel label |

`ownProbeResult` resets to `"unknown"` whenever NetworkManager's own
`restricted` flag clears, so a result cached from one incident can never
silently paper over the start of a completely different one later.

Captive-portal sign-in ("SIGN-IN REQUIRED") is deliberately **not** gated by
this verification — a real portal redirect is treated as always
actionable, even if some traffic happens to be getting through another way,
since you may still need to sign in for the connection to be usable once a
tunnel isn't covering everything.

| State | Color | Meaning |
|---|---|---|
| normal signal icon | default | full connectivity, or NetworkManager said restricted but this widget's own probe confirmed real internet works |
| red, "Limited internet access" | red (`bar.urgent`) | NetworkManager says restricted and this widget's own independent probe could not disprove it — treat as a real outage |
| red, "Sign in to this network" | red | captive portal, unchanged from upstream |

## Install

```bash
omarchy plugin add https://github.com/Paulie420/omarchy-network.git --enable
```

This is a clone of the first-party `omarchy.network`, so adding it replaces
the built-in widget on your bar the same way `omarchy plugin clone
omarchy.network` would (existing keybinds/IPC calls to `omarchy.network`
route to it automatically — see Omarchy's plugin docs). No VPN-specific
configuration needed — the self-check works against NetworkManager's own
connectivity-check URL, whatever that's set to.

## Who made this

I'm paulie420. I run a homelab, a BBS, and [techheart.life](https://techheart.life),
and I put the builds and the debugging up on YouTube at
**[@techheart6090](https://youtube.com/@techheart6090)**.

## License

MIT
