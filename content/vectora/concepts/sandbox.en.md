---
title: Sandbox
weight: 5
---

Sandbox is Vectora's isolation layer for the agent's terminal and filesystem access. It follows the
design of Fabio Akita's [ai-jail](https://github.com/akitaonrails/ai-jail) — the agent's actions run
_inside_ the jail, not behind a single sandboxed function call — adapted to Vectora's architecture:
a single long-running backend process serving many concurrent workspaces, rather than one process
per invocation.

## Threat model

Without a sandbox, `terminal`, `file_write`, and `file_edit` operate directly in the backend process
— the agent (or a prompt injection) can read `~/.ssh`, exfiltrate secrets, or run destructive
commands with the same privileges as the backend itself. Sandbox exists to bound that blast radius
to a workspace's declared `rw_paths`/`ro_paths`, without giving up file and terminal access entirely
(that would defeat the point of an autonomous coding agent).

## Activation per workspace

If the workspace has **no** `vectora.toml` and you have not set a policy, Vectora turns the sandbox
on automatically when the environment supports native isolation (`bwrap` on Linux, WSL2 on Windows);
without that support, and always on macOS, it stays off. If the workspace has a `vectora.toml`
without a `[sandbox]` section, the sandbox stays off and the workspace behaves as before this
feature existed.

### Who sets the policy

The policy that **widens** what the jail can do is yours, not the repository's. A cloned repository
can ship any `vectora.toml`, so Vectora keeps your policy **outside the workspace**, in
`~/.vectora/sandbox-policies/` (one file per workspace), and edits it through the
`GET`/`PUT /workspaces/{id}/sandbox/policy` API (writing requires a trusted workspace).

The repository's `[sandbox]` can only **restrict**:

- turn the sandbox and `lockdown` on, require full isolation, add masks, turn GPU off;
- narrow `rw_paths`, `ro_paths`, `env_pass`, `allow_hosts`, `allow_tcp_ports` and `git_writable`,
  and move the network to a more closed mode.

Anything it asks for that **widens** access (a new writable path, `env_pass`, `git_writable`, a more
open network, another `backend`, `docker_image`, `remote_host`) is ignored, logged, and listed in
`ignored_repo_keys` in the policy response. Turning off a sandbox you turned on is ignored too.

```toml
# User policy (the same table the repository may use to restrict)
[sandbox]
enabled = true
rw_paths = ["/data"]             # writable inside the jail, besides the workspace
ro_paths = []                    # read-only
mask = [".env", "**/*.pem", "**/.ssh/**", "**/.aws/**"]
env_pass = ["NPM_TOKEN"]         # host variables the jail may inherit
git_writable = false             # true allows writing .git/hooks and .git/config
network = "filtered"             # off | filtered | open
allow_hosts = ["api.example.com", "*.cdn.example.com"]
registry_hosts = true            # pip, npm, cargo, go, maven, composer and GitHub
lockdown = false                 # true: no network, restricted resource profile
require_full_isolation = false   # true: do not start without seccomp and Landlock
```

Malformed TOML, or a field with the wrong type (for example a string where a list is expected),
fails **closed** (locked down) instead of running unprotected.

## What the jail sees

- The system (`/usr`, `/bin`, `/lib`, `/lib64`, `/etc`) read-only, with the host's symlinks, and a
  `/tmp` of its own.
- The Python virtual environment and the `backend` package, read-only. The rest of the Vectora
  repository is not exposed.
- An empty tmpfs `$HOME`. Nothing from your home shows up.
- **Clean environment:** only `PATH`, the jail's `HOME`, `LANG`, `TERM`, `TZ` and what you allow in
  `env_pass`. Provider keys and backend tokens never reach the jail.
- A new session with no controlling terminal, and `no_new_privs`.
- `.git/hooks` and `.git/config` read-only (a hook written by the agent would run outside the jail
  on your next `git`); commits and branches keep working.

## Resource limits

Every process in the jail inherits limits a runaway agent cannot exceed:

| Field             | Default  | Effect                                                                |
| ----------------- | -------- | --------------------------------------------------------------------- |
| `max_file_mb`     | `4096`   | Largest file the jail writes (`RLIMIT_FSIZE`)                         |
| `memory_limit_mb` | no limit | Per-process memory ceiling (`RLIMIT_AS`), when declared               |
| `tmp_size_mb`     | `1024`   | Size of the jail's `/tmp` and `$HOME` (tmpfs); needs a recent `bwrap` |

`RLIMIT_NPROC`, `RLIMIT_NOFILE` and `RLIMIT_CORE=0` also apply (with stricter profiles under
`lockdown`). The repository's `vectora.toml` can only **lower** these limits. There is no CPU limit
or cgroup: without privileges, Linux per-process resource limits are what exists.

## Network

The jail has no network route of its own. In the default `filtered` mode it only reaches an egress
proxy that runs outside the jail and decides every HTTPS connection (`CONNECT`) against the
allowlist: package registries (if `registry_hosts` is on) plus your `allow_hosts`. The proxy refuses
private, loopback and metadata addresses, only accepts ports 80 and 443, connects to the already
validated IP (it does not resolve the name again, which guards against DNS rebinding) and records
every decision. Plain HTTP is refused.

| Mode       | Effect                                                                 |
| ---------- | ---------------------------------------------------------------------- |
| `filtered` | Only the allowed destinations, through the proxy (default)             |
| `off`      | No network                                                             |
| `open`     | The host network, unfiltered. Use only when needed; anything can leave |

`lockdown = true` always cuts the network. On WSL2 the distro cannot reach the proxy, so the network
stays open and Vectora logs a warning. Egress policy details for tools are in
[Egress policies](../../guides/egress-policies).

## How it works: one worker per workspace

Instead of spawning a new sandboxed process for every tool call, Vectora keeps a single persistent
jailed worker process alive per workspace with `[sandbox]` enabled — born on the first sandboxable
action, killed when the workspace goes idle. `file_write`, `file_edit`, and one-shot `terminal`
commands route through this worker via a small JSON-lines protocol
(`exec`/`read_file`/`write_file`).

On the `local` backend (Linux, or Windows through WSL2), the worker runs under `bwrap` when
available (Bubblewrap) with:

- Filesystem namespacing restricted to `rw_paths`/`ro_paths`.
- Masked paths (`mask`) never exposed inside the jail, even if they fall under an allowed path.
- Landlock and resource limits applied by the worker itself.
- A seccomp-BPF filter (via `pyseccomp`/`libseccomp`) denying dangerous syscalls: `ptrace`, `mount`,
  `unshare`, `setns`, `bpf`, kernel modules, `io_uring`, `userfaultfd`, `keyctl`, `perf_event_open`,
  `ioctl(TIOCSTI)` and namespace creation through `clone`. Without `libseccomp` the worker runs
  without the filter and isolation is **partial**; with `require_full_isolation = true` it does not
  start at all.
- If the workspace folder or policy changes, the worker is recreated on the next action.

On Windows, automatic activation checks for an eligible WSL2 distribution, not for `bwrap` inside
it. If `bwrap` is missing, the worker falls back to a native soft process with no filesystem or
network isolation, even with `lockdown = true`. macOS does not automatically enable isolation: the
persistent local worker attempts `bwrap`, not Seatbelt (`sandbox-exec`).

Soft mode does not qualify for a sandbox HITL exemption. Until a live worker with effective
isolation has been confirmed, ordinary approval rules apply; after HITL relies on native isolation,
a soft replacement is refused.

Other backends (`docker`, `ssh`, `modal`) run the same worker model against their respective
isolation primitive; `docker` additionally denies network access under `lockdown`.

## MCP servers

A `stdio` MCP server that Vectora has not verified (community-listed, unsigned, or with verification
unavailable) **only starts inside the jail**, with no option to run loose on the computer. Because
the MCP protocol is bidirectional, the server runs as a child process of the client, in the same
jail as the workspace: read-only system, empty `$HOME`, no backend keys, seccomp, Landlock,
`no_new_privs`, resource limits and a filtered network. Details:

- The server's working folder is its own, in `~/.vectora/mcp-sandbox/<name>`.
- The environment variables you approved for the server reach it; nothing else from the backend's
  environment does.
- The program has to be visible: what lives on the system (`/usr`, `/bin`) already is; what lives in
  user folders (nvm, pyenv, uv) has its install folder mounted read-only.
- The network is the filtered one: package registries plus the server's `allow_hosts` field
  (`domain.com` or `*.domain.com`).
- Without `bwrap` (for example, outside Linux) the server **does not start**, instead of falling
  back to an unprotected process. An external launcher, if configured, still takes precedence.

## State, audit and preview

In **Settings → Sandbox** you see what isolation guarantees, dimension by dimension (files, network,
environment, seccomp, Landlock, `no_new_privs`, resources and `.git`), measured on the running jail
or estimated when it has not started, and never marked effective without confirmation. The same
screen edits your policy, shows what the repository asked for and was ignored, displays the `bwrap`
command preview (nothing is executed, values coming from the host are hidden) and the audit.

The audit is a hash-chained `~/.vectora/sandbox-audit.jsonl` file: each record carries the hash of
the previous one, so altering, removing or reordering a line breaks the chain and the verification
points to where. It records the jail start (with the policy fingerprint and isolation state),
refusals, every allowed or blocked outbound connection and policy changes. Secrets are masked.
Terminal approval is only waived when the jail is alive, natively isolated and the network is under
control (off or filtered).

## Interactive terminal

The Terminal tab (PTY) runs in the same `bwrap` jail of the `local` backend, with the same clean
environment, the same seccomp, Landlock and limits, and the same filtered network. On the `docker`,
`ssh` and `modal` backends it is not supported yet: starting the interactive terminal returns a
clear error rather than falling back to an unprotected shell.

## See also

- [Using the Workbench](../../guides/using-the-workbench) — the Terminal and Files tabs in practice
- [Sessions & Workspaces](../sessions-and-workspaces) — how a workspace is trusted and scoped
