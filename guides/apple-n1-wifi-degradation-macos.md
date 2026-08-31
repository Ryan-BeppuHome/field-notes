# Mac Wi-Fi is unusably slow after switching networks

**Who this is for:** you have a recent MacBook (an M5, from 2026). Wi-Fi works, then you go out, use
your phone's hotspot, come back — and now Wi-Fi is so slow it's useless. Restarting the Mac fixes it.
Turning Wi-Fi off and on doesn't.

This is a bug in Apple's Wi-Fi software. **You don't have to restart your Mac.** There's a faster fix,
and you can make it happen automatically.

---

## The quick fix

Paste this into Terminal. It takes about 15 seconds.

```bash
sudo pkill -9 -f AppleCentauri; sudo pkill -9 -x centaurid; sudo pkill -9 -x airportd
sleep 12
sudo networksetup -setairportpower en0 on
```

This restarts just the Wi-Fi software, instead of the whole Mac. In a measured test, a web request
that was taking **15.8 seconds dropped to 0.09 seconds** straight afterwards.

If it doesn't work, the worst case is that Wi-Fi stays off until you restart the Mac — which is where
you already were. Plug in an Ethernet cable first if you want a safety net.

Skip to [Make it automatic](#make-it-automatic) if you just want it handled from now on.

---

## Is this actually your problem?

This bug looks exactly like a broken router, so it's worth being sure. Tick these off:

- [ ] Your Mac is an **M5** (2026 or later). These are the first Macs with Apple's own Wi-Fi chip,
      called the **N1**. Check: Apple menu → About This Mac.
- [ ] It started **right after you switched networks** — not gradually.
- [ ] **Restarting the Mac fixes it. Turning Wi-Fi off and on doesn't.**
- [ ] Web pages take **many seconds** to load, but the Wi-Fi icon shows full bars.
- [ ] **An Ethernet cable works perfectly** on the same Mac.
- [ ] **Other devices on the same Wi-Fi are fine** at the same moment.

The third one is the giveaway. Broken hardware doesn't fix itself for a while after a restart and
then break again. Software that gets stuck does exactly that.

### You can trigger it on purpose

Useful if you want to test a fix rather than wait around:

1. Connect to your normal Wi-Fi.
2. Switch to your phone's hotspot.
3. Switch back.

It breaks every time.

---

## First: prove it isn't your router

**Do this before changing any router settings.** Take a second device — another laptop, a phone — put
it on the same Wi-Fi, and load a website on both at the same time.

If the other device is fine and your Mac isn't, **your network is not the problem** and no router
setting will help. Skipping this test cost me an entire evening of changing router settings that
couldn't possibly have mattered.

### Measuring it properly

To put numbers on it, paste this into Terminal:

```bash
for i in 1 2 3 4 5; do
  curl -s --interface en0 -o /dev/null -w '%{time_total}\n' https://www.google.com/generate_204
done
```

It fetches a tiny page five times and prints how many seconds each took.

- **Healthy: about 0.08 seconds.**
- **Broken: 1 to 25 seconds.**

There's nothing in between, which is handy — you never have to guess which state you're in.

### A second check

This one reads Apple's own Wi-Fi log:

```bash
log show --last 10m --predicate 'process == "airportd"' \
  | grep -oE "channel = [0-9]+ BW = [0-9]+ band = [0-9]+" | sort -u
```

Each line is one radio setting your Mac has been using. **One line means healthy. Four means broken**
— it's flipping between settings instead of settling on one.

> **One thing that looks like a clue but isn't:** the log sometimes shows the connection speed
> collapsing to `rxRate=6.0Mbps`. That happened in 22% of readings during one failure — and in **zero
> of 73 readings** during the next one. Don't trust it.

---

## Why restarting the Wi-Fi software works

Wi-Fi drivers used to be buried deep in macOS, and the only way to reload one was to restart the Mac.
On these new Macs that changed. The Wi-Fi driver is now five ordinary background programs:

```bash
ps -Ao pid,user,comm | grep -E "AppleCentauri|centaurid|airportd"
```

```
AppleCentauriAlpha      AppleCentauriControl      AppleCentauriBeta
/usr/libexec/centaurid  /usr/libexec/airportd
```

("Centauri" is Apple's internal name for the N1 chip.)

Because they're ordinary programs, **you can quit them and macOS immediately starts them again,
fresh.** That's all the fix does. It's the same thing restarting your Mac achieves, without the
restart.

The driver appears to hold on to something from your old network that it shouldn't. Restarting it
throws that away.

### Why "turn Wi-Fi off and on" never helps

The part that gets stuck is the driver. Turning Wi-Fi off and on — and even restarting `airportd`,
the program that manages Wi-Fi — leaves the driver running untouched. Measured, against the same
live failure:

| What I tried | Page load after |
|---|---|
| nothing (broken) | 15.8 seconds |
| turned off AirDrop's background scanning | 12.1 seconds |
| restarted the Wi-Fi manager (`airportd`) | 14.0 seconds |
| turned Wi-Fi off and back on | 14.8 seconds |
| **restarted the driver** | **0.09 seconds** |

---

## Make it automatic

Since the bug appears when you change networks, a small background job can watch for exactly that
moment, check whether Wi-Fi is broken, and fix it if so. No restart, no Terminal.

**It only acts when there's a real problem.** It measures first, and if Wi-Fi is fine it does nothing
at all.

### Step 1 — create the script

Save this as `/usr/local/bin/wifi-autoheal.sh`:

```bash
#!/bin/bash
# Runs when the network changes. Checks Wi-Fi, and restarts the driver only if it's broken.
LOG=/var/log/wifi-autoheal.log
STAMP=/var/run/wifi-autoheal.last
COOLDOWN=600     # never fix more than once per 10 minutes
BAD=1.5          # page load slower than this = broken (healthy is ~0.08)
T0=$(date +%s)
say(){ echo "$(date '+%F %H:%M:%S') [+$(( $(date +%s) - T0 ))s] $*" >> "$LOG"; }

if [ -f "$STAMP" ]; then
  [ $(( $(date +%s) - $(stat -f %m "$STAMP") )) -lt "$COOLDOWN" ] && exit 0
fi

wait_for_ip(){ local n=0; while [ $n -lt "$1" ]; do [ -n "$(ipconfig getifaddr en0)" ] && return 0; sleep 1; n=$((n+1)); done; return 1; }
wait_for_drivers(){ local n=0; while [ $n -lt "$1" ]; do
    [ "$(pgrep -f AppleCentauri | wc -l | tr -d ' ')" -ge 3 ] && return 0; sleep 1; n=$((n+1)); done; return 1; }

wait_for_ip 25 || exit 0                                   # Wi-Fi not connected - nothing to do
[ "$(route -n get default 2>/dev/null | awk '/interface:/{print $2}')" != "en0" ] && exit 0  # on Ethernet

median(){ local t=(); for i in 1 2 3; do
    t+=("$(curl -s --interface en0 -o /dev/null -w '%{time_total}' --max-time 4 \
        https://www.google.com/generate_204 2>/dev/null || echo 4)")
  done; printf '%s\n' "${t[@]}" | sort -n | sed -n '2p'; }
bad(){ awk -v m="$1" -v b="$BAD" 'BEGIN{exit !(m+0>b+0)}'; }

M=$(median)
bad "$M" || exit 0                                          # Wi-Fi is fine - do nothing, say nothing

say "BROKEN on $(ipconfig getifaddr en0): ${M}s - restarting Wi-Fi driver"
touch "$STAMP"
pkill -9 -f AppleCentauri; pkill -9 -x centaurid; pkill -9 -x airportd
wait_for_drivers 15
networksetup -setairportpower en0 on >/dev/null 2>&1
wait_for_ip 15
A=$(median)
if bad "$A"; then say "STILL BROKEN after restart (${A}s) - you may need to reboot"
else say "FIXED: ${M}s -> ${A}s"; fi
```

### Step 2 — create the trigger

Save this as `/Library/LaunchDaemons/com.n1wifi.autoheal.plist`:

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

macOS rewrites those two files every time you change networks. Watching them is what makes the script
run at the right moment, and never otherwise.

### Step 3 — turn it on

```bash
sudo chown root:wheel /usr/local/bin/wifi-autoheal.sh /Library/LaunchDaemons/com.n1wifi.autoheal.plist
sudo chmod 755 /usr/local/bin/wifi-autoheal.sh
sudo chmod 644 /Library/LaunchDaemons/com.n1wifi.autoheal.plist
sudo launchctl bootstrap system /Library/LaunchDaemons/com.n1wifi.autoheal.plist
```

That's it. It starts automatically every time you turn the Mac on.

### Checking on it

```bash
sudo tail /var/log/wifi-autoheal.log
```

**An empty log is good** — it only writes something when it actually fixed a problem. A real entry
looks like:

```
2026-08-31 20:13:00 BROKEN on 192.168.1.62: 8.03s - restarting Wi-Fi driver
2026-08-31 20:13:22 FIXED: 8.03s -> 0.078s
```

**Keep the script in `/usr/local/bin`, owned by root** — not in your home folder. A background job
running as an administrator, from a file you can edit without a password, is a way in for anything
else on your Mac.

---

## Things that don't work

All tested and ruled out. Several are the top answers you'll find by searching, which is why they're
here:

- **Anything on your router.** Another Mac on the same Wi-Fi at the same moment was perfect. Channel
  width, band steering, beamforming, restarting the router — none of it made any difference.
- **Buying a new Mac.** It has the same chip and the same driver. And the fact that a restart fixes it
  proves your hardware is fine.
- **Updating macOS.** No Wi-Fi fix in 26.6 or 26.6.2. Apple's one N1 Wi-Fi fix was in 26.4.1, for a
  different problem entirely.
- **Turning off Bluetooth.** The log showed Bluetooth demanding the radio thousands of times and
  being refused every time — it looked like a smoking gun. Turning it off changed nothing.
- **Forgetting and rejoining the network**, changing DNS, VPN software, Low Power Mode, Private Wi-Fi
  Address.

Other people have hit this and got nowhere:
[report 1](https://discussions.apple.com/thread/256272249) (that person went as far as reinstalling
macOS completely) and [report 2](https://discussions.apple.com/thread/256315041).

---

## If you write your own self-healing script

Worth reading even if you never touch a Mac. My first attempt at this **made everything worse**, and
the reasons apply to any automatic repair job:

- It ran **every 60 seconds** — about 1,400 times a day.
- It gave up after **6 seconds**, on a connection that legitimately took up to 25. So it declared
  failure on connections that were slow but working.
- When it "failed", it **reset the network connection**, which changed the Mac's address and cut off
  everything in progress.

Those three together made a loop: a false alarm broke the connection, the breakage caused more
slowness, which caused the next false alarm. Worse, it **wrote its guesses into its own log as
facts** — recording "cause: VPN" simply because that was the step running when the problem happened
to clear. I then spent an hour chasing that VPN.

So:

- **Trigger on the event, not on a timer.** If you can name what causes the fault, watch for that.
- **Only automate a signal with a huge gap.** Here, healthy is 0.08 seconds and broken is 15 — nearly
  200 times apart. If your good and bad states are close together, you'll fire on the good ones.
- **Do one thing, and make it harmless.** Don't reset connections or change settings.
- **Do nothing and say nothing when everything is fine.**
- **Limit how often it can act.**
- **Never let it record a cause.** "The problem went away during step 3" is not proof step 3 fixed it.

---

## Small print

Tested on macOS 26.5.2 and 26.6, on an M5 MacBook Air, on a Wi-Fi 6E mesh network. The bug is in the
Mac, not any particular router — it happened on a phone hotspot too.

**This works around the problem; it doesn't fix it.** Apple needs to. If this is happening to you,
tell them at <https://www.apple.com/feedback/macos.html>. Mention that you can reproduce it by
switching networks, include your before-and-after timings, and say that another device on the same
Wi-Fi is fine. That's far more useful than "my Wi-Fi is slow", and it's more than the existing public
reports contain.
