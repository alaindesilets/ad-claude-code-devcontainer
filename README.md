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
2. Set the project's name: in `.devcontainer/devcontainer.json`, change the
   top-level `"name"` field from `"Claude Code sandbox (template)"` to
   something identifying the new project (e.g. `"MyProject — Claude Code
   sandbox"`). This is what VS Code shows in the bottom-left corner once
   you're in the container — left as-is, every project copied from this
   template looks identical there.
3. Replace the `PROJECT_SLUG` placeholder in `.devcontainer/devcontainer.json`'s
   `mounts` (search for it — a few lines, all together) with a short slug for
   the project (e.g. `myproject`). This names the named Docker volumes that
   hold Claude Code's config/auth and shell history. It needs to be done only
   once, and — unlike the running container's own name, which is already
   derived automatically per folder via `runArgs` — must stay identical
   across every git worktree of this project, so they all share the same
   Claude Code login and history. See
   [Running multiple agents in parallel](#running-multiple-agents-in-parallel-git-worktrees)
   below if that's not immediately obvious why.
4. (Optional) Copy some VS Code settings from settings.json.template to the 
   project's .vscode/settings.json
5. Add a toolchain — see [Adding a toolchain](#adding-a-toolchain-eg-python)
   below.
6. Open the project folder in VS Code and reopen in container (see
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

### Running multiple agents in parallel (git worktrees)

Claude Code supports multiple simultaneous instances sharing the same
`~/.claude` config, so you can run several agents in parallel on the same
project, each on its own branch, each in its own container — as long as each
one works in its own [git worktree](https://git-scm.com/docs/git-worktree)
rather than all sharing one checkout. This template supports that pattern
out of the box, with two caveats already accounted for above:

- **Container names don't collide.** `runArgs` derives the container name
  from the opened folder (`${localWorkspaceFolderBasename}-claude-sandbox`),
  so each worktree gets a distinct container automatically.
- **Config/history stay shared, not duplicated.** The `PROJECT_SLUG`-named
  volumes (see step 3 above) are the same across every worktree, so you
  don't have to re-authenticate Claude Code or lose shell history each time
  you spin up a new worktree's container.

What's *not* automatic is git itself working inside a worktree's container —
see the next point — and telling worktree windows apart visually.

**Workflow:**

1. **Convention: keep worktrees as sibling folders of the main repo**, under
   one common parent directory (e.g. `~/Documents/myproject`,
   `~/Documents/myproject-feature-x`, `~/Documents/myproject-bugfix-y`).
   The mount below assumes this layout.
2. **One-time per project:** in `.devcontainer/devcontainer.json`'s
   `mounts`, uncomment the git-worktree mount line near the bottom and
   replace `MAIN_REPO_DIR_NAME` with the main repo's actual folder name.
   This is needed because a worktree's `.git` is just a text file pointing
   at an *absolute host path* inside the main repo's `.git` — invisible
   inside the container, which only mounts the worktree's own folder, so
   without this every git command in the worktree fails with `fatal: not a
   git repository: (null)`. The mount makes that absolute path resolve
   correctly inside the container too, no matter which worktree is open.
   One mount line covers every worktree of the project — nothing to repeat
   per worktree.
3. **Create a worktree:** `git worktree add ../myproject-feature-x feature-x`
   (from inside the main repo).
4. **Open a new VS Code window** on that worktree's folder and **Reopen in
   Container** (see [Opening this in VS Code](#opening-this-in-vs-code)
   above) — this builds/starts a separate container for it.
5. **(Optional) Tell the windows apart at a glance:** copy
   `.vscode/settings.json.template` to that worktree's own
   `.vscode/settings.json` if you haven't already, and uncomment/set a
   different `workbench.colorCustomizations` title bar color for this
   window than your other open worktrees.
6. **Run `claude` in the integrated terminal** — it works side by side with
   Claude Code instances running in your other worktrees/containers of the
   same project, sharing config but isolated in every worktree's own
   sandboxed container and branch.

To remove a worktree once you're done with it: close its VS Code
window/container, then from the main repo run
`git worktree remove ../myproject-feature-x`.

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
Code feature installs, a JVM/Gradle setup with Maven/Gradle domains, and —
nested under that — an Android SDK setup with its own persistent-volume and
Apple-Silicon notes).

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

The container's name is pinned via `--name` in `devcontainer.json`'s
`runArgs` (derived from the opened folder's name), so `docker ps` shows
something identifiable instead of a random Docker-assigned name like
`nervous_clarke`.

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
