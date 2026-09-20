# Omarchy Network (PIA-aware)

A clone of [Omarchy](https://omarchy.org)'s built-in Wi-Fi/network bar widget
that tells a VPN killswitch's false "Limited internet access" apart from an
actual outage — instead of painting both the same red.

## The bug this fixes

Whenever [PIA](https://www.privateinternetaccess.com/) is connected on this
machine, the stock network widget goes red and says "Limited internet
access" — even though real internet access is completely fine, just routed
through the tunnel.

Root cause: NetworkManager decides "limited vs full" by fetching
`http://ping.archlinux.org/nm-check.txt` **bound directly to the physical
interface** (`wlp170s0` here), independent of the normal default route. PIA's
killswitch runs as policy routing (`ip rule … lookup piavpnrt`) that captures
*any* traffic sourced from that interface's IP — including NetworkManager's
own probe — and refuses it. So NetworkManager's device-bound check genuinely
fails, and the stock widget just mirrors that faithfully. It isn't lying,
it's answering a slightly different question than "do I have internet."

Reproducible directly:

```sh
# Fails while PIA is connected — this is what NetworkManager's own probe does
curl --interface wlp170s0 http://ping.archlinux.org/nm-check.txt

# Works fine — this is what everything else on the machine actually gets
curl http://ping.archlinux.org/nm-check.txt
```

A second, independent bug compounds it: **`nmcli networking connectivity`
(without the `check` subcommand) only reads NetworkManager's cached
property — it does not force a fresh probe.** NetworkManager itself only
re-checks on device activation or certain internal trigger events, then
caches the result indefinitely. In testing this sat on a stale "full" for
20+ minutes through a full PIA disconnect/reconnect cycle, because nothing
happened to trigger a recheck. Only `nmcli networking connectivity check`
(or the equivalent D-Bus `CheckConnectivity()` call) forces a real probe.

## What this clone changes

- **Cross-checks `restricted` against `piactl get connectionstate`.** When
  NetworkManager reports limited/portal *and* PIA is connected, the icon and
  panel text go amber ("Limited (PIA killswitch)") instead of red — a
  distinct, lower-urgency signal that says "this is the known PIA artifact,
  not a real outage." A genuine outage (PIA off, or NM restricted for any
  other reason) still shows red, unchanged from upstream.
- **Actively re-probes on a short timer regardless of current state**
  (10s while restricted, 20s otherwise), instead of only polling while
  already flagged restricted. The stock widget's poll only ever helps detect
  *recovery* from a known-bad state; it never notices the initial onset
  faster than NetworkManager's own infrequent internal triggers do.

Everything else — Wi-Fi scanning, connecting, the speed test, QR sharing,
captive portal handling — is untouched, straight from `omarchy.network`.

| State | Color | Meaning |
|---|---|---|
| normal signal icon | default | full connectivity |
| amber, "Limited (PIA killswitch)" | amber | NM's device-bound probe failed, but PIA is connected — almost certainly the killswitch, not a real problem |
| red, "Limited internet access" | red (`bar.urgent`) | NM's probe failed and PIA is not the explanation — treat as a real outage |
| red, "Sign in to this network" | red | captive portal, unchanged from upstream |

## Install

This is a clone of a first-party Omarchy plugin, not a standalone install.
On an Omarchy system:

```sh
omarchy plugin clone omarchy.network
```

then replace the generated `~/.config/omarchy/plugins/<you>.network/Panel.qml`
and `manifest.json` with this repo's versions (or just `git clone` this repo
directly over that directory — it's a normal git checkout).

**PIA-specific:** `checkPia()` shells out to `/opt/piavpn/bin/piactl`
(PIA's default install path) — if you're on a different VPN client or a
different install path, that's the one function to adapt. Everything else
is generic NetworkManager/Quickshell behavior.

## License

MIT
