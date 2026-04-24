# claude-sandbox

Run [Claude Code](https://claude.com/claude-code) inside a Docker container with optional NVIDIA GPU passthrough. Your current directory is mounted at the same path inside the container, and `~/.claude` is shared so your login, project memory, and history persist across runs.

The sandbox runs Claude with `--dangerously-skip-permissions`, which is the point: you trade filesystem isolation for a permission-free agent loop.

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

Requires Docker. For `--gpu`, requires the NVIDIA Container Toolkit.

## Usage

```sh
claude-sandbox                    # run in current directory
claude-sandbox --gpu              # with GPU passthrough
claude-sandbox --resume           # resume last conversation
claude-sandbox --resume <id>      # resume specific conversation
claude-sandbox -p "do the thing"  # pass a prompt
```

Any flags not recognized by the wrapper are forwarded to `claude`.

## Configuration

Environment variables:

| Variable | Default | Purpose |
| --- | --- | --- |
| `CLAUDE_SANDBOX_SHM` | `24g` | `--shm-size` passed to Docker |
| `CLAUDE_SANDBOX_BASE_IMAGE` | `nvidia/cuda:12.6.3-runtime-ubuntu24.04` | Base image for the build |

## What gets mounted

- `$(pwd)` → same path inside the container (so Claude's per-project memory keys correctly)
- `$HOME/.claude` → `/home/ubuntu/.claude`
- `/etc/localtime` (read-only)

The container uses `--network host`.

## Security note

This is **workflow isolation, not a security boundary.** The container has:

- Your working directory mounted read-write
- Your `~/.claude` mounted read-write (credentials, history, settings)
- Host networking
- `--dangerously-skip-permissions` enabled for Claude

Don't run it against code or instructions you don't trust. If you need a real boundary, run it on a disposable VM.

## License

MIT — see [LICENSE](LICENSE).
