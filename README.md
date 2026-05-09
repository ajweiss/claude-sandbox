# claude-sandbox

Run [Claude Code](https://claude.com/claude-code) in a Docker container with permission prompts disabled (`--dangerously-skip-permissions`).

Your current directory is mounted at the same path inside the container, and `~/.claude` is shared so your login, project memory, and history persist across runs — with read-only overlays on the hook/plugin/command paths (`settings.json`, `agents/`, `commands/`, `hooks/`, `plugins/`) so a compromised sandbox run can't poison them for later host runs. Optional NVIDIA GPU passthrough.

## What you get

- CUDA 12.6 runtime base image (configurable)
- Python 3 + `uv`, Node 20, `just`, plus common utilities (`git`, `curl`, `jq`, `rsync`, `openssh-client`)
- Host UID/GID matched so files created in the container are owned by you
- Image is tagged by Dockerfile hash and rebuilt automatically when the Dockerfile changes

## Install

```sh
git clone https://github.com/ajweiss/claude-sandbox.git
cd claude-sandbox
cp claude-sandbox ~/.local/bin/
```

Put `Dockerfile.claude-sandbox` in one of:

- Next to the script (simplest)
- `~/.config/claude-sandbox/Dockerfile`
- `~/.claude/sandbox/Dockerfile`

Requires Docker. For `--gpu`, see the [GPU support](#gpu-support) section below.

### GPU support

`--gpu` requires the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/) installed on the host and registered with Docker:

1. Install the host NVIDIA driver. Verify with `nvidia-smi`.
2. Install the NVIDIA Container Toolkit:
   - Arch: `pacman -S nvidia-container-toolkit`
   - Debian/Ubuntu, Fedora/RHEL: install from NVIDIA's container-toolkit repo (per the upstream docs).
3. Register the runtime with Docker and restart it:
   ```sh
   sudo nvidia-ctk runtime configure --runtime=docker
   sudo systemctl restart docker
   ```
4. Sanity check:
   ```sh
   docker run --rm --gpus all nvidia/cuda:12.6.3-runtime-ubuntu24.04 nvidia-smi
   ```

The base image ships CUDA 12.6 runtime by default; the host driver must be new enough for that CUDA version. To change the base image, set `CLAUDE_SANDBOX_BASE_IMAGE`.

## Usage

```sh
claude-sandbox                    # run in current directory (bridge networking)
claude-sandbox --gpu              # with GPU passthrough
claude-sandbox --host-net         # share the host's network namespace
claude-sandbox --microvm          # use Kata Containers as the runtime (Linux)
claude-sandbox --resume           # resume last conversation
claude-sandbox --resume <id>      # resume specific conversation
claude-sandbox -p "do the thing"  # pass a prompt
claude-sandbox --help             # wrapper-specific help
claude-sandbox -- --help          # forward --help to claude itself
```

Any flags not recognized by the wrapper are forwarded to `claude`. `--help` and `-h` are intercepted by the wrapper; use `--` as a sentinel to pass arguments straight through (e.g. `claude-sandbox -- --help` for Claude's own help).

### Networking

The container uses **bridge networking by default**, so `127.0.0.1` and `::1` inside the container are the container's own loopback — not the host's. Host services aren't reachable on `localhost` from inside.

To reach a service running on the host, use the hostname `host.docker.internal` (the wrapper wires this up automatically). For example, `http://host.docker.internal:11434` for Ollama.

If you genuinely need host networking — for instance, an MCP server or other tool that only binds to `127.0.0.1` and you can't change it — pass `--host-net`. That restores the previous behavior and exposes the host's loopback to the container.

## Configuration

Environment variables:

| Variable | Default | Purpose |
| --- | --- | --- |
| `CLAUDE_SANDBOX_SHM` | `24g` | `--shm-size` passed to Docker |
| `CLAUDE_SANDBOX_BASE_IMAGE` | `nvidia/cuda:12.6.3-runtime-ubuntu24.04` | Base image for the build |
| `CLAUDE_SANDBOX_RUNTIME` | unset | Docker runtime to pass via `--runtime` (e.g. `kata`, `runsc`). Lets you make a microVM / userspace-kernel runtime your default without typing `--microvm` each time. |

## What gets mounted

- `$(pwd)` → same path inside the container (so Claude's per-project memory keys correctly)
- `$HOME/.claude` → `/home/ubuntu/.claude` (read-write, with read-only overlays — see below)
- `$HOME/.claude.json` → `/home/ubuntu/.claude.json` (read-write, if present). This sibling of `~/.claude/` holds account, subscription, and onboarding state; without it Claude re-runs the OAuth flow on every startup.
- `/etc/localtime` (read-only)

### Read-only overlays inside `~/.claude`

The base `~/.claude` mount is read-write so credentials, project memory, transcripts, and history persist. On top of that, the wrapper layers read-only bind mounts over the paths that double as code/instruction-execution surfaces:

- `settings.json`, `settings.local.json`
- `agents/`, `commands/`, `hooks/`, `plugins/`

Each is mounted only if it exists on the host. This blocks the main host-impacting attack: a compromised agent inside the sandbox writing a poisoned hook, slash command, subagent definition, or plugin that fires the next time you run Claude (sandboxed or otherwise) on this host.

Trade-off: you can't install plugins, edit settings, or author new slash commands / subagents *from inside the sandbox*. Do those from a host shell.

## microVM isolation (`--microvm`)

The default container shares its kernel with your host. If you want a real KVM boundary instead, pass `--microvm` (or set `CLAUDE_SANDBOX_RUNTIME=kata`). The wrapper then asks Docker to use [Kata Containers](https://katacontainers.io/) as the runtime — `docker run` boots a small KVM microVM, the container runs inside it, and the workflow otherwise stays the same.

This is **Linux only.** On macOS and Windows, Docker Desktop already runs the daemon inside a Linux VM, so containers are *already* isolated from the host kernel by a VM boundary; adding Kata on top would be VM-in-VM and pointless.

Setup on the host (Linux):

1. Install Kata (e.g. `pacman -S kata-containers` on Arch; check your distro for equivalents).
2. Make sure `/dev/kvm` is accessible (CPU virtualization on; nested virt enabled if you're already in a VM).
3. Register the runtime with Docker by adding a `runtimes` entry to `/etc/docker/daemon.json` and restarting Docker. Sanity check:
   ```sh
   docker run --rm --runtime=kata hello-world
   ```

GPU + microVM caveat: Kata's stock guest image doesn't include NVIDIA drivers, and host-side GPU passthrough needs `intel_iommu=on` / `amd_iommu=on` and the GPU bound to `vfio-pci`. If you have one GPU shared with your desktop, that's awkward; a dedicated compute GPU makes it tolerable. See the Kata docs for the full setup.

## Container hardening

The wrapper applies a few defaults to keep the blast radius down:

- `--security-opt=no-new-privileges` — blocks setuid- and file-cap-based privilege escalation inside the container.
- `--cap-drop=ALL` — drops all Linux capabilities. Claude runs as a regular user (`ubuntu`), so it doesn't need any.
- Bridge networking by default (see above).
- Read-only overlays on `~/.claude` execution surfaces (see above).

## Security note

This is **workflow isolation, not a security boundary** by default. Even with the hardening above, the container still has:

- Your working directory mounted read-write
- Read access to all of `~/.claude`, including credentials, transcripts, and history
- `--dangerously-skip-permissions` enabled for Claude

A compromised agent can read your Claude credentials and read/write anything in the project tree. The read-only overlays prevent it from installing a hook that fires on your host, but they don't prevent exfiltration. Don't run it against code or instructions you don't trust.

### Kernel boundary

By default, the container shares its kernel with the host. A container escape (kernel exploit, container-runtime bug) reaches the host. The platform you're running on changes how much that matters:

- **macOS / Windows**: Docker Desktop runs the daemon inside a Linux VM, so the host kernel is *already* isolated by a VM boundary. The shared-kernel concern doesn't really apply on these platforms.
- **Linux, default runtime (`runc`)**: shared kernel with your host. The mitigations in [Container hardening](#container-hardening) reduce the surface but don't eliminate it.
- **Linux, `--microvm`**: each `docker run` boots a small KVM microVM via [Kata Containers](https://katacontainers.io/), and the container runs inside that VM. Now there's a real kernel boundary between the agent and your host. This addresses the shared-kernel problem; it doesn't change the credentials/project-tree exposure listed above.

If you're running untrusted code or instructions on Linux and want the strongest available boundary, use `--microvm` (or `CLAUDE_SANDBOX_RUNTIME=kata`). For the strongest possible isolation, run the whole sandbox on a disposable VM or cloud instance — that limits even credential and project-tree exposure to a throwaway environment.

## License

MIT — see [LICENSE](LICENSE).
