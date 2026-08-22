# Generic base: official Microsoft devcontainers "base" image — a minimal
# Debian image with a non-root "vscode" user, sudo, git, and curl already
# set up, and no language-specific toolchain baked in. Swap this for a more
# specific base, or add a toolchain in the "project-specific toolchain"
# section below, depending on the project. The
# ghcr.io/anthropics/devcontainer-features/claude-code feature (see
# devcontainer.json) installs its own Node.js runtime automatically if the
# base image doesn't already ship one, so Claude Code itself works
# regardless of which base you pick.
FROM mcr.microsoft.com/devcontainers/base:bookworm

ARG TZ
ENV TZ="$TZ"

# Tools needed by init-firewall.sh (iptables/ipset/dig/aggregate/jq).
# `aggregate` (for collapsing CIDR ranges) isn't in Debian's default repos
# under that exact package name on all releases; if this fails on first
# build, replace with the `netaddr`-based aggregate6 equivalent or drop the
# GitHub-IP-range aggregation step in the firewall script and allow the
# ranges unaggregated (more ipset entries, same effect).
RUN apt-get update && apt-get install -y --no-install-recommends \
  iptables \
  ipset \
  iproute2 \
  dnsutils \
  aggregate \
  jq \
  && apt-get clean && rm -rf /var/lib/apt/lists/*

# --- Project-specific toolchain goes here ---
# Examples (uncomment/adapt as needed, and add matching domains to
# init-firewall.sh's allowlist so package installs can actually reach the
# network):
#
# Python:
# RUN apt-get update && apt-get install -y --no-install-recommends \
#   python3 python3-pip python3-venv \
#   && apt-get clean && rm -rf /var/lib/apt/lists/*
# Add to init-firewall.sh: pypi.org, files.pythonhosted.org
#
# Node/TypeScript beyond what the Claude Code feature already installs:
# RUN apt-get update && apt-get install -y --no-install-recommends nodejs npm \
#   && apt-get clean && rm -rf /var/lib/apt/lists/*
# Add to init-firewall.sh: registry.npmjs.org (already in the base list below)
#
# JDK + Kotlin/Gradle:
# RUN apt-get update && apt-get install -y --no-install-recommends default-jdk \
#   && apt-get clean && rm -rf /var/lib/apt/lists/*
# Or swap the FROM above for a JDK-preloaded base, e.g.
# mcr.microsoft.com/devcontainers/java:21-bookworm.
# Add to init-firewall.sh: repo1.maven.org, repo.maven.apache.org,
# plugins.gradle.org, services.gradle.org
# If the build needs a persistent Gradle cache, add a PROJECT_SLUG-gradle-cache
# volume in devcontainer.json's "mounts" (see the commented example there) and
# chown /home/vscode/.gradle below alongside .claude.
#
# Android (on top of the JDK setup above) — Google only ships cmdline-tools
# as a zip, so `unzip` needs to be in the apt-get install list too:
# ENV ANDROID_HOME=/opt/android-sdk
# ENV ANDROID_SDK_ROOT=/opt/android-sdk
# ENV PATH="${ANDROID_HOME}/cmdline-tools/latest/bin:${ANDROID_HOME}/platform-tools:${PATH}"
# RUN mkdir -p "${ANDROID_HOME}/cmdline-tools" \
#   && curl -fsSL -o /tmp/cmdline-tools.zip https://dl.google.com/android/repository/commandlinetools-linux-15859902_latest.zip \
#   && unzip -q /tmp/cmdline-tools.zip -d "${ANDROID_HOME}/cmdline-tools" \
#   && mv "${ANDROID_HOME}/cmdline-tools/cmdline-tools" "${ANDROID_HOME}/cmdline-tools/latest" \
#   && rm /tmp/cmdline-tools.zip \
#   && chown -R vscode:vscode "${ANDROID_HOME}"
# USER vscode
# RUN yes | sdkmanager --licenses > /dev/null 2>&1 || true \
#   && sdkmanager --install "platform-tools" "platforms;android-<N>" "build-tools;<N>.0.0"
# USER root
# (check compileSdk/build-tools versions against the project's build.gradle.kts —
# don't guess them here)
# Persist the SDK across rebuilds with a PROJECT_SLUG-android-sdk volume
# mounted at /opt/android-sdk (see devcontainer.json), and chown it below too.
# Add to init-firewall.sh: dl.google.com, maven.google.com, androidx.dev
#
# Apple Silicon + Android: Google's build tools (aapt2 in particular) are
# x86_64-only. If builds fail with an exec-format error, force the whole
# container to x86_64 via "options": ["--platform=linux/amd64"] in
# devcontainer.json's "build" block (relies on Docker Desktop's "Use Rosetta
# for x86/amd64 emulation on Apple Silicon" setting for reasonable speed).

# Copy and set up the firewall script (see init-firewall.sh for the
# allowlisted domains — extend the domain list there for whatever package
# registries the project's toolchain needs).
#
# Deliberately NOT installed as /usr/local/bin/init-firewall.sh: the
# ghcr.io/anthropics/devcontainer-features/claude-code feature (see
# devcontainer.json) unconditionally installs its own generic
# init-firewall.sh at that exact path as one of its build steps, which runs
# *after* this Dockerfile stage — so anything placed there here gets
# silently clobbered by the feature on every build, cache or no cache.
# Using a distinct filename sidesteps the collision instead of fighting the
# feature's build-stage ordering.
COPY init-firewall.sh /usr/local/bin/project-init-firewall.sh
RUN chmod +x /usr/local/bin/project-init-firewall.sh && \
  echo "vscode ALL=(root) NOPASSWD: /usr/local/bin/project-init-firewall.sh" > /etc/sudoers.d/vscode-firewall && \
  chmod 0440 /etc/sudoers.d/vscode-firewall

# The mcr.microsoft.com/devcontainers/base image grants vscode unrestricted
# passwordless root (/etc/sudoers.d/vscode: "NOPASSWD:ALL") as a
# dev-convenience default. That defeats the firewall above, since vscode
# could just `sudo iptables -F` it away — remove it so vscode's only
# passwordless root path is the one line above.
RUN rm -f /etc/sudoers.d/vscode

# Persist shell history the same way Anthropic's reference container does.
RUN mkdir -p /commandhistory && \
  echo "export PROMPT_COMMAND='history -a' && export HISTFILE=/commandhistory/.bash_history" >> /home/vscode/.bashrc && \
  touch /commandhistory/.bash_history && \
  chown -R vscode:vscode /commandhistory /home/vscode/.bashrc

# PROJECT_SLUG-claude-code-config (devcontainer.json's "mounts") is
# seeded from whatever ownership this path has in the image at container
# creation. Without this, the named volume comes up root-owned and the
# vscode user can't write its own login credentials into it after OAuth.
#
# If you add other persistent named-volume caches in devcontainer.json's
# "mounts" (e.g. ~/.gradle, ~/.npm, ~/.cache/pip, /opt/android-sdk), give
# each one the same treatment here — a volume mounted under a path the
# image didn't already chown comes up root-owned and vscode can't write to
# it.
RUN mkdir -p /home/vscode/.claude && chown -R vscode:vscode /home/vscode/.claude

USER vscode
WORKDIR /workspace
