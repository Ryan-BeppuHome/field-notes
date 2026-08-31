# Field Notes

Working setups that took real effort to get right, written down properly so the next person
doesn't have to rediscover them.

Not tutorials rewritten from documentation. Each guide here documents something actually
running, including the parts that broke first, why the defaults weren't enough, and what
was verified rather than assumed.

## Guides

| Guide | What it covers |
|---|---|
| [MCP servers on macOS](guides/mcp-servers-on-macos.md) | Connecting an AI assistant to Google Workspace (Gmail, Drive, Docs, Calendar…) for several accounts at once, and the same pattern for any other hosted MCP server. Fixes the "why am I logging in again?" problem. |
| [Apple N1 Wi-Fi degradation](guides/apple-n1-wifi-degradation-macos.md) | M5-era Macs where Wi-Fi goes unusably slow every time you leave a network and reconnect to it, while Ethernet and every other machine on the same access point are fine. How to confirm it's the driver and not your network, how to recover in ~25 s by restarting the driver processes instead of rebooting, and how to automate that safely. Includes the full elimination trail and a self-healing daemon that made things worse. |

## How these are written

- **Click-by-click, with the reasoning beside it.** You should be able to follow it without
  understanding it, and understand it if you want to.
- **Every command run before it was written down.** Flags, paths and version numbers are
  checked against a working machine, not recalled.
- **Failures included.** The dead ends are usually the most useful part.
- **What differs from upstream, and why.** Where a guide departs from a project's own
  defaults, it says so and gives the reason, so you can disagree on purpose.
- **Fact-checked adversarially.** Drafts are reviewed by a separate AI model told to hunt
  for errors, against the real source and the running system. Corrections get folded in
  before publishing.

## Caveats worth reading

These describe **one working setup on one machine**, usually macOS on Apple Silicon. They
aren't the only way, and they aren't tested across every configuration.

Pinned version numbers age. Where a guide pins something, it explains why, so you can decide
whether the reason still applies.

Software changes underneath a guide. If something doesn't match what you see, trust the
software and please open an issue.

## Contributing

Corrections very welcome — especially "this is wrong on my machine" with the details.
Open an issue or a pull request.

## Licence

[MIT](LICENSE). Use it, adapt it, republish it. Attribution appreciated, not required.
