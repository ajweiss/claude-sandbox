# claude-sandbox

Run [Claude Code](https://claude.com/claude-code) with `--dangerously-skip-permissions` inside a Docker container, so the agent works without interactive permission prompts. **The container is the point**: it bounds the blast radius enough to make disabling those prompts acceptable. If Claude (or something prompt-injected into it) does something you wouldn't have approved, the damage is contained to the container's view of the host — your project tree and `~/.claude` — instead of the rest of your machine.

Your current directory is mounted at the same path inside the container, and `~/.claude` is shared so your login, project memory, and history persist across runs. Optional NVIDIA GPU passthrough.

## What you get

- CUDA 12.6 runtime base image (configurable)
- Python 3 + `uv`, Node 20, `just`, plus common utilities (`git`, `curl`, `jq`, `rsync`, `openssh-client`)
- Host UID/GID matched so files created in the container are owned by you
- Image is tagged by Dockerfile hash and rebuilt automatically when the Dockerfile changes

## Install

```sh
git clone https://github.com/YOUR-USER/claude-sandbox.git
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

## What gets mounted

- `$(pwd)` → same path inside the container (so Claude's per-project memory keys correctly)
- `$HOME/.claude` → `/home/ubuntu/.claude` (read-write, with read-only overlays — see below)
- `/etc/localtime` (read-only)

### Read-only overlays inside `~/.claude`

The base `~/.claude` mount is read-write so credentials, project memory, transcripts, and history persist. On top of that, the wrapper layers read-only bind mounts over the paths that double as code/instruction-execution surfaces:

- `settings.json`, `settings.local.json`
- `agents/`, `commands/`, `hooks/`, `plugins/`

Each is mounted only if it exists on the host. This blocks the main host-impacting attack: a compromised agent inside the sandbox writing a poisoned hook, slash command, subagent definition, or plugin that fires the next time you run Claude (sandboxed or otherwise) on this host.

Trade-off: you can't install plugins, edit settings, or author new slash commands / subagents *from inside the sandbox*. Do those from a host shell.

## Container hardening

The wrapper applies a few defaults to keep the blast radius down:

- `--security-opt=no-new-privileges` — blocks setuid- and file-cap-based privilege escalation inside the container.
- `--cap-drop=ALL` — drops all Linux capabilities. Claude runs as a regular user (`ubuntu`), so it doesn't need any.
- Bridge networking by default (see above).
- Read-only overlays on `~/.claude` execution surfaces (see above).

## Security note

This is **workflow isolation, not a security boundary.** Even with the hardening above, the container still has:

- Your working directory mounted read-write
- Read access to all of `~/.claude`, including credentials, transcripts, and history
- `--dangerously-skip-permissions` enabled for Claude
- A shared kernel with the host (it's a container, not a VM)

A compromised agent can still read your Claude credentials and read/write anything in the project tree. The read-only overlays prevent it from installing a hook that fires on your host, but they don't prevent exfiltration. Don't run it against code or instructions you don't trust. If you need a real boundary, run it on a disposable VM (Incus, QEMU, a cloud instance) or a VM-per-container runtime like Kata Containers.

## License

MIT — see [LICENSE](LICENSE).
