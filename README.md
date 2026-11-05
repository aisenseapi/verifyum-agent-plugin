# Verifyum: Agent Plugin

A portable [Agent Plugins](https://agent-plugins.org) 1.0.0 package that
connects an agent to [Verifyum](https://verifyum.com/agents) file proofs on
Solana Mainnet, with one skill that tells the agent when and how to use them.

| File | What it is |
| --- | --- |
| `plugin.json` | The manifest: name `verifyum`, version, author, license, and Hermes Agent trust for both servers |
| `mcp.json` | Two servers. `verifyum` is the hosted server at `https://api.verifyum.com/mcp` (Streamable HTTP, no authentication). `verifyum-local` runs the `verifyum-mcp` npm package on this machine with `npx` |
| `skills/verifyum-proofs/SKILL.md` | Which tool fits which need, the rules on permanence, privacy and limits, and a read-only check with a known answer |
| `.claude-plugin/`, `.mcp.json` | Claude Code: the plugin manifest, a marketplace with this one plugin, and the same two servers |
| `.agents/plugins/marketplace.json` | Codex: a marketplace with this one plugin. Codex reads `plugin.json` and `mcp.json` itself |
| `gemini-extension.json` | Gemini CLI: the extension manifest with the same two servers |
| `.antigravity-plugin/`, `mcp_config.json`, `assets/logo.svg` | Antigravity: the manifest with display name, logo and starter prompts, and the same two servers |

The package itself contains no code. `verifyum-local` downloads the exact npm
release `verifyum-mcp@0.1.1` (MIT, JavaScript source) the first time it
starts.

## Install

Claude Code:

```text
/plugin marketplace add aisenseapi/verifyum-agent-plugin
/plugin install verifyum@verifyum
```

Codex, which then lists Verifyum among the plugins it can install:

```bash
codex plugin marketplace add aisenseapi/verifyum-agent-plugin
```

Gemini CLI:

```bash
gemini extensions install https://github.com/aisenseapi/verifyum-agent-plugin
```

Antigravity, from a local copy:

```bash
git clone https://github.com/aisenseapi/verifyum-agent-plugin
agy plugin install ./verifyum-agent-plugin
```

Any other client that loads Agent Plugins can load this directory. The
[compatible clients](https://agent-plugins.org/compatible-clients) page links
each client's own instructions.

Without the plugin, each server is one entry in an MCP client. The hosted
server:

```bash
claude mcp add --transport http verifyum https://api.verifyum.com/mcp
hermes mcp add verifyum --url https://api.verifyum.com/mcp
openclaw mcp add verifyum --url https://api.verifyum.com/mcp --transport streamable-http
```

The local server, here for Claude Code with one folder the file tool may use:

```bash
claude mcp add verifyum-local -e VERIFYUM_MCP_ALLOWED_DIRS=/path/to/folder -- npx -y verifyum-mcp@0.1.1
```

## What the tools do

| Tool | Servers | Kind | What it does |
| --- | --- | --- | --- |
| `verifyum_get_proof` | both | read-only | Reads one proof's state: `queued`, `submitted`, `finalized` or `failed` |
| `verifyum_verify_public_proof` | both | read-only | Checks a finalized proof. The local server also reads the transaction from a public Solana RPC endpoint. The hosted server checks Verifyum's own data and returns the RPC request for a separate check |
| `verifyum_anchor_commitment` | both | public write | Anchors a commitment that was already computed, with an idempotency key |
| `verifyum_create_proof_from_file` | `verifyum-local` | local read, public write | Hashes a local file, saves the private draft and anchors only the commitment |

## Turn on the file tool

`verifyum_create_proof_from_file` reads files only inside the directories
listed in `VERIFYUM_MCP_ALLOWED_DIRS`, and writes the private draft there too.
Entries are separated by `:` on macOS and Linux and by `;` on Windows. The
package does not set the variable, so the tool refuses every path until the
user sets it in the client's configuration for `verifyum-local`. The three
other tools work without it.

## What to know before installing

- **Anchoring is permanent and public.** `verifyum_anchor_commitment` and
  `verifyum_create_proof_from_file` create a real Solana Mainnet transaction
  that cannot be withdrawn. The skill tells the agent to anchor only on the
  user's explicit request. `plugin.json` marks both servers `untrusted` for
  Hermes Agent, so Hermes asks before write-capable calls.
- **What leaves the machine.** Verifyum receives only the commitment, the
  idempotency key and proof IDs. The file, filename, raw hash, nonce and
  private manifest are never sent to Verifyum. Tool arguments, including the
  local paths given to the file tool, still reach the user's model provider
  through the session.
- **Network.** The hosted server is at `api.verifyum.com`. `verifyum-local`
  calls `api.verifyum.com`, `verifyum.com` and `<proof-id>.verifyum.com`, and
  reads transactions from `api.mainnet-beta.solana.com`. `npx` fetches the
  package from the npm registry on first start.
- **Local files.** `verifyum-local` reads only regular files inside
  `VERIFYUM_MCP_ALLOWED_DIRS` and rejects symbolic links. Private drafts are
  written with mode 0600 on macOS and Linux. On Windows a draft inherits the
  folder's permissions, so keep that folder private.
- **The private draft matters.** Full verification needs the original file
  and its private draft. A lost draft cannot be recreated.
- **Cost and limits.** Public use is currently free and AI SENSE AS pays the
  Solana fee. That is current policy, not a guarantee. At most ten new anchors
  per rolling hour per IP address, with lower daily caps.
- **Privacy.** What the service receives and keeps is described at
  https://verifyum.com/privacy.
- **The tool lists come from the servers at run time.** A pinned commit of
  this package does not fix them. The local server is pinned to
  `verifyum-mcp@0.1.1`.

## Check it works

This check reads a public proof and never anchors anything. Ask the agent to
call `verifyum_get_proof` with proof ID `72c38s8rrqa4vjhw6dxsvmbqkq`. The
answer is status `finalized` on network `mainnet-beta`. Then ask for
`verifyum_verify_public_proof` with the same ID. The verdict is `valid`, and
the local server reports slot `442907852` and block time
`2026-08-30T14:59:44Z`.

## What a proof does not prove

A finalized proof shows that a commitment to the exact file bytes existed no
later than the estimated Solana block time. It does not prove authorship,
ownership, legal signature validity, when the file was first created, or that
its contents are true.

## License and contact

MIT, see [LICENSE](LICENSE). AI SENSE AS, post@aisense.no,
https://verifyum.com.
