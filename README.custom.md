# Custom Docker Setup

Workarounds for running OpenClaw in Docker on WSL2. Addresses known issues with the upstream `docker-setup.sh`:

- DNS resolution fails inside containers ([#14593](https://github.com/openclaw/openclaw/issues/14593))
- Skill installs fail — brew not available, Go too old via apt, npm permissions ([#3987](https://github.com/openclaw/openclaw/issues/3987))
- Gateway crash-loops during onboard due to `allowedOrigins` chicken-and-egg

## What's changed

### `docker-compose.yml`

Added explicit DNS servers to `openclaw-gateway` so the CLI container (which shares its network namespace) can resolve external hosts:

```yaml
dns:
  - 8.8.8.8
  - 1.1.1.1
```

### `Dockerfile.local`

Extends `openclaw:local` with:

- **Homebrew (Linuxbrew)** so brew-only skills install without manual workarounds
- **Go 1.24** from the official tarball (Debian apt ships 1.19 which is too old for most Go-based skills)
- **User-writable npm global prefix** so `npm install -g` works as the non-root `node` user

## Usage

```bash
# 1. Build the base image first
docker build -t openclaw:local -f Dockerfile .

# 2. Build the custom layer on top, tagged as openclaw:local
# docker-setup.sh only skips pulling when the image is named "openclaw:local"
docker build -t openclaw:local -f Dockerfile.local .

# 3. Run setup
export OPENCLAW_DOCKER_APT_PACKAGES=gh
export OPENCLAW_HOME_VOLUME=openclaw-home
./docker-setup.sh
```

## Post-setup skill install

The gateway crash-loops during onboard (before `allowedOrigins` is set), so skill installs may fail during the wizard. Skip them and install after setup completes:

```bash
docker compose run --rm openclaw-cli onboard-skills
```

Or install individual tools directly:

```bash
# npm-based skills
docker compose run --rm --entrypoint sh openclaw-cli -c "npm install -g @xdevplatform/xurl clawhub mcporter"

# Go-based skills
docker compose run --rm --entrypoint sh openclaw-cli -c "go install github.com/Hyaxia/blogwatcher/cmd/blogwatcher@latest"
```

## Notes

- `OPENCLAW_HOME_VOLUME` persists `/home/node` so installed binaries survive container restarts.
- Homebrew is included in the image so all skill install methods (brew, npm, go) work out of the box. Adds ~500MB to the image.

## Upstream issues

- [#14593](https://github.com/openclaw/openclaw/issues/14593) — Skill install fails in Docker: brew not installed
- [#3987](https://github.com/openclaw/openclaw/issues/3987) — Feature: Include brew install in Dockerfile
- [#12622](https://github.com/openclaw/openclaw/issues/12622) — Docker unable to install skills
- [#4130](https://github.com/openclaw/openclaw/issues/4130) — Skill install errors during docker setup
