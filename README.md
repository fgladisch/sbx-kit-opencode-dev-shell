# OpenCode development shell kit

A standalone Docker Sandboxes kit for [OpenCode V2](https://opencode.ai/v2/docs/) with Zsh, [fnm](https://github.com/Schniz/fnm), Node.js 24 and 26, [pnpm](https://pnpm.io/), TypeScript and ESLint language servers, and architecture-native Chromium.

The kit builds on `docker/sandbox-templates:shell-docker` rather than the official `opencode` kit. That kit ships OpenCode 1 and manages `~/.config/opencode/opencode.json` for its MCP gateway, while this kit runs OpenCode V2 with your own configuration and logins.

## What it does

During sandbox creation, the kit:

- installs the latest OpenCode V2 release to `~/.opencode/bin` and configures it as the sandbox entrypoint;
- loads the repository's `AGENTS.md` through the kit's `agentInstructions`;
- installs Zsh, Ubuntu's `build-essential` toolchain, and fnm;
- installs the latest Node.js 24 and 26 releases with fnm and makes 26 the default;
- installs pnpm, the TypeScript language server with TypeScript 6, and the ESLint language server into the image's global npm prefix;
- downloads the sipgate root CA from `ca.live.sipgate.net` and adds it to the system trust store, so internal services such as the datapls MCP server pass TLS verification;
- installs Playwright's pinned, architecture-native Chromium build and its system libraries;
- exposes OpenCode, fnm, Node.js, Chromium, pnpm, and the language servers on the sandbox `PATH`;
- binds the `github`, `sipgate_os`, `sonarqube`, and `langsmith` sandbox secrets to proxy-managed `GH_TOKEN`, `SG_AI_PROXY_API_KEY`, `SONARQUBE_TOKEN`, and `LANGSMITH_API_KEY` environment variables;
- initializes fnm from `~/.zshenv`.

### Node.js versions

fnm selects the Node.js version in every Zsh, including the non-interactive shells that run OpenCode's shell commands. On start and on every directory change, it reads the nearest:

- `.node-version`;
- `.nvmrc`;
- `package.json#engines.node`.

Without a version file, the shell uses the default Node.js 26. When the requested version isn't installed, the shell keeps its current version. Install other versions with `fnm install <version>`; fnm then selects them automatically.

Processes started outside Zsh, including the language servers that OpenCode launches, use the default Node.js 26. Global npm packages live in `/usr/local/share/npm-global`, so they're available with every Node.js version. The TypeScript language server uses the project's own `typescript` dependency when it has one.

## Allow the kit publisher

Docker Sandboxes restricts remote kit publishers through `kit.allowedSources`. Add `github.com/fgladisch/` while preserving any sources you already trust:

```bash
sbx settings get kit.allowedSources
sbx settings set kit.allowedSources \
  '["docker.io/","github.com/docker/","github.com/fgladisch/"]'
```

## Usage

```bash
sbx run \
  "git+https://github.com/fgladisch/sbx-kit-opencode-dev-shell.git" \
  "$PWD"
```

A sandbox started this way has the tooling but none of your OpenCode settings or logins. Use the launcher to bring them along.

### Launcher

[`bin/opencode-sbx`](bin/opencode-sbx) wraps the full per-project lifecycle. Run it from a repository to:

1. derive the sandbox name `opencode-dev-<repository>` from the current directory name;
2. create the sandbox when necessary, with the current directory as its workspace;
3. copy your OpenCode configuration, skills, and saved logins into a newly created sandbox (see [Copied host state](#copied-host-state));
4. upgrade OpenCode and its package plugins on every launch;
5. attach to OpenCode and forward every argument to OpenCode unchanged.

Link it into `~/bin`:

```bash
ln -s "$(pwd)/bin/opencode-sbx" "$HOME/bin/opencode-sbx"
```

Then launch OpenCode from any repository:

```bash
opencode-sbx
```

Every argument is passed to OpenCode, including subcommands such as `acp`. In a terminal the launcher attaches with `sbx run`. When stdin or stdout is not a terminal, as when an editor spawns it, it starts OpenCode with `sbx exec -i` instead, because `sbx run` relays no output without a TTY. Setup commands read no stdin and write to stderr, so OpenCode's stdin and stdout carry only the session. That keeps stdio-based protocols like ACP intact.

Continue the latest session, or resume a specific one:

```bash
opencode-sbx --continue
opencode-sbx --session <session-id>
```

To run OpenCode as an ACP agent in Zed, register the launcher as a custom agent server in Zed's `settings.json`:

```json
{
  "agent_servers": {
    "opencode": {
      "type": "custom",
      "command": "opencode-sbx",
      "args": ["acp"]
    }
  }
}
```

The launcher uses the published Git kit by default. Set `OPENCODE_SBX_KIT` to a local directory or another supported kit reference when testing a different build:

```bash
OPENCODE_SBX_KIT=/path/to/sbx-kit-opencode-dev-shell opencode-sbx
```

### Herdr

[Herdr](https://herdr.dev/) can't see the OpenCode process inside the sandbox, only the launcher and `sbx run` on the host. Set [`HERDR_AGENT`](https://herdr.dev/tui/docs/agents/#vms-and-sandbox-wrappers) on the launcher command in a Herdr pane so Herdr tracks it as OpenCode:

```bash
HERDR_AGENT=opencode opencode-sbx
```

The hint applies only to that command, so set it per invocation instead of exporting it. Arguments pass through as usual, for example `HERDR_AGENT=opencode opencode-sbx --continue`.

Herdr reads OpenCode's state from the screen. The Herdr OpenCode plugins copied with your configuration don't report state from the sandbox, because Herdr's pane variables and socket don't reach it. If a pane shows the wrong state, inspect the match with `herdr agent explain <pane>`.

### Copied host state

When it creates a sandbox, the launcher copies:

| Host | Sandbox |
| --- | --- |
| `~/.config/opencode` (without the background service's `service.json`) | `/home/agent/.config/opencode` |
| `~/.agents/skills` | `/home/agent/.agents/skills` |
| `~/.local/share/opencode/opencode.db` | `/home/agent/.local/share/opencode/opencode.db` |

The launcher honors `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, and `OPENCODE_DB`.

- **Symlinks.** The configuration is copied with `rsync --copy-unsafe-links`. Relative links that stay inside the configuration directory remain links, so a linked plugin source keeps its relative imports. Absolute links and links leading outside the directory, such as agents or commands from a shared repository, are replaced with copies of their targets.
- **Host paths.** Paths in the configuration files are copied unchanged, so keep them free of host-specific absolute paths. Use relative paths, or link local plugin checkouts into `~/.config/opencode/plugins/`, where OpenCode discovers them automatically:

  ```bash
  ln -s ~/code/my-plugin ~/.config/opencode/plugins/my-plugin
  ```

- **Logins.** OpenCode V2 stores provider accounts and MCP OAuth tokens in its database. The launcher copies a consistent `sqlite3 .backup` snapshot, so your Claude subscription, ChatGPT login, and OAuth MCP servers such as Atlassian, Slack, and datapls work without signing in again. The snapshot also carries your session history.

> [!WARNING]
> Each copy refreshes its tokens independently. Providers that rotate refresh tokens, including Anthropic and Atlassian, invalidate the previous token on refresh, so the host or another sandbox may occasionally need to sign in again.

Existing sandboxes keep independent configuration, logins, sessions, and skills. Later changes on the host aren't synchronized. Removing the sandbox with `sbx rm` also removes that sandbox-local state.

## Credentials

Store the four credentials used by the kit before creating a sandbox. Service secrets are global by default:

```bash
sbx secret set github --command 'gh auth token'
sbx secret set sipgate_os
sbx secret set sonarqube
sbx secret set langsmith
```

Schema v2 third-party kits also require credential bindings. Before using the non-interactive launcher, merge these approvals into `~/.config/sbx/credentials.yaml` without removing existing bindings:

```yaml
bindings:
  github:
    apiKey:
      domains:
        - api.github.com
        - github.com
        - raw.githubusercontent.com
  sipgate_os:
    apiKey:
      domains:
        - coding.sipgate.ai
  sonarqube:
    apiKey:
      domains:
        - api.sonarcloud.io
  langsmith:
    apiKey:
      domains:
        - eu.api.smith.langchain.com
```

Alternatively, run the kit interactively once and approve each requested binding at the prompt. Non-interactive `sbx create` runs with unbound credentials withheld.

Existing sandboxes retain the network policy and credential injection domains from creation time. After changing a binding or kit domain, remove and recreate the sandbox before retrying the affected service.

The sandbox receives proxy-managed sentinel values rather than the real credentials. The host-side proxy injects bearer tokens for GitHub API, sipgate, and SonarCloud requests, HTTP Basic authentication for Git over HTTPS on `github.com`, and the `X-Api-Key` header for LangSmith requests.

Provider logins for Claude and ChatGPT don't use sandbox secrets: they come from the copied OpenCode database.

## Network access

The kit allows the domains needed for:

- Ubuntu and Docker APT metadata;
- OpenCode, fnm, Node.js, npm, pnpm, and Playwright installation;
- GitHub API access and release downloads;
- OpenAI and Anthropic authentication and model endpoints, including Claude subscription token refreshes;
- the sipgate model endpoint and root CA download;
- the Atlassian, Slack, SonarCloud, LangSmith, and datapls MCP servers.

Review `permissions.network.allow` in [`spec.yaml`](spec.yaml) before use if your environment requires a narrower egress policy.

## Compatibility

The kit uses `schemaVersion: "2"` with the `kind: sandbox`, `sandbox`, `agentInstructions`, `permissions`, `credentials`, and `setup` fields. It validates with Docker Sandboxes `v0.50.1`.

Validate it locally with:

```bash
sbx kit validate .
```

## Lifecycle

All setup happens in `setup.install` during sandbox creation: OpenCode V2, Zsh, fnm, Node.js 24 and 26 with pnpm and the language servers, the sipgate root CA, and a pinned architecture-native Chromium build with Playwright's browser dependencies. The kit has no startup commands; fnm selects Node.js versions as shells start and change directories.

The installation steps are idempotent so sandbox recreation can safely repeat them.
