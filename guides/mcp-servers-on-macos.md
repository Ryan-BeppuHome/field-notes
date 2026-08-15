# MCP servers on macOS — setup guide

**Parts A–D: Google Workspace.** Claude can read and write Gmail, Drive, Docs, Sheets,
Slides, Calendar, Tasks, Forms, Contacts and Apps Script — for several Google accounts at
once, from one Mac.

**Part E: any other remote MCP server** (MoneyForward, Canva, Notion, Linear, and so on).
Same architecture, and the cure for "why does it make me log in again every few hours?"

**Who this is for:** whoever is setting it up. No coding needed, but you will paste
commands into Terminal.

**Time:** ~20 min the first time ever (Google Cloud part). ~10 min on each additional Mac.

---

## 0. How it works (read this once, it explains every step)

```
   Claude  ──stdio──▶  mcp-remote  ──HTTP──▶  workspace-mcp  ──HTTPS──▶  Google
  (the app)            (thin bridge)          (one background        (Gmail, Drive,
                                               server on :8000)       Calendar, …)
```

Three moving parts:

| Part | What it is | Where it lives |
|---|---|---|
| **workspace-mcp** | The actual server. Open-source Python app, holds the Google logins. | Runs in the background, always on, port 8000 |
| **mcp-remote** | A tiny translator so Claude can talk to it. | Started by Claude automatically |
| **Google Cloud project** | Your own "app" registration with Google, so Google will issue logins. | console.cloud.google.com |

**The one thing to understand:** run workspace-mcp as **exactly one** background service.
Left to itself, the app starts a fresh copy of the server every time you open a window —
several servers, each thinking it owns your Google logins, none of them outliving the window
that started it. On this Mac that produced outright sign-in failures ("port 8000 already in
use"); newer versions handle the port clash more gracefully, but the duplication is still
the root of it. One always-on service, reached through `mcp-remote`, removes the whole
class of problem. Don't shortcut this — everything below assumes it.

**Software:** [taylorwilsdon/google_workspace_mcp](https://github.com/taylorwilsdon/google_workspace_mcp)

---

# PART A — Google Cloud (do this ONCE per organisation)

If your organisation already has this project set up, skip to Part B and get a copy of
`client_secret.json` from whoever owns it. Don't create a second project.

### A1. Make a new, separate project

Go to https://console.cloud.google.com → project selector, top-left → **New Project**.

Name it something like `claude-workspace`.

> ⚠️ **Do not reuse an existing project** — for example one already powering another
> integration. The branding/consent screen is shared across the whole project, so editing it
> changes what the *other* integration's login screen says and risks breaking something that
> works today.

### A2. Turn on the APIs

Go to https://console.cloud.google.com/apis/library (check the project name top-left is the
new one). Search each of these and click **Enable**:

**Required:**
- Gmail API
- Google Drive API
- Google Docs API
- Google Sheets API
- Google Slides API
- Google Calendar API

**Also enable these** if you want the full tool set:
- Google Tasks API
- Google Forms API
- People API *(contacts)*
- Google Chat API
- Apps Script API
- Custom Search API *(web search tool — see note below)*

> ⚠️ **The web search tool needs more than the API.** Enabling Custom Search alone isn't
> enough: you must also create a [Programmable Search Engine](https://programmablesearchengine.google.com/)
> and pass **both** `GOOGLE_PSE_API_KEY` and `GOOGLE_PSE_ENGINE_ID` as environment variables
> in the service file (step B3). Without them the search tool loads but fails on first use.
> If you don't need web search, skip this API entirely.

> 💡 Missing one doesn't break setup — it fails later with a clear message and a one-click
> "Enable it here" link. Calendar is the easiest one to overlook.

### A3. Consent screen (Google calls it "Google Auth Platform")

Left menu → **Branding**:
- App name: e.g. `Claude Workspace`
- User support email: your address
- Developer contact: your address
- Everything else: skip → **Save**

Left menu → **Audience**:
- User type: **External**
- Add every Google account you want Claude to reach, under **Test users**
- Then click **PUBLISH APP** and confirm the "unverified app" warning

> ⚠️ **This publish step matters.** While the app is in *Testing*, Google expires the login
> **every 7 days** and you have to re-authorise all accounts. Publishing makes the logins
> long-lived. The "unverified app" warning is expected — it's your own app, used only by
> people you've added.

### A4. Create the credentials file

Left menu → **Clients** → **Create client**:
- Application type: **Desktop app**
- Name: anything, e.g. `claude-workspace-desktop`
- **Create** → **Download JSON**

That downloaded file is your `client_secret.json`. It contains a password
(`client_secret`). Treat it like a password:

- ✅ Store the master copy in a password manager
- ❌ Never put it in a shared cloud folder, never paste it into a chat, never commit it to git

---

# PART B — The Mac (do this on EVERY Mac)

### B1. Install the two runtimes

Open Terminal and check what's already there:

```bash
which uvx; /opt/homebrew/opt/node@20/bin/node --version
```

If `uvx` is missing, install it:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

If Node 20 is missing, install it:

```bash
brew install node@20
```

> ⚠️ **Node 20 specifically.** Node 18 fails with a hard crash (`mcp-remote` needs a newer
> `undici`). Point at the Node 20 binary explicitly — don't rely on whatever `node` happens
> to be first in the PATH.

> ⚠️ **Don't run `uvx workspace-cli`.** That's an unrelated, long-dormant PyPI package
> (last released 2021), not this server. The correct name is `workspace-mcp`.

### B2. Put the credentials file in place

Create the folder and copy the JSON from A4 into it:

```bash
mkdir -p ~/.google_workspace_mcp
cp ~/Downloads/client_secret_*.json ~/.google_workspace_mcp/client_secret.json
chmod 600 ~/.google_workspace_mcp/client_secret.json
```

> ⚠️ **Keep it on the local disk**, exactly at this path. Storing it in a synced cloud
> folder is a trap — the day someone tidies that folder, the server stops working.

### B3. Create the background service

This is the file that keeps the server running and restarts it if it dies. Paste this whole
block into Terminal — replace `YOURNAME` with your Mac username (`whoami` will tell you):

```bash
cat > ~/Library/LaunchAgents/local.workspace-mcp.plist <<'PLIST'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Label</key>
	<string>local.workspace-mcp</string>
	<key>ProgramArguments</key>
	<array>
		<string>/Users/YOURNAME/.local/bin/uvx</string>
		<string>--offline</string>
		<string>workspace-mcp</string>
		<string>--single-user</string>
		<string>--tool-tier</string>
		<string>complete</string>
		<string>--transport</string>
		<string>streamable-http</string>
	</array>
	<key>EnvironmentVariables</key>
	<dict>
		<key>GOOGLE_CLIENT_SECRET_PATH</key>
		<string>/Users/YOURNAME/.google_workspace_mcp/client_secret.json</string>
		<key>WORKSPACE_MCP_HOST</key>
		<string>127.0.0.1</string>
		<key>WORKSPACE_MCP_PORT</key>
		<string>8000</string>
		<key>PATH</key>
		<string>/Users/YOURNAME/.local/bin:/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin</string>
	</dict>
	<key>RunAtLoad</key>
	<true/>
	<key>KeepAlive</key>
	<true/>
	<key>StandardOutPath</key>
	<string>/tmp/workspace-mcp.out.log</string>
	<key>StandardErrorPath</key>
	<string>/tmp/workspace-mcp.err.log</string>
</dict>
</plist>
PLIST
```

> The `Label` is just a name — pick anything. Matching it to the filename is convention, not
> a requirement, but it keeps things findable, and it's what you'll use in the restart
> commands below.

**What each setting does — worth knowing when it misbehaves:**

| Setting | Why |
|---|---|
| `--single-user` | One person's Mac. Skips the multi-tenant auth layer. |
| `--tool-tier complete` | Loads every tool in the tier catalogue. **There is no default tier** — leaving the flag off also registers everything. Use `core` or `extended` to cut the tool count down. |
| `--transport streamable-http` | Runs as an HTTP server instead of one-process-per-client. **This is the fix for the port fight.** |
| `--offline` | Boots from the local cache. Without it, a PyPI outage at login means no Google tools all day. |
| `WORKSPACE_MCP_HOST 127.0.0.1` | Local only. Nothing on the network can reach it. |
| `KeepAlive` | macOS restarts it if it crashes. |
| `RunAtLoad` | Starts at login. |

> 💡 **Start read-only if you're nervous.** Add `--read-only` to the list above and the
> server requests only read scopes and hides every tool that writes — Claude can read your
> mail and files but cannot send, delete or edit anything. Good first week. Remove the flag
> and re-authorise each account when you're ready for write access.

> 💡 **Want fewer tools?** Instead of `--tool-tier complete`, list only what you need:
> `--tools gmail drive calendar`. Valid services are `appscript`, `calendar`, `chat`,
> `contacts`, `docs`, `drive`, `forms`, `gmail`, `search`, `sheets`, `slides`, `tasks`.
> For finer control, `--permissions gmail:organize drive:readonly` — Gmail levels are
> `readonly`, `organize`, `drafts`, `send`, `full` (cumulative); Tasks levels are
> `readonly`, `manage`, `full`; every other service is `readonly` or `full`.
> Note `--permissions` **cannot be combined** with `--read-only` or `--tools` — pick one
> approach.

**Alternative to `uvx --offline`:** pin the install instead of running an ephemeral copy —

```bash
uv tool install workspace-mcp
```

Then change the first two lines of `ProgramArguments` from
`/Users/YOURNAME/.local/bin/uvx` + `--offline` + `workspace-mcp` to just
`/Users/YOURNAME/.local/bin/workspace-mcp`. This is the upstream-recommended approach and
removes the network dependency at startup properly, rather than papering over it. Upgrade
deliberately with `uv tool upgrade workspace-mcp`.

Before starting it, **prime the cache once** (only needed if you kept `uvx --offline`):

```bash
~/.local/bin/uvx workspace-mcp --help >/dev/null && echo "cached OK"
```

Now start it:

```bash
launchctl load ~/Library/LaunchAgents/local.workspace-mcp.plist
```

Check it's alive:

```bash
launchctl list | grep workspace-mcp
lsof -nP -iTCP:8000 -sTCP:LISTEN
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://127.0.0.1:8000/mcp
```

You want: a line from `launchctl`, a `Python … 127.0.0.1:8000 (LISTEN)` line, and
**`HTTP 406`**. 406 is correct — it means the server answered.

### B4. Tell Claude about it

Register the bridge (not the server itself):

```bash
claude mcp add --scope user workspace-mcp -- /opt/homebrew/opt/node@20/bin/npx -y mcp-remote@latest http://127.0.0.1:8000/mcp --allow-http --transport http-only
```

If the `claude` command isn't available, add it by hand instead — in `~/.claude.json`,
under `"mcpServers"`:

```json
"workspace-mcp": {
  "command": "/opt/homebrew/opt/node@20/bin/npx",
  "args": ["-y", "mcp-remote@latest", "http://127.0.0.1:8000/mcp", "--allow-http", "--transport", "http-only"],
  "env": { "PATH": "/opt/homebrew/opt/node@20/bin:/usr/local/bin:/usr/bin:/bin" }
}
```

For the Claude **desktop app**, the same block goes in
`~/Library/Application Support/Claude/claude_desktop_config.json`.

**Then fully quit and reopen Claude** (⌘Q, not just closing the window).

---

# PART C — Sign in each Google account

There is no settings screen for this. You trigger it by asking Claude to do something, and
it hands you a link.

1. Ask Claude: **"list my calendars for you@yourcompany.com"**
2. Claude replies with an authorisation link → click it
3. Sign in **as that account**, accept the "unverified app" warning, approve the permissions
4. Come back and ask again — it should now list the calendars
5. Repeat for every account

> ⚠️ The links expire after about ten minutes. If it fails, just ask again for a fresh one.

Logins are saved to `~/.google_workspace_mcp/credentials/<email>.json`, one file per
account, readable only by you.

### Several Google accounts at once — read this before you build it wrong

**`--single-user` does NOT mean one account.** Its own help text: *"bypass session mapping
and use any credentials from the credentials directory."* It means **one person on one Mac**
— and that person can have as many Google accounts as they like. Work, personal, a spouse's,
a shared team address: all at once, all through the one server. Badly named flag.

How it works:

- One saved login file per account, all sitting in the same `credentials/` folder
- Every request names the account it's for, so the server picks the matching login
- You say *"search my work Gmail"* vs *"check my personal calendar"* and Claude passes the
  right address

> ⛔ **The mistake to avoid: do not run a second server for the second account.** People try
> this and both copies fight over port 8000 — it's a known dead end on the project's issue
> tracker, closed with no fix. **One server, many accounts.** Always.

**Adding an account later** takes no config change and no restart. Ask Claude to do
something with the new address, click the link, done.

**Tired of naming the account every time?** If you have one main account and rarely touch
the others, set a default — add this to the `EnvironmentVariables` section of the service
file, then restart it:

```xml
<key>USER_GOOGLE_EMAIL</key>
<string>you@yourcompany.com</string>
```

Requests that don't name an account then use that one, and the server still honours a
different address when you name it explicitly.

> ⚠️ **Don't set this if you use several accounts evenly.** It does more than supply a
> default: the server also tells the AI *"always use `<that email>` … do not ask the user
> for their email address."* That instruction fights a multi-account setup — you may find
> your other accounts quietly ignored unless you name them very forcefully. Leave it unset
> and just say which account you mean.

> ⚠️ Don't combine `--single-user` with `MCP_ENABLE_OAUTH21=true` — they're mutually
> exclusive and the server refuses to start. That setting belongs to the multi-user server
> deployment covered in the project's own docs, not to this guide.

---

# PART D — Prove it works

Ask Claude, for each account:

- "list my calendars for `<email>`"
- "search my Gmail for anything from the last 7 days, `<email>`"

Both must return real data. Anything less is not finished.

---

# PART E — The same pattern for other (non-Google) MCP servers

Everything above was about Google. The interesting part is that **the same fix applies to
almost any hosted MCP server** — and if you're being asked to log in again several times a
day, this is why.

### First, know which kind you have

| Kind | Examples | Needs this? |
|---|---|---|
| **Local** — runs on your Mac, no login | n8n docs, filesystem tools | ❌ No. Nothing to race over. |
| **Remote with an API key** | services using a token in the config | ❌ No. A key can still be rotated or revoked, but there's no hourly refresh for clients to collide over. |
| **Remote with OAuth login** | MoneyForward, Canva, Notion, Linear | ✅ **Yes, if it keeps logging you out.** |

### Why the repeated logins happen

A remote server is reached through a helper called `mcp-remote`. **Every** app that connects
starts its own copy — each Claude window, each session of any other agent. Copies running the
same version against the same service share one saved login on disk (under
`~/.mcp-auth/mcp-remote-<version>/`). When a short-lived token expires, several copies can
try to refresh it at once; one succeeds and rewrites the file, and the others can be left
holding a token that no longer works — so you're asked to log in again.

`mcp-remote` does try to coordinate this with lock files, so it isn't guaranteed to be the
cause of *every* repeat login — a vendor can also revoke access or shorten a session. But
when several AI apps share one service, this is the usual culprit, and it's the one you can
actually fix.

Structurally it's the port-8000 problem from Part 0 in a different costume, and the cure is
the same: **one process owns the login, everyone else connects to it.**

### The fix — one bridge per service

A LaunchAgent runs `mcp-proxy`, which holds a single `mcp-remote` connection to the vendor
and re-serves it on a local port. Every app connects to that port instead of the vendor.

**Step 1 — install `mcp-remote` globally at a fixed version, once:**

```bash
npm install -g mcp-remote@0.1.37
```

> ⚠️ **Name a version, and install it globally rather than using `npx -y`.** Two reasons.
> The saved logins live in a folder named after the version
> (`~/.mcp-auth/mcp-remote-0.1.37/`), so upgrading silently starts from an empty folder and
> logs you out of every remote service at once. And a global install lives at one stable
> path, instead of an npx cache directory that changes and can end up with a non-executable
> script. `0.1.37` is the version this guide is written against; a later one will work, but
> change it deliberately and expect to log in again once.

> 🪲 **Field note (2026-08-15): 0.1.37 saves its logins in the 0.1.36 folder.** The npm
> package published as `mcp-remote@0.1.37` has the version string `0.1.36` embedded in its
> bundled code (`dist/chunk-F76MHFRJ.js`), so at runtime it writes to
> `~/.mcp-auth/mcp-remote-0.1.36/` — **not** the `0.1.37` folder the version-named-folder
> rule above predicts. When checking saved logins, look in the `0.1.36` folder. Verify on
> your own Mac: `grep -ao '0\.1\.3[0-9]' /opt/homebrew/lib/node_modules/mcp-remote/dist/*.js`

Discovered while debugging repeated MoneyForward login screens. Two failure modes live in
these folders:

- **A failed or abandoned OAuth flow** leaves `client_info…`/`code_verifier…`/`lock.json`
  artifacts with no `tokens.json`. The bridge's `mcp-remote` child then loops
  "Please authorize" — opening the browser automatically on **every** client connect.
- **A stale `lock.json`** (holding a dead PID) blocks token sharing between copies.

The fix for both:

1. Delete that service's hash-prefixed files from **both**
   `~/.mcp-auth/mcp-remote-0.1.36/` **and** `~/.mcp-auth/mcp-remote-0.1.37/` — with the
   version mismatch you can't trust which folder is live, and stale artifacts in either
   can bite
2. Restart the bridge: `launchctl kickstart -k gui/$(id -u)/local.SERVICE-mcp-bridge`
3. Use any tool from that service once and complete the vendor login in the browser —
   from then on the bridge holds it as normal

**Step 2 — create the bridge.** Replace `SERVICE`, `PORT`, `REMOTE_URL` and `YOURNAME`.

First check the port is free and the pieces exist — this takes five seconds and saves a
confusing failure later:

```bash
lsof -nP -iTCP:PORT -sTCP:LISTEN || echo "port free ✅"
ls ~/.local/bin/uvx /opt/homebrew/opt/node@20/bin/node /opt/homebrew/bin/mcp-remote
```

All three must exist. Then write the file — this refuses to clobber an existing bridge:

```bash
P=~/Library/LaunchAgents/local.SERVICE-mcp-bridge.plist
[ -e "$P" ] && { echo "REFUSING — $P already exists"; } || cat > "$P" <<'PLIST'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Label</key>
	<string>local.SERVICE-mcp-bridge</string>
	<key>ProgramArguments</key>
	<array>
		<string>/Users/YOURNAME/.local/bin/uvx</string>
		<string>--with</string>
		<string>mcp==1.9.4</string>
		<string>mcp-proxy==0.9.0</string>
		<string>--port</string>
		<string>PORT</string>
		<string>--pass-environment</string>
		<string>--</string>
		<string>/opt/homebrew/opt/node@20/bin/node</string>
		<string>/opt/homebrew/bin/mcp-remote</string>
		<string>REMOTE_URL</string>
	</array>
	<key>EnvironmentVariables</key>
	<dict>
		<key>PATH</key>
		<string>/opt/homebrew/opt/node@20/bin:/Users/YOURNAME/.local/bin:/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin</string>
	</dict>
	<key>RunAtLoad</key>
	<true/>
	<key>KeepAlive</key>
	<true/>
	<key>StandardOutPath</key>
	<string>/tmp/SERVICE-mcp-bridge.out.log</string>
	<key>StandardErrorPath</key>
	<string>/tmp/SERVICE-mcp-bridge.err.log</string>
</dict>
</plist>
PLIST

plutil -lint "$P" && launchctl bootstrap gui/$(id -u) "$P"
launchctl print gui/$(id -u)/local.SERVICE-mcp-bridge | grep -E "state|pid"
```

`plutil -lint` catches a mistyped plist before launchd does, where the error is obscure. If
the label is already loaded, `bootstrap` fails — use
`launchctl kickstart -k gui/$(id -u)/local.SERVICE-mcp-bridge` to restart instead.

**Why the pinned versions:** `mcp-proxy==0.9.0` with `mcp==1.9.4` is the combination proven
to work. `mcp-proxy` accepts a *range* of `mcp` versions rather than a specific one, so an
unpinned install can silently resolve to a combination that fails — which is how this was
originally discovered. Pin both, and change them deliberately.

**Why `--pass-environment`:** `mcp-proxy` starts its child with an empty environment by
default; this hands the child the environment above. The template calls Node by absolute
path so it doesn't strictly depend on `PATH`, but the vendor helper may read other variables,
and both working bridges use the flag.

**Step 3 — point your apps at the bridge instead of the vendor:**

```bash
claude mcp add --scope user SERVICE -- /opt/homebrew/opt/node@20/bin/node /opt/homebrew/bin/mcp-remote http://127.0.0.1:PORT/sse --allow-http --transport sse-only
```

> ⚠️ **Call Node explicitly, with `mcp-remote` as its first argument** — don't run
> `mcp-remote` directly. It starts with `#!/usr/bin/env node`, so running it directly makes
> it hunt for `node` on the PATH. A desktop app launched from the Dock doesn't inherit your
> Terminal's PATH, so it can work in a terminal and fail in the app. Naming Node by absolute
> path removes the guesswork.

> ⚠️ **The address ends in `/sse`, not `/mcp`.** This bridge speaks SSE; asking for `/mcp`
> returns 404 and looks like a broken server. (The Google server in Part B is the opposite —
> it genuinely serves `/mcp`. Different software, different endpoint.)

**Step 4 — first login and proof:**

```bash
curl -s -o /dev/null -w "status %{http_code}\n" http://127.0.0.1:PORT/status
```

`200` means the bridge is up. Restart your app, use any tool from that service once, and
complete the vendor's login in the browser if asked. From then on the bridge holds that
login and every app shares it, so the *repeated daily* prompts stop.

You will still be asked again occasionally — if the vendor revokes access or expires the
session, if you change `mcp-remote` versions, or if the saved-login folder is cleared. The
bridge removes the self-inflicted cause, not every possible one.

### Notes worth knowing

- **One bridge per service, one port each.** They don't share.
- **Only bridge what actually annoys you.** A service you re-authorise twice a year isn't
  worth a background process; leave it pointed straight at the vendor.
- **This helps most when you run more than one AI app** (e.g. Claude and another agent) —
  that's what turns an occasional re-login into a daily one.

---

# PART F — When it breaks

Replace `local.workspace-mcp` below with whatever `Label` you used.

| Symptom | Cause | Fix |
|---|---|---|
| "Port 8000 is already in use. Cannot start minimal OAuth server." | A second copy of workspace-mcp started — usually Claude was configured to launch it directly instead of via `mcp-remote`. | `pkill -f workspace-mcp`, fix the Claude config to use `mcp-remote`, then `launchctl kickstart -k gui/$(id -u)/local.workspace-mcp` |
| Google tools vanish from Claude | The server crashed and didn't come back | `launchctl kickstart -k gui/$(id -u)/local.workspace-mcp` — back in ~1 second. Cause is in `/tmp/workspace-mcp.err.log` |
| "Google Calendar API is not enabled for your project" | Missed an API in step A2 | Click the link in the error, wait 1–2 min, retry |
| Asks you to re-login every week | The OAuth app is still in *Testing* | Publish it (step A3) |
| Everything dies after a cloud-folder cleanup | `client_secret.json` was stored in a synced folder | Move it to `~/.google_workspace_mcp/client_secret.json` (step B2) |
| `mcp-remote` crashes with an `undici` stack trace | Running on Node 18 | Point at the Node 20 binary explicitly (step B1) |
| Server won't start after a PyPI/network outage | `--offline` with nothing in the cache | Run `uvx workspace-mcp --help` once while online to re-prime, or switch to `uv tool install` (step B3) |
| Server exits immediately, complains about OAuth 2.1 | `--single-user` combined with `MCP_ENABLE_OAUTH21=true` | Pick one — for a personal Mac, keep `--single-user` and drop the OAuth 2.1 setting |
| Second account returns the first account's data | You're running two server copies, or the request didn't name an account | One server only (Part C). Check `USER_GOOGLE_EMAIL` isn't silently defaulting every request |
| Command not found / weird unrelated errors on install | You ran `uvx workspace-cli` — an unrelated package | The correct name is `workspace-mcp` |
| **(Part E)** A bridged service still asks you to log in repeatedly | Likeliest: an app still points at the vendor URL instead of the bridge, re-creating the race | Compare every app's config — URL, executable and version must match. If all agree, the cause is at the vendor's end instead |
| **(Part E)** Bridge starts but calls fail | Version combination — `mcp-proxy` accepts a range of `mcp` versions and can resolve a bad pair | Pin `mcp==1.9.4` with `mcp-proxy==0.9.0` |
| **(Part E)** `404` from the bridge | You used `/mcp` | These bridges serve `/sse`. Health-check on `/status` |
| **(Part E)** Worked yesterday, "permission denied" today | Possibly a cached copy that isn't executable — check first: `ls -l ~/.npm/_npx/*/node_modules/mcp-remote/dist/proxy.js` | If it lacks `x`, install globally and point at the fixed path instead of `npx` |
| **(Part E)** Logged out of every remote service at once | `mcp-remote` version changed, so it's reading a new, empty login folder | Check which `~/.mcp-auth/mcp-remote-*` folders exist; pin the version you authorised against. Note the folder can lag the installed version — `0.1.37` writes to `0.1.36/` (see the field note in Part E, Step 1) |

**Health check, any time:**

```bash
launchctl list | grep workspace-mcp && lsof -nP -iTCP:8000 -sTCP:LISTEN && tail -5 /tmp/workspace-mcp.err.log
```

---

# Appendix A — how this differs from the official project, and why

The upstream project is [taylorwilsdon/google_workspace_mcp](https://github.com/taylorwilsdon/google_workspace_mcp).
Its README is written for a wide audience, from a laptop to a team server. This guide is
narrower — **one person, one Mac, several accounts** — and deviates where the defaults hurt
in that setting. Each change and the reason:

| # | Upstream / the common example | This guide | Why |
|---|---|---|---|
| 1 | **stdio transport** — still the default in the code | `--transport streamable-http` behind a LaunchAgent | stdio means one server process per connected app. Newer upstream versions dodge the worst symptom (the callback server now starts only when needed and can fall back across ports 8000–8004), but you still get several servers, several credential owners, and nothing keeping one alive. One always-on process removes the whole class of problem. Historically this *was* a hard failure here — repeated "port 8000 already in use" during sign-in. |
| 2 | **The app launches the server** — the usual stdio arrangement | The app connects to an already-running server via `mcp-remote` | Follows from row 1: a server that outlives any one app, restarts itself, and is the single owner of the logins. |
| 3 | `--tool-tier core` in the Quick Start example | `--tool-tier complete` | `core` is thin where it matters for document work: 3 of 19 Docs tools and 3 of 14 Sheets tools. (Drive is less affected — `core` already has 9 of 16.) |
| 4 | **No stance on publishing status** | Explicitly PUBLISH the OAuth app | Left in *Testing*, Google expires refresh tokens after 7 days — you re-authorise every account weekly, forever. |
| 5 | `uvx workspace-mcp` in the Quick Start | `uvx --offline`, or `uv tool install` | `uvx` runs from an isolated environment and may reach for a package index at launch; `--offline` forces the local cache, and an installed copy avoids the question entirely. Note this is general `uv` practice for tools you run constantly — the project's README doesn't call for it. |
| 6 | Client-secret path **unset** | `GOOGLE_CLIENT_SECRET_PATH` → `~/.google_workspace_mcp/client_secret.json` | Unset, it looks for `client_secret.json` in the package folder. Run via `uvx`, that folder is a cache directory that gets replaced on upgrade. A fixed home-directory path is stable however you install. |
| 7 | **Docker** offered as a deployment option | Not used | Overkill for one Mac. A LaunchAgent starts at login and restarts on crash with nothing to manage. |
| 8 | **Optional hosted modes** — OAuth 2.1 / bearer tokens, service accounts, stateless | `--single-user` | Those exist to serve a team from a server. For one person they add moving parts — and OAuth 2.1 is *incompatible* with `--single-user`; the server exits rather than start. |
| 9 | **No Node version guidance** | Node 20, by explicit path | Node 18 crashed `mcp-remote` when this was set up here. Naming the binary also avoids inheriting whatever `node` a GUI app happens to find. |
| 10 | **`mcp-remote` isn't upstream at all** — its own README usually shows it unversioned | A specific version, installed globally | The `mcp-remote` bridge is this guide's architectural choice, not the project's. Once you use it, pin it: the saved-login folder is named after the version, so a bump reads an empty folder and logs you out of every remote service at once. |

**Settings taken straight from upstream:** a Desktop-app OAuth client, the default token
directory `~/.google_workspace_mcp/credentials/`, and the project's own tier, permission and
service names. (Web-application OAuth clients are supported too — Desktop is simply the
easier one for a personal Mac.)

Everything else above — the explicit client-secret path, `--single-user`,
`--tool-tier complete`, the LaunchAgent, and the `mcp-remote` bridge — are **this guide's
choices**, not upstream defaults. If they stop suiting you, they're all reversible.

**If you're deploying for a team rather than yourself, ignore this guide** and use the
project's own documentation — the multi-user, OAuth 2.1 and container paths exist for
exactly that and are better suited than anything here.

---

# Appendix B — where everything lives

| What | Path |
|---|---|
| The service definition | `~/Library/LaunchAgents/local.workspace-mcp.plist` |
| Google app credentials | `~/.google_workspace_mcp/client_secret.json` |
| Per-account logins | `~/.google_workspace_mcp/credentials/<email>.json` |
| Logs | `/tmp/workspace-mcp.out.log`, `/tmp/workspace-mcp.err.log` |
| Claude's connection | `~/.claude.json` → `mcpServers` → `workspace-mcp` |
| **(Part E)** Other-service bridges | `~/Library/LaunchAgents/local.<service>-mcp-bridge.plist` (whatever you named them) |
| **(Part E)** Their logs | `/tmp/<service>-mcp-bridge.err.log` **and** `.out.log` — check both; startup problems often land in `.out.log` |
| **(Part E)** Saved logins for remote services | `~/.mcp-auth/mcp-remote-<version>/` — note the version in the folder name; that's why pinning matters. Caveat: the name comes from the version *embedded in the bundle*, which can lag the published one — `0.1.37` writes to `mcp-remote-0.1.36/` (see the field note in Part E, Step 1) |

### Security notes for whoever runs this

- Each file in `credentials/` is **full read+write access** to that Google account — send
  mail as them, delete their files. Anyone who can read the Mac's disk has the account.
- Keep FileVault on. Keep `chmod 600` on those files.
- To revoke one account: delete its `credentials/<email>.json` **and** remove access at
  https://myaccount.google.com/permissions (deleting the local file alone doesn't revoke it
  at Google's end).
- The Google Cloud project is shared across your organisation; each Mac gets its own copy of
  `client_secret.json` and does its own account sign-ins. Logins are **not** shared between
  Macs.
