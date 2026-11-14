---
title: Egress policies
weight: 10
---

This guide explains how Vectora controls the agent's outbound traffic: the egress policy, which
applies to network tools, and sandbox network isolation, which applies to commands run in the
worker.

> **Status:** these capabilities are included in version 0.1.24. Live tests marked `skipped` do not
> prove behavior in a real environment.

## Prerequisites

- A workspace open in Vectora.
- For network isolation of commands: the sandbox active in the workspace (see
  [Sandbox](../../concepts/sandbox)).

Configure each layer according to the operations you need to restrict.

## Configuration

Outbound traffic is controlled by two independent layers, each configured in a different place.

### Two layers

| Layer           | Where it applies                                                                                                           | How it is configured                                   |
| --------------- | -------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| Egress policy   | Initial URLs for fetch, crawl and map, and Browser Workbench navigation; see Chromium redirect and search exclusions below | Runtime settings (`egress_policy`)                     |
| Sandbox network | Commands and files run in the jailed worker                                                                                | Your sandbox policy (the repository can only restrict) |

The MCP server allowlist and tool pre-approvals are not egress policies. They control which servers
and actions can be used, not where traffic goes.

### Egress policy

The policy is read from runtime settings: `egress_policy:<workspace_id>` holds the workspace policy
directly; alternatively, `egress_policy` maps workspace IDs to policy objects, for example
`{"my-workspace": {"allowed_hosts": ["example.com"]}}`. For a workspace, a single unkeyed policy
object under `egress_policy` is ignored and the default policy applies. There is no dedicated screen
or endpoint to edit it.

| Field                       | Default     | Effect                                                                                                            |
| --------------------------- | ----------- | ----------------------------------------------------------------------------------------------------------------- |
| `block_private_ips`         | `true`      | Blocks destinations that resolve to private, loopback, or non-public IPs                                          |
| `allowed_hosts`             | empty       | When set, only these hosts are allowed                                                                            |
| `denied_hosts`              | empty       | Hosts that are always blocked                                                                                     |
| `allowed_ports`             | `[80, 443]` | Allowed ports; an empty list does not restrict ports                                                              |
| `allow_localhost_dev_ports` | empty       | Loopback ports allowed by the egress layer; `fetch_url` still rejects loopback through its independent SSRF guard |

How the policy is evaluated:

1. Only `http` and `https` URLs are accepted.
2. `denied_hosts` is checked before `allowed_hosts`. A `*.example.com` pattern covers `example.com`
   and all its subdomains.
3. The host is resolved through DNS and every address returned by that lookup is checked. The direct
   HTTP path pins the validated IP; the Chromium fallback does not pin it, so this check does not
   guarantee protection against DNS rebinding on that path.
4. The port must be in `allowed_ports`, or in `allow_localhost_dev_ports` when the destination is
   loopback.

Subagents inherit the policy of the run that created them.

### Sandbox network

When the sandbox is active, the sandbox policy (yours, outside the repository; see
[Sandbox](../../concepts/sandbox)) controls the network of commands:

```toml
[sandbox]
enabled = true
network = "filtered"                    # off | filtered | open
allow_hosts = ["api.example.com"]       # besides package registries
registry_hosts = true
allow_tcp_ports = [443]                 # extra per-port restriction (Landlock)
```

- `network = "filtered"` (default): the jail has no network route and only reaches the egress proxy,
  which accepts HTTPS `CONNECT` to package registries and `allow_hosts`, refuses private, loopback
  and metadata addresses, only opens ports 80 and 443, and connects to the already validated IP.
  Every decision is recorded.
- `network = "off"` or `lockdown = true`: no network (`--unshare-net`).
- `network = "open"`: the host network, unfiltered.
- `allow_tcp_ports` denies all outbound TCP except the listed ports, through Landlock ABI V4 (Linux
  6.7 or later). With the list empty (the default) there is no per-port restriction.

The repository's `vectora.toml` can only close the network and narrow `allow_hosts`.

Before using these capabilities, check the platform, configuration and validation limits below.

## Limitations

- The sandbox `local` backend (bubblewrap and Landlock) runs on Linux; on Windows it depends on
  WSL2. Without WSL2, or without `bwrap` inside the distribution, the worker falls back to a soft
  mode that does not apply `--unshare-net`: with `lockdown = true`, commands still run and the
  network stays open. On WSL2 the filtered network is not available (the proxy runs on the Windows
  host): the network stays open.
- On macOS the sandbox does not turn on automatically, and the persistent worker does not use
  Seatbelt (`sandbox-exec`).
- On kernels without Landlock ABI V4, `allow_tcp_ports` is not applied; the worker logs that the
  restriction is not in effect.
- Vectora maps which sandbox backends can enforce network restrictions, but it does not yet refuse a
  run when the chosen backend cannot.
- The egress policy covers the tools listed above; do not assume other network paths use the same
  controls.
- `web_search` sends a query, not a URL: requests to the search provider and to the fallback search
  (DuckDuckGo) do not go through the egress policy.
- `fetch_url` uses Chromium when `TAVILY_API_KEY` is absent or when the extraction provider reports
  quota/rate-limit errors, even with a key configured. In both cases the initial URL is checked, but
  browser-followed redirects are not rechecked and the destination IP is not pinned.
