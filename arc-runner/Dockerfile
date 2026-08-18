# ARC runner image for the np-home-homelab deploy workflows.
#
# Base: ghcr.io/actions/actions-runner:2.336.0
#   Retrieved 2026-08-18 via the GHCR registry API (anonymous token).
#   Multi-arch index digest: sha256:0cfdcc701ce933c6d243c6b0b2da767366dc9f2e99961d4c3754b0b78084cdda
#   linux/amd64 manifest digest: sha256:a1919047b038c38871d667c58cfdc7a878452711ab1212fb6036188f27a7ab16
#
# Already present in the base image — verified against actions/runner's
# images/Dockerfile at the v2.336.0 tag, do not reinstall:
#   git, curl, jq, unzip, ca-certificates, sudo (runner is already in the
#   `sudo` group with passwordless NOPASSWD:ALL preconfigured in /etc/sudoers),
#   tar/gzip/coreutils (Ubuntu noble essentials, hence base64).
#
# Added below, and why (see arc-runner/README.md for the full audit):
#   openssh-client (ssh, ssh-keyscan) — every deploy workflow SSHes to Proxmox
#     hosts / bare-metal nodes; not in the base image.
#   rsync — manifests are rsynced to nodes before kubectl apply; not in the base.
#   kubectl v1.36.3 — cluster is v1.36.2+k3s1, kubectl's +/-1 minor skew policy
#     makes 1.36 correct. deploy-k3s-dgx-spark and deploy-k3s-andrew-gamingpc
#     run kubectl straight on the runner (LAN-local reach to the k3s2 VIP).
#   1Password CLI v2.30.0 — pinned to match the exact version
#     1password/install-cli-action@v2.0.0 installs in the consuming workflows
#     (`version: "2.30.0"`), so only one op version is ever in play. Baked in to
#     avoid a redundant download on the common path — NOT because the action
#     needs root: its own installer uses @actions/tool-cache + core.addPath(),
#     no privilege escalation, confirmed by reading its source.
#   gnupg — transient build-time dependency to verify op's GPG signature against
#     1Password's published signing key; purged in the same layer afterward.
#
# Deliberately NOT installed: flux, helm, docker, pct, tofu. flux/pct/docker
# only ever appear inside `ssh ... <<REMOTE` / `pct exec` heredocs that run on
# the Proxmox hosts, never on the runner itself (verified against the actual
# deploy-*.yml workflows). tofu is installed on-demand by
# opentofu/setup-opentofu@v1 in the one workflow that needs it.
#
# linux/amd64 only — the k3s2 nodes running ARC pods are amd64 LXC containers.

FROM ghcr.io/actions/actions-runner:2.336.0

USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends openssh-client rsync gnupg \
    && rm -rf /var/lib/apt/lists/*

# kubectl v1.36.3, checksum-verified against the published release digest.
RUN curl -fLo /usr/local/bin/kubectl \
      "https://dl.k8s.io/release/v1.36.3/bin/linux/amd64/kubectl" \
    && echo "ebbd080e7c2e275093b55915722043257eb24004363e20acb3c4d71919f88336  /usr/local/bin/kubectl" | sha256sum -c - \
    && chmod +x /usr/local/bin/kubectl

# 1Password CLI v2.30.0, GPG-verified against 1Password's published signing key
# (same key + fingerprint 1password/install-cli-action verifies against).
RUN curl -fLo /tmp/op.zip \
      "https://cache.agilebits.com/dist/1P/op2/pkg/v2.30.0/op_linux_amd64_v2.30.0.zip" \
    && curl -fLo /tmp/1password.asc "https://downloads.1password.com/linux/keys/1password.asc" \
    && unzip -q /tmp/op.zip -d /tmp/op \
    && gpg --import /tmp/1password.asc \
    && gpg --list-keys --with-colons | grep -q "3FEF9748469ADBE15DA7CA80AC2D62742012EA22" \
    && gpg --verify /tmp/op/op.sig /tmp/op/op \
    && install -o root -g root -m 0755 /tmp/op/op /usr/local/bin/op \
    && rm -rf /tmp/op /tmp/op.zip /tmp/1password.asc /root/.gnupg \
    && apt-get purge -y gnupg \
    && apt-get autoremove -y \
    && rm -rf /var/lib/apt/lists/*

USER runner
