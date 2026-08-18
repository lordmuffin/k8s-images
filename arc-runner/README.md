# arc-runner

A custom [Actions Runner Controller](https://github.com/actions/actions-runner-controller)
(ARC) image that can execute `np-home-homelab`'s deploy workflows.

## Why this exists

`np-home-homelab` deploys run on `runs-on: self-hosted`, which today resolves to
either of two runners: a long-lived LXC pet VM (`np-home-homelab-runner-01`), or
an ARC scale set (`homelab-runners`) on the `k3s2` cluster with `minRunners: 0`,
ephemeral pods. The LXC box is a single point of failure — it went offline once
and took CI with it — but ARC currently runs the stock
`ghcr.io/actions/actions-runner` image, and the deploy workflows' dependency-check
steps fall back to `sudo apt-get install` for anything missing. On the stock image
that mostly works (see below), but `rsync` and `kubectl` are genuinely absent, so
ARC pods would claim jobs and then fail them.

This image adds exactly what's missing so ARC can run those workflows unmodified,
so the LXC runner stops being load-bearing for every homelab deploy.

## What's already in the base image (don't reinstall)

`ghcr.io/actions/actions-runner:2.336.0` is built from
[`images/Dockerfile`](https://github.com/actions/runner/blob/v2.336.0/images/Dockerfile)
in `actions/runner`, checked directly at that tag rather than assumed. Its final
layer already installs:

| Already present | Source |
|---|---|
| `git`, `curl`, `jq`, `unzip` | `apt install` in the base image |
| `sudo`, **with `runner` already in the `sudo` group and passwordless `NOPASSWD:ALL`** | `apt install sudo` + `usermod -aG sudo runner` + `/etc/sudoers` write, all in the base image |
| `ca-certificates` | inherited from `mcr.microsoft.com/dotnet/runtime-deps:8.0-noble` |
| `tar`, `gzip`, `base64` (coreutils) | Ubuntu `noble` essential packages |

This is a meaningful correction to the usual assumption that the stock runner
image is bare-minimum: it isn't. In particular, **passwordless `sudo` is already
configured for the `runner` user** — this image does not touch `/etc/sudoers` or
install `sudo` itself.

## What this image adds, and why

| Tool | Why | Pin |
|---|---|---|
| `openssh-client` (`ssh`, `ssh-keyscan`) | Every deploy workflow SSHes to Proxmox hosts / bare-metal nodes. Not in the base image. | latest from `apt` at build time (not a security-sensitive version-pinned tool) |
| `rsync` | Manifests are rsynced to nodes before `kubectl apply`. Not in the base image. | latest from `apt` |
| `kubectl` | `deploy-k3s-dgx-spark` and `deploy-k3s-andrew-gamingpc` run `kubectl` directly on the runner (LAN-local reach to the k3s2 API VIP). | **v1.36.3**, checksum-verified. The cluster is `v1.36.2+k3s1`; kubectl's ±1 minor skew policy makes 1.36 correct. |
| `op` (1Password CLI) | Baked in so the common path (`1password/install-cli-action`) doesn't need a network round-trip. | **v2.30.0** — matches the exact version the consuming workflows pin via `1password/install-cli-action`'s `version:` input. GPG-verified against 1Password's published signing key. |

### A correction worth flagging

The usual reason given for baking in a CLI tool that a GitHub Action also
installs is "the action needs root to install it." That's not true here: I read
[`1password/install-cli-action`'s installer source](https://github.com/1Password/install-cli-action/blob/main/src/op-cli-installer/github-action/cli-installer/linux.ts)
— it downloads via `@actions/tool-cache` into the runner's own tool-cache
directory and calls `core.addPath()`. No system path, no privilege escalation.
`op` is baked into this image purely to skip a redundant download on the common
path, not because anything requires root for it. Since passwordless `sudo` was
already present in the base image anyway (see above), this finding didn't change
anything about the image — it just means the original "why sudo" reasoning in
the original request was wrong, even though the conclusion (ship sudo) already
held for a different reason (it's already there).

## What's deliberately left out

Audited directly against the real `deploy-*.yml` workflows (not just a summary of
them) with:

```bash
grep -oE '\b(jq|rsync|kubectl|ssh|curl|git|tar|base64|op|flux|helm|tofu|docker|pct)\b' \
  .github/workflows/deploy-*.y*ml | sort | uniq -c | sort -rn
```

- **`flux`, `pct`** — appear only inside `ssh ... <<REMOTE` heredocs or after
  `pct exec`, executed on the Proxmox hosts. Never invoked on the runner itself.
- **`docker` / `docker-compose`** — every mention in `deploy-lxc.yaml` is inside
  a `REMOTE` heredoc block. Zero direct `docker` CLI use on the runner.
- **`helm`** — same pattern, remote-only via `flux`/`pct exec`.
- **`tofu`** (OpenTofu) — runs directly on the runner in
  `deploy-core-services.yml`, but is installed on demand by
  `opentofu/setup-opentofu@v1` in that workflow. No need to bake it in.

Resisting `flux`/`docker`/`helm`/`pct` is the main reason this image stays small.
If a future workflow needs one of them to run on the runner directly (not inside
an `ssh`/`pct exec` block), it needs to be added deliberately here — don't assume
it's already covered.

## Bumping the base runner version

1. Find the new release tag at [`actions/runner` releases](https://github.com/actions/runner/releases).
2. Get its digest from the GHCR registry API:
   ```bash
   TOKEN=$(curl -s "https://ghcr.io/token?service=ghcr.io&scope=repository:actions/actions-runner:pull" | jq -r .token)
   curl -s -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/vnd.oci.image.index.v1+json" \
     https://ghcr.io/v2/actions/actions-runner/manifests/<NEW_VERSION> -D - -o manifest.json \
     | grep -i docker-content-digest
   jq '.manifests[] | select(.platform.architecture=="amd64" and .platform.os=="linux") | .digest' manifest.json
   ```
3. Diff [`images/Dockerfile`](https://github.com/actions/runner/blob/main/images/Dockerfile)
   at the new tag against what this README assumes is already present (sudo, git,
   curl, jq, unzip, ca-certificates) — a base image change could add or drop one
   of those.
4. Update the `FROM` tag and the header comment in `arc-runner/Dockerfile` with
   both digests and the retrieval date.
5. If the k3s2 cluster's Kubernetes version has moved, re-check kubectl's ±1
   minor skew and bump the `kubectl` version + checksum
   (`https://dl.k8s.io/release/<VERSION>/bin/linux/amd64/kubectl.sha256`) to match.

## Making the GHCR package public

`lordmuffin` is a personal GitHub account, not an org — the REST API's
package-visibility endpoint (`PATCH /orgs/{org}/packages/...`) only exists for
org-owned packages, so this can't be scripted into the workflow. After the first
successful push, go to the package settings page and flip it manually:

```
https://github.com/users/lordmuffin/packages/container/arc-runner/settings
```

Danger Zone → Change visibility → Public. Without this, the k3s2 nodes need an
`imagePullSecret` that doesn't currently exist, and pulls will fail.

## Building

Manual trigger only, via GitHub Actions → **Build ARC runner image** →
Run workflow, with a `tag` input (e.g. `2026-08-18`). Publishes:

- `ghcr.io/lordmuffin/arc-runner:<tag>`
- `ghcr.io/lordmuffin/arc-runner:<commit-sha>`

The job summary prints the exact tag to pin in the consuming repo.

## Verifying locally

```bash
docker build -t arc-runner:test ./arc-runner

docker run --rm --entrypoint bash arc-runner:test -c '
  id -u                 # 1001, not 0
  for b in git ssh ssh-keyscan rsync kubectl jq curl tar base64 sudo op; do
    command -v "$b" >/dev/null && echo "ok   $b" || echo "MISSING $b"
  done
  sudo -n true && echo "ok   passwordless sudo" || echo "MISSING passwordless sudo"
  kubectl version --client -o json | jq -r .clientVersion.gitVersion
  test -x /home/runner/run.sh && echo "ok   run.sh intact" || echo "MISSING run.sh"
'
```

Every line should report `ok`, `kubectl` should print `v1.36.3`, and uid should
be `1001`.

The real test is end-to-end in `np-home-homelab`: point
`helmrelease-runners.yaml` at the new image tag with the LXC runner stopped, and
confirm `deploy-k3s-dgx-spark` completes on an ARC pod. Expect the first pull to
be slow on a cold k3s2 node — the LXC nodes use the native containerd
snapshotter, which unpacks layers serially.

## Consuming this image

In `np-home-homelab/apps/ci/arc/helmrelease-runners.yaml`:

```yaml
  values:
    githubConfigUrl: "https://github.com/lordmuffin/np-home-homelab"
    githubConfigSecret: arc-github-app-secret
    runnerScaleSetName: "homelab-runners"
    scaleSetLabels:
      - self-hosted
    minRunners: 0
    maxRunners: 5
    template:
      spec:
        containers:
          - name: runner
            image: ghcr.io/lordmuffin/arc-runner:<TAG>
            command: ["/home/runner/run.sh"]
```

The container **must** be named `runner` and use that `command` — the chart and
the ARC controller both depend on it.
