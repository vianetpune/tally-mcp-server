# Using this Tally MCP server with Claude

There are two ways to drive Tally from Claude. Pick by where you want to use it from.

| | Route A: Claude desktop app (local) | Route B: remote connector |
|---|---|---|
| Works from | Any Claude chat linked to the computer running the server | Anywhere, including the phone app |
| Needs | Claude desktop app open on that computer | A public HTTPS address (tunnel) and a strong password |
| Exposure | Nothing leaves your network | Tally data reachable over the internet behind a password |
| Effort | 10 minutes | An hour or so |

Start with Route A. Move to Route B only if you need Tally from your phone or away from the office.

## Before either route: Tally side (on the Windows PC running Tally)

1. In TallyPrime: **F1 › Settings › Connectivity › Client/Server configuration**
   - TallyPrime acts as: **Server** (or **Both**)
   - Port: **9000**
2. Keep TallyPrime open with the company (Vianet / Viacom) loaded. The server reads whatever companies are open.
3. If the MCP server will run on a *different* computer (such as the MacBook), allow inbound TCP 9000 in Windows Defender Firewall for your office network only, and give the Tally PC a fixed LAN IP (a DHCP reservation on the router).

## Route A: Claude desktop app

### A1. Server on the Tally PC itself (simplest, always on in office)

1. Install Node.js (LTS) and the Claude desktop app on the Tally PC.
2. Download this repo (Code › Download ZIP) and extract it, say to `D:\Tally MCP Server`. In that folder run `npm install` once.
3. Edit the Claude desktop config (Settings › Developer › Edit Config) to:

   ```json
   {
     "mcpServers": {
       "Tally Prime": {
         "command": "node",
         "args": ["D:\\Tally MCP Server\\dist\\index.mjs"]
       }
     }
   }
   ```

4. Quit and reopen the Claude desktop app. "Tally Prime" should appear under tools.

### A2. Server on the MacBook, Tally on the office PC

Works only while the Mac is on the same network as the Tally PC (in office, or over VPN).

1. Clone or download this repo on the Mac, then run `npm install` in it.
2. Edit the Claude desktop config on the Mac to (adjust the path and IP):

   ```json
   {
     "mcpServers": {
       "Tally Prime": {
         "command": "node",
         "args": ["/Users/<you>/tally-mcp-server/dist/index.mjs"],
         "env": { "TALLY_HOST": "192.168.1.50", "TALLY_PORT": "9000" }
       }
     }
   }
   ```

   `TALLY_HOST` is the Tally PC's LAN IP. You can put the same values in this folder's `.env` instead; `.env` is read from the project folder, so it works no matter where Claude launches the server from. If a value is set in both places, the Claude config wins, except for `ALLOWED_COMPANIES` and `ALLOWED_TOOLS` (see below).
3. Quick check from the Mac's terminal before restarting Claude: `curl -s --max-time 5 -X POST http://192.168.1.50:9000 -d '<ENVELOPE/>'` should return a response from Tally instead of hanging.
4. Quit and reopen the Claude desktop app.

### Using it from a claude.ai chat (web or another device)

A chat on claude.ai can use the desktop app's local tools when that chat is linked to the computer running them: open the chat in the Claude desktop app on that computer and send a message. The Tally tools then show up in that chat alongside the computer's other tools, for as long as the desktop app stays open and online.

## Route B: remote connector (phone and anywhere)

Run the HTTP server on the Tally PC and publish it over HTTPS. The repo's [Linux guide](server-setup-linux.md) covers a hosted setup; the Windows guide is still a stub, so on the Tally PC itself the steps are:

1. In this folder's `.env` set:

   ```
   PORT=3000
   MCP_DOMAIN=https://tally.yourdomain.in
   PASSWORD=<long random password, 20+ characters>
   ```

   **Set `PASSWORD`.** If it is missing the server falls back to the literal password `password`, and anyone who finds the URL can read your books.
2. Start it with `node dist/server.mjs` (and later as a Windows service so it survives restarts).
3. Publish `http://localhost:3000` over HTTPS, for example with a Cloudflare Tunnel (`cloudflared`) pointed at a subdomain you own. Don't open router ports.
4. In claude.ai, add a custom connector with the URL `https://tally.yourdomain.in/mcp`. Claude opens the server's sign-in page; enter the password.

## Add the skill

`skills/tally-accounts/SKILL.md` teaches Claude how to use these tools properly for Vianet and Viacom: sign conventions, totals through SQL, final accounts, audit and reco flags, receivables ageing, intercompany matching and stock checks.

Zip the `skills/tally-accounts` folder and upload it as a custom skill in Claude's settings. After that, asking things like "what's overdue over 60 days in Vianet" or "check Vianet vs Viacom intercompany as on 31 March" picks up the skill automatically.

## Setting it up for office staff

You can run the server on a staff member's PC (a Tally client machine works) and limit what it can read. How strong the limit is depends on who controls the machine the server runs on.

**Limit what the server offers.** In the server's `.env`:

```
ALLOWED_COMPANIES=Vianet
ALLOWED_TOOLS=list-master,bills-outstanding,stock-summary,stock-item-balance
```

Requests for any other company are refused before they reach Tally, the company list only shows allowed companies, and tools not listed don't exist for Claude. Useful profiles:

| Role | `ALLOWED_TOOLS` |
|---|---|
| Collections / sales follow-up | `list-master,bills-outstanding,ledger-account` |
| Stock and dispatch | `list-master,stock-summary,stock-item-balance,stock-item-account` |
| Accounts (full) | leave empty |

**Make the limit stick.** These limits live in a file. Anyone who can edit that file, or who can reach Tally's port 9000 with their own tools, can get around them. So:

1. **Windows Firewall on the Tally PC:** allow inbound TCP 9000 only from the IP of the PC running this server, not the whole office network. Tally's XML port has no password of its own, and it accepts imports as well as exports.
2. **Staff log in as a standard Windows user**, not an administrator. Put the server folder somewhere they can read but not write (for example `C:\Program Files\Tally MCP Server`, set up from an admin account), so they can't edit `.env`. Put the limits in that `.env`, not in the Claude desktop config: the config is editable by the user, so limits set in `.env` always take precedence over it.
3. **Use the staff member's own Claude login** on a business account, so the chats and the Tally figures in them stay under your organisation, and their access ends when you remove their seat.
4. **Tally login:** have the Tally instance that the server talks to logged in with a restricted Tally user, and test whether your Tally version applies that user's restrictions to XML requests before relying on it.

For the strongest setup, run the server only on the Tally PC (Route B), which staff can't change, and give them access to the connector rather than to a copy of the server.

## Environment variables

| Variable | Used by | Default | Meaning |
|---|---|---|---|
| `TALLY_HOST` | both | `localhost` | Host or LAN IP of the PC running TallyPrime |
| `TALLY_PORT` | both | `9000` | TallyPrime XML port |
| `PORT` | `server.mjs` | `3000` | HTTP port for Route B |
| `MCP_DOMAIN` | `server.mjs` | `http://localhost:3000` | Public HTTPS address for Route B |
| `PASSWORD` | `server.mjs` | `password` (unsafe) | Sign-in password for Route B |
| `ALLOWED_COMPANIES` | both | (all) | Comma-separated Tally companies this server may read. With no company named, the first one is used |
| `ALLOWED_TOOLS` | both | (all) | Comma-separated tools to offer, e.g. `list-master,bills-outstanding,stock-summary`. `query-database` is always kept |

Copy `.env.example` to `.env` and fill it in. `.env` is git-ignored so real values never reach GitHub.

Where a variable is set both in `.env` and by the launcher (the `env` block of the Claude desktop config, or the shell), the launcher's value is used. `ALLOWED_COMPANIES` and `ALLOWED_TOOLS` are the exception: when `.env` sets them, `.env` wins, so a user can't loosen them from their own config.

`query-database` can only see the tables the other tools cache. It can't read or write files on the computer or reach the network.
