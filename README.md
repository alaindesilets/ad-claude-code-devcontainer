# Claude Code sandbox (template)

A reusable [devcontainer](https://containers.dev/) template for bootstrapping
new projects with [Claude Code](https://www.anthropic.com/claude-code), inside of VS Code. The container is
hardened so that Claude, running inside the container, cannot affect anything
outside of it.

This template is a generic starting point for that sandbox — copy it into a
new project, add the toolchain that project needs, and go.

Details on the design of the container can be found further down this document.

## Getting started

### Using this template for a new project

1. Copy `.devcontainer/` into the new project's root.
2. Add a toolchain — see [Adding a toolchain](#adding-a-toolchain-eg-python)
   below.
3. Open the project folder in VS Code and reopen in container (see
   [Opening this in VS Code](#opening-this-in-vs-code) below), or run
   `devcontainer up --workspace-folder .` via the
   [`@devcontainers/cli`](https://github.com/devcontainers/cli).

### Opening this in VS Code

1. Make sure Docker (e.g. Docker Desktop) is installed and running.
2. Install the **Dev Containers** extension in VS Code (`ms-vscode-remote.remote-containers`)
   — search for "Dev Containers" in the Extensions panel if it's not already
   installed.
3. Open the project folder (the one containing `.devcontainer/`) in VS Code
   normally, like any other folder.
4. VS Code usually notices the `.devcontainer/` folder on its own and shows a
   pop-up in the bottom-right corner: **"Folder contains a Dev Container
   configuration file. Reopen in Container?"** — click **Reopen in
   Container**.
   - If the pop-up doesn't show up (or you dismissed it), open the Command
     Palette (**Cmd+Shift+P**) and run **Dev Containers: Reopen in
     Container**.
5. The first time, VS Code builds the Docker image and starts the container
   — this can take a few minutes (installing packages, setting up the
   firewall, etc.). Subsequent opens are much faster since the image is
   already built.
6. Once it's done, VS Code's window reloads and is now running *inside* the
   container: the integrated terminal, file explorer, and any command you
   run are all happening inside the sandbox, not on your Mac. You can tell
   because the bottom-left corner of the window shows something like
   **"Dev Container: Claude Code sandbox (template)"** instead of nothing.
7. To leave the container and go back to working on your Mac directly,
   click that same bottom-left corner and choose **Reopen Folder Locally**.

### Adding a toolchain (e.g. Python)

A "toolchain" is whatever language/runtime the project needs — Python, Node,
a JVM, etc. Adding one always means touching **two** files, because two
different things happen at two different times:

- **`Dockerfile` — installs the tool itself.** This runs once, during
  `docker build`, before the container ever starts. The firewall doesn't
  exist yet at this point (it's set up later, when the container starts), so
  this step always has full network access regardless of the allowlist.
- **`init-firewall.sh` — lets that tool reach the internet later, while the
  container is running.** Once the container is up, *everything* outbound is
  blocked except an explicit allowlist of domains. So even though `pip` gets
  installed fine in the step above, running `pip install pandas` *inside the
  running container* would silently hang or fail — because `pypi.org` isn't
  on the allowlist yet. This step adds it.

Skip the second file and you'll end up with a Python that's installed but
can never install a package.

**Worked example: adding Python.**

In `.devcontainer/Dockerfile`, find the `# --- Project-specific toolchain
goes here ---` section and uncomment/add:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
  python3 python3-pip python3-venv \
  && apt-get clean && rm -rf /var/lib/apt/lists/*
```

In `.devcontainer/init-firewall.sh`, find the `for domain in ...` loop near
the bottom and add Python's package-registry domains — remember, only the
*last* entry in the list has no trailing `\`:

```bash
for domain in \
    "registry.npmjs.org" \
    "api.anthropic.com" \
    "sentry.io" \
    "statsig.anthropic.com" \
    "statsig.com" \
    "marketplace.visualstudio.com" \
    "vscode.blob.core.windows.net" \
    "update.code.visualstudio.com" \
    "pypi.org" \
    "files.pythonhosted.org"; do
```

Rebuild the container, then confirm both halves actually work:

```bash
docker exec -u vscode <container> python3 --version   # tool is installed
docker exec -u vscode <container> pip install --user requests  # tool can reach the network
```

If the second command hangs or errors out, the domain probably wasn't added
correctly (or resolved to nothing at container start — check the
`init-firewall.sh` output for a `WARNING: Failed to resolve ...` line).

Other toolchains follow the same two-file pattern; a few more are already
sketched as comments in both files (Node/TypeScript beyond what the Claude
Code feature installs, and a JVM/Gradle setup with Maven/Gradle domains).

If the toolchain needs a persistent cache (e.g. `~/.gradle`, `~/.npm`), add a
named volume for it in `devcontainer.json`'s `mounts`, and `chown` its mount
point in the `Dockerfile` (see the comment above the `.claude` chown line
there — an uncowned volume mount comes up root-owned and is unwritable by
`vscode`).

### Testing a build locally

```bash
npx --yes @devcontainers/cli up --workspace-folder .
```

This builds the image, starts the container, and runs the firewall script
via `postStartCommand`, including its own self-check (fails the build if the
firewall isn't actually blocking/allowing what it should).

To poke around and confirm the hardening is holding:

```bash
# find the running container
docker ps

# .devcontainer/ should be read-only from inside
docker exec -u vscode <container> sh -c 'echo x >> /workspace/.devcontainer/devcontainer.json'
# -> Read-only file system

# vscode should have no general sudo access
docker exec -u vscode <container> sudo -l
# -> only project-init-firewall.sh listed
```

Docker assigns the running container a random name (e.g. `nervous_clarke`)
unless one is pinned with `--name` in `devcontainer.json`'s `runArgs`.

## Design

### Genesis

This template started as Anthropic's reference devcontainer (built
specifically for Node.js/JS projects). We stripped the Node/JS-specific parts
to make it generic, then closed two gaps found in the original:

1. **Claude could disable the firewall.** The base image gives the `vscode`
   user passwordless `sudo ALL`, which is enough to run `sudo iptables -F`
   and wipe the firewall rules. Fixed by removing the blanket sudo grant and
   replacing it with a NOPASSWD rule scoped to only the firewall script.
2. **Claude could modify `devcontainer.json` (and the rest of `.devcontainer/`).**
   Since `.devcontainer/` lives inside the bind-mounted workspace, it was
   writable from inside the container — Claude could plant a weakened config
   (e.g. drop the firewall's `postStartCommand`) that would silently take
   over on the next rebuild. Fixed by re-mounting `.devcontainer/` read-only
   inside the container, nested over the writable workspace bind. Claude can
   still read those files; only writes through the container's mount point
   are blocked. They remain normal, freely editable files on the host.

### What's in the box

| File | Purpose |
|---|---|
| `.devcontainer/devcontainer.json` | Devcontainer config: build args, mounts, VS Code settings, `postStartCommand` that runs the firewall. |
| `.devcontainer/Dockerfile` | Base image (`mcr.microsoft.com/devcontainers/base:bookworm`), firewall tooling, sudo hardening, marked section for project-specific toolchains. |
| `.devcontainer/init-firewall.sh` | Default-deny outbound firewall (iptables + ipset). Everything is blocked except an explicit domain allowlist. |

### Security model

- **Network**: default-deny outbound firewall. Only an explicit allowlist
  (GitHub IP ranges, npm, `api.anthropic.com`, VS Code, Sentry/Statsig
  telemetry, plus whatever you add for your toolchain) is reachable. Verified
  automatically at container start (checks that `example.com` is unreachable
  and `api.github.com` is reachable — the script fails loudly if either
  check doesn't come out as expected).
- **Filesystem**: only the project workspace is bind-mounted into the
  container (`/workspace`). `.devcontainer/` is additionally re-mounted
  read-only, so its own config can't be tampered with from inside. Claude
  config (`~/.claude`) and shell history live in separate named Docker
  volumes so they persist across rebuilds without exposing anything else on
  the host.
- **Privilege**: the `vscode` user has no general sudo access — only a single
  NOPASSWD rule to run the firewall script. It cannot flush firewall rules,
  install arbitrary packages as root, or otherwise escalate.
