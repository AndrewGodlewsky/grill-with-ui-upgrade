# Security audit — core grill skill

Audited 2026-10-06 against the current working tree. Scope: `server.mjs`,
`page.html`, `markdown.mjs`, `SKILL.md`, `visual-brief.md`, tests, and CI
workflows. The server has no third-party runtime dependencies. This is a
source review plus local HTTP and browser checks, not an external penetration
test or a guarantee about the host machine.

## Trust model

- The agent and the local user can read and write the session directory. The
  browser may read state and submit actions; only the agent CLI patches state.
- Loopback HTTP trusts other processes running on the same machine. Binding to
  the LAN is explicit and uses a fresh 128-bit bearer token on every `serve`.
  Anyone who obtains the LAN URL can read the grill and submit actions.
- `visual.html` is agent-produced HTML. It is displayed in a sandboxed iframe
  with an opaque origin and a restrictive content security policy.

## Findings and changes

| Area | Finding | Resolution |
| --- | --- | --- |
| Network exposure | A LAN listener without a request secret would expose the session and agent send endpoint. | Default binding remains loopback. Non-loopback requests require a per-serve token. The LAN URL is printed only when a LAN address is available. |
| Browser requests | Another website could try cross-origin POSTs to the local service. | POSTs reject foreign `Origin` values. Requests also reject unexpected `Host` values, limiting DNS rebinding. |
| Untrusted text | State, discussion, and question text enter HTML templates. | Text is escaped before the small Markdown renderer inserts its own tags. Links accept only HTTP(S), relative, or fragment targets and use `noopener noreferrer`. |
| Agent visual | An HTML visual can contain scripts. | The iframe has `sandbox="allow-scripts"` without same-origin access. `/visual` blocks network connections, external scripts, fonts, and images through CSP; visual instructions require self-contained assets. |
| Resource use | A local or LAN client could send arbitrarily large action bodies. | POST bodies are capped at 1 MiB and each send at 100 actions. Oversize requests return 413 without an agent event. |
| Local files | New session files could be readable by other local accounts on POSIX. | New session directories use mode 0700 and new state/event files use mode 0600. |
| Browser launch | Passing a URL through the Windows command shell could interpret shell characters in an explicit host. | The default Windows opener now invokes `rundll32.exe` directly with a URL argument; custom `--open-command` also uses direct process arguments. |
| Response leakage | LAN URL tokens might leak through referrals or cached HTTP responses. | All responses use `Referrer-Policy: no-referrer`, `Cache-Control: no-store`, and `X-Content-Type-Options: nosniff`. |

## Remaining boundaries

- LAN traffic is plain HTTP. The bearer URL is visible to the destination
  machine and to observers of that network path. Use LAN mode only on a trusted
  network, and do not put secrets in a grill. TLS and user authentication are
  outside this local skill's scope.
- `--lan` binds all interfaces so localhost keeps working. A VPN or public
  interface may also expose the listener if the host firewall permits it; the
  bearer token is required on non-loopback requests on those interfaces too.
- The token appears in the LAN browser history and `server.json`. Treat both
  as sensitive. The token expires when `serve` restarts.
- Existing session directories retain their current filesystem permissions;
  the permission change applies when new sessions are created. Windows ACLs
  follow the user's profile and system defaults.
- A compromised process under the same local user can access loopback and
  session files. The skill does not isolate processes from their own user.
- CI uses GitHub maintained actions by version tag and read-only repository
  permissions. Actions are not pinned to commit SHAs, so a future tag update
  remains a supply-chain trust decision.

## Verification

- Targeted Node tests covered origin and Host rejection, oversized input,
  theme writes, visual response headers, LAN URL generation, state patch
  validation, and general discussion appends.
- Markdown tests covered HTML escaping, unsafe links, and parity between the
  module and page implementation.
- A desktop browser smoke test covered rendering, general discussion staging,
  send and reply, width persistence, reload persistence, and dark mode.
- On Windows, `serve --open` reached the ready state and invoked the default
  URL handler. The launched external window was not observable through the
  available browser test surface, so this confirms the attempt, not that a
  particular browser displayed the page.
- The full Playwright browser suite was unavailable locally because its
  Chromium executable is not installed. The project's browser CI remains the
  release gate for the complete end-to-end scenario.
