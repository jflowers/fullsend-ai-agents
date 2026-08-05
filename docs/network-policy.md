# Custom network policy

Agents run inside an OpenShell sandbox that restricts outbound network
access. If a package install or build step fails because the sandbox
blocks a host your project needs, create a custom policy that adds the
missing endpoints.

## How the sandbox policy works

The policy controls two things:

- **Endpoints** -- which hosts and ports a process can connect to.
- **Binaries** -- which executables are allowed to use each endpoint.

A request is allowed only when both the destination host and the calling
binary match an entry in the policy. Even if a host is in the allowlist,
only the listed binaries can reach it. `curl` is excluded from every
policy to prevent raw HTTP access with the injected GitHub token.

Each agent has its own default policy under
[`policies/`](../policies/). The defaults cover Vertex AI, the GitHub
API, package registries, and gitleaks releases. See the individual
policy files for the full list of allowed hosts and binaries.

## Before you start

- You have a custom harness (`.fullsend/<agent>.yaml`) in your
  repository for the agent you want to configure.
- You know which host is being blocked. Check the agent logs for
  `ECONNREFUSED`, `ETIMEDOUT`, `npm ERR! network`, or similar
  connection errors that name the blocked hostname.

## Add custom endpoints to the policy

### 1. Create a policy file

Copy the default policy for the agent you are configuring (for example,
[`policies/code.yaml`](../policies/code.yaml) for the code agent) to
`.fullsend/policies/<agent>.yaml` in your repository and append a
block for the host you need:

```yaml
network_policies:
  other_policies: ...
  your_policy:
    name: custom-hosts
    endpoints:
      - host: "example.com"
        port: 443
        protocol: rest
        enforcement: enforce
        access: read-only
    binaries:
      - path: "**/node"
      - path: "**/npm"
```

Use `access: read-only` unless the endpoint requires write access.

### 2. Reference the policy in your harness

Add the `policy:` field to `.fullsend/<agent>.yaml`:

```yaml
# .fullsend/<agent>.yaml
base: https://raw.githubusercontent.com/fullsend-ai/agents/<SHA>/harness/<agent>.yaml#sha256=<sha256sum>
policy: policies/<agent>.yaml
```

If you already have a custom harness (for example, with a
[custom sandbox image](code.md#custom-sandbox-image)), add the
`policy:` field alongside the existing fields.

### 3. Repeat for other agents

Each agent has its own policy file with similar but not identical
defaults. If multiple agents need the same custom hosts, create a
separate override for each one. For example, the
[code](code.md) and [fix](fix.md) agents use
[`policies/code.yaml`](../policies/code.yaml) and
[`policies/fix.yaml`](../policies/fix.yaml) respectively.

## Troubleshooting

When a package install fails due to a blocked host, look for connection
errors in the agent logs:

| Symptom | Cause |
|---------|-------|
| `ECONNREFUSED` or `ETIMEDOUT` on a hostname | Host is not in the policy |
| `npm ERR! network` followed by a hostname | npm cannot reach the registry or download URL |
| `pip` or `go get` failure referencing an external URL | Download host is not in the policy |

To identify which host to add:

1. Find the failing command in the agent log output.
2. Look for the hostname in the error message (for example,
   `codeload.github.com`).
3. Add the host to your custom policy under an appropriate
   `network_policies` entry with the binaries that need access.

## See also

- [Code agent](code.md) -- custom harness and sandbox image setup
- [Fix agent](fix.md) -- fix agent overview
- [`policies/`](../policies/) -- default network policies for all
  agents
