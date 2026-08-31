# Apple N1 Wi-Fi: "connected but unusably slow", and how to fix it without rebooting

On M5-era Macs — the first with Apple's in-house **N1** Wi-Fi chip — Wi-Fi degrades over time until
you reboot. This note shows how to confirm it's this fault and not your network, how to recover in
about 25 seconds instead of rebooting, and how to make that automatic.

Apple has shipped no fix as of macOS 26.6.2. This is **recovery, not a cure.**

---

## Is this your problem?

The signature is unusual, and it is worth checking carefully, because it looks exactly like a broken
router and isn't one.

- A tiny HTTPS request takes **1.2–25 seconds**. TCP connects fast (0.1–0.5 s), then the transfer stalls.
- **Ethernet on the same Mac is perfect** — about 0.07 s.
- **Another Mac on the same access point, at the same moment, is perfect** — about 0.08 s.
- Signal is excellent: −31 to −38 dBm, 2.1 Gbps link rate, clear channel, near-zero packet loss.
- It is pure latency *variance*: minimum ping 3 ms, maximum 500–750 ms.
- **A reboot cures it. Cycling the Wi-Fi radio does not.**

That last asymmetry is the diagnostic tell. A radio or antenna fault does not come good for hours
after a restart and then decay again — but driver state does.

Onset is associated with switching networks: tethering to a phone, then rejoining Wi-Fi later.
Frequency increases with uptime. In one measured case: four failures in a day, a reboot, five clean
days, then five failures on day six.

### Confirm it in one command

The reliable marker is the **number of distinct link configurations** the driver reports:

```bash
log show --last 10m --predicate 'process == "airportd"' \
  | grep -oE "channel = [0-9]+ BW = [0-9]+ band = [0-9]+" | sort -u
```

**One line = healthy. Four = wedged.** A healthy association sits on a single channel at a single
width. A wedged one thrashes between band/width combinations.

And measure the actual damage:

```bash
for i in 1 2 3 4 5; do
  curl -s --interface en0 -o /dev/null -w '%{time_total}\n' https://www.google.com/generate_204
done
```

Healthy is ~0.08 s. Wedged is 1.2–25 s. There is no ambiguous middle — the gap is roughly 150×,
which matters later when we set an automatic threshold.

> **A marker that does *not* work:** the downlink rate collapsing to `rxRate=6.0Mbps` in the airportd
> log. It looked compelling — 22% of samples in one failure, on a clear channel at −38 dBm. In the
> very next confirmed failure it was **0 of 73 samples**. Don't rely on it.

### Rule out your network first

Do this before touching anything. Put a **second machine on the same access point** and measure it
at the same moment. If the other machine is fine while yours is not, the network is exonerated and
no router setting will help you. This one test would have saved hours.

---

## What's actually wrong

The N1's Wi-Fi driver is **not** a kernel extension. It's three DriverKit processes running in
userspace, plus two daemons, all parented by `launchd`:

```bash
ps -Ao pid,user,comm | grep -E "AppleCentauri|centaurid|airportd"
```

```
/System/Library/DriverExtensions/AppleCentauriAlpha.dext/AppleCentauriAlpha
/System/Library/DriverExtensions/AppleCentauriControl.dext/AppleCentauriControl
/System/Library/DriverExtensions/AppleCentauriBeta.dext/AppleCentauriBeta
/usr/libexec/centaurid
/usr/libexec/airportd
```

That matters enormously. Old kernel extensions could not be reloaded without a reboot. These are
ordinary userspace processes — **kill them and launchd respawns them with clean state.** That is a
genuine driver restart, and it is what a reboot was doing for you all along.

(For the record: `AppleCentauriManager` reports version `1.0.0d1` — a first-revision driver. "Centauri"
is Apple's internal name for the N1 silicon.)

---

## The fix

```bash
sudo pkill -9 -f AppleCentauri
sudo pkill -9 -x centaurid
sudo pkill -9 -x airportd
sleep 12
sudo networksetup -setairportpower en0 on
```

Measured against a confirmed failure: **median fetch 15.82 s → 0.091 s.**

Worst case is that Wi-Fi stays down until you reboot — which is where you already were. Do it on
Ethernet the first time if you want a safety net.

### Things that did not work, measured

The cheaper remedies were tried first, against the same live failure, in order:

| Attempt | Median fetch after |
|---|---|
| baseline (wedged) | 15.82 s |
| `sudo ifconfig awdl0 down` — AirDrop/Handoff off-channel hopping | 12.07 s |
| restart `airportd` alone | 14.01 s |
| Wi-Fi radio off/on | 14.84 s |
| **restart the driver processes** | **0.091 s** |

Note that restarting `airportd` by itself does nothing. `airportd` is the Wi-Fi *daemon*; the fault
is in the *driver* beneath it. This is why every "turn Wi-Fi off and on again" suggestion fails.

---

## Dead ends, so you can skip them

All of these were tested and eliminated. Several are the top answers you'll find by searching, which
is exactly why they're listed:

- **The network itself** — a second Mac on the same access point at the same instant was perfect.
- **Channel width**, 160 MHz vs 80 MHz — no change; the control Mac ran fine at 160 MHz throughout.
- **Wi-Fi 7 MLO / EMLSR link switching** — a compelling theory, and wrong: the access point was
  Wi-Fi 6E, which has no MLO at all. *Check your router's actual model before building a theory on
  its features.*
- **Band steering and mesh roaming** — failures predated the mesh being enabled by six weeks.
- **Bluetooth coexistence** — the airportd log showed Bluetooth requesting the radio 1,000–3,000
  times per sample and being granted it zero times, every sample. Looked damning. Turning Bluetooth
  off changed nothing.
- **A VPN mesh client** (routes, split DNS) — no hijacked routes, no subnet routes advertised.
- **Duplicate IP address** — nothing else answered on the address.
- **MTU, Low Power Mode, Private Wi-Fi Address, beamforming, router reboot, forgetting and rejoining
  the network, content filters** (zero system extensions installed), **TCP sysctl tuning** (all stock).
- **Replacing the Mac** — pointless. A replacement carries the same chip and the same driver. And a
  reboot curing it proves the hardware is fine.
- **Updating macOS** — no Wi-Fi fix in 26.6 or 26.6.2. The only N1 Wi-Fi fix Apple has shipped was
  26.4.1, for 802.1X with content filter extensions — unrelated.

Related public reports, both unresolved:
[256272249](https://discussions.apple.com/thread/256272249) (that reporter went as far as a full
macOS reinstall) and [256315041](https://discussions.apple.com/thread/256315041).

---

## Making it automatic

Manual recovery is fine, but if the failure reliably follows a network change, automate it.

**`/usr/local/bin/wifi-autoheal.sh`** — root-owned, `chmod 755`:

```bash
#!/bin/bash
# Fires on NETWORK CHANGE (never a timer). Verifies, then restarts the N1 Wi-Fi
# driver only if genuinely wedged.
LOG=/var/log/wifi-autoheal.log
STAMP=/var/run/wifi-autoheal.last
COOLDOWN=600     # at most one heal per 10 min
BAD=1.5          # median secs above this = wedged (healthy ~0.08s)
T0=$(date +%s)
say(){ echo "$(date '+%F %H:%M:%S') [+$(( $(date +%s) - T0 ))s] $*" >> "$LOG"; }

if [ -f "$STAMP" ]; then
  [ $(( $(date +%s) - $(stat -f %m "$STAMP") )) -lt "$COOLDOWN" ] && exit 0
fi

wait_for_ip(){ local n=0; while [ $n -lt "$1" ]; do [ -n "$(ipconfig getifaddr en0)" ] && return 0; sleep 1; n=$((n+1)); done; return 1; }
wait_for_drivers(){ local n=0; while [ $n -lt "$1" ]; do
    [ "$(pgrep -f AppleCentauri | wc -l | tr -d ' ')" -ge 3 ] && return 0; sleep 1; n=$((n+1)); done; return 1; }

wait_for_ip 25 || exit 0                                   # no Wi-Fi IP -> not our business
[ "$(route -n get default 2>/dev/null | awk '/interface:/{print $2}')" != "en0" ] && exit 0  # on Ethernet

median(){ local t=(); for i in 1 2 3; do
    t+=("$(curl -s --interface en0 -o /dev/null -w '%{time_total}' --max-time 4 \
        https://www.google.com/generate_204 2>/dev/null || echo 4)")
  done; printf '%s\n' "${t[@]}" | sort -n | sed -n '2p'; }
bad(){ awk -v m="$1" -v b="$BAD" 'BEGIN{exit !(m+0>b+0)}'; }

M=$(median)
bad "$M" || exit 0                                          # healthy: silent, no action

say "WEDGED on $(ipconfig getifaddr en0): median ${M}s - restarting Wi-Fi driver"
touch "$STAMP"
pkill -9 -f AppleCentauri; pkill -9 -x centaurid; pkill -9 -x airportd
wait_for_drivers 15
networksetup -setairportpower en0 on >/dev/null 2>&1
wait_for_ip 15
A=$(median)
if bad "$A"; then say "STILL WEDGED after driver restart (${A}s) - reboot may be needed"
else say "CURED: ${M}s -> ${A}s"; fi
```

**`/Library/LaunchDaemons/com.n1wifi.autoheal.plist`** — root-owned, `chmod 644`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.n1wifi.autoheal</string>
  <key>ProgramArguments</key><array><string>/usr/local/bin/wifi-autoheal.sh</string></array>
  <key>WatchPaths</key>
  <array>
    <string>/etc/resolv.conf</string>
    <string>/Library/Preferences/SystemConfiguration/NetworkInterfaces.plist</string>
  </array>
  <key>RunAtLoad</key><false/>
  <key>StandardErrorPath</key><string>/var/log/wifi-autoheal.err</string>
</dict>
</plist>
```

Install:

```bash
sudo launchctl bootstrap system /Library/LaunchDaemons/com.n1wifi.autoheal.plist
```

Check it with `sudo tail /var/log/wifi-autoheal.log`. **Silence is the correct output** — it logs
only when it actually heals something.

Put the script in `/usr/local/bin` owned by root, not in your home folder. A root daemon running a
script you can write to is a privilege escalation path.

---

## Why the *first* version of this daemon made things worse

This is the part worth reading even if you never touch an N1 Mac.

An earlier attempt at the same idea actively generated the fault it was meant to fix:

- It ran **on a 60-second timer** — 1,440 firings a day instead of a handful.
- Its health check was a **6-second timeout on a link that legitimately took 1.2–25 seconds**, so it
  declared failure on connections that were merely slow *and working*.
- Its remedy was **destructive**: a DHCP reset that changed the machine's IP and killed every open
  connection.

So: a false alarm triggered a teardown, the teardown caused retransmits and stalls, which triggered
the next false alarm. It also **wrote false root-cause attributions into its own log** — recording
"root cause = VPN client" because that happened to be the step running when the link recovered on its
own. Those log lines then misdirected the investigation for an hour.

The rules that make the version above safe are worth stealing for any self-healing job:

- **Event-triggered, never polled.** Tie it to the thing that actually changes.
- **A threshold with a large margin.** 1.5 s against a healthy 0.08 s. If your healthy and unhealthy
  states aren't separated by an order of magnitude, you don't yet have a signal worth automating on.
- **One proven, non-destructive action.** Never touch DHCP, VPNs or network service configuration.
- **Do nothing, and say nothing, when healthy.**
- **Rate limit it**, so a bad day can't become a loop.
- **Never let a healer attribute causes.** "The fault cleared while step 3 was running" is not
  evidence that step 3 fixed it.

---

## Caveats

Tested on macOS 26.5.2 and 26.6.x, on an M5 MacBook Air, against a tri-band Wi-Fi 6E mesh. The fault
is in Apple's driver, not in any particular router — it was reproduced on a phone hotspot too.

The underlying defect is unfixed. If you hit this, report it at
<https://www.apple.com/feedback/macos.html>. A before/after measurement plus a control machine on the
same access point is far stronger evidence than the existing public reports contain.
