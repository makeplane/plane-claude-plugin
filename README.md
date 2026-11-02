# Plane plugins for Claude Code

The official [Plane](https://plane.so) plugin marketplace for Claude Code.

| Plugin | What it does |
| --- | --- |
| [`plane`](./plugins/plane) | Work with your Plane Cloud workspace from Claude Code through Plane's hosted MCP server: work items, cycles, modules, intake, pages and more. |
| [`plane-self-hosting`](./plugins/plane-self-hosting) | Deploy, upgrade and troubleshoot self-hosted Plane, Community or Commercial, on your own infrastructure. Claude probes your Docker host or Kubernetes cluster, asks which install method and add-ons you want, then installs and verifies. |

## Install

In a Claude Code session (2.1.275 or later):

```text
/plugin install plane-self-hosting --marketplace makeplane/plane-claude-plugin
/plugin install plane --marketplace makeplane/plane-claude-plugin
```

Or from your shell:

```bash
claude plugin marketplace add makeplane/plane-claude-plugin
claude plugin install plane-self-hosting@makeplane
claude plugin install plane@makeplane
```

Updates arrive through the marketplace. Run `claude plugin marketplace update makeplane`, then
`claude plugin update plane-self-hosting@makeplane`.

## Developing

```bash
claude plugin validate .                                  # marketplace + every plugin
claude --plugin-dir ./plugins/plane-self-hosting          # try a plugin without installing it
```

Each plugin's version lives in its `.claude-plugin/plugin.json`. Bump it with every change you
want installed users to receive.
