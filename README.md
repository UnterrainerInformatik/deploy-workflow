# deploy-workflow

Reusable GitHub Actions workflow that deploys a docker-compose based service to a server via SSH.

It copies the repo's `./deploy/` folder to the target server, writes a `.env` file with version and image info, and runs `up.sh` there. Optionally it first connects to the target network via OpenVPN or WireGuard.

Runs on self-hosted runners (`[self-hosted, Linux, X64]`) that have `sudo` and `apt`.

## Usage

```yaml
jobs:
  deploy:
    uses: UnterrainerInformatik/deploy-workflow/.github/workflows/workflow.yml@master
    with:
      major_version: ${{ needs.build.outputs.major }}
      minor_version: ${{ needs.build.outputs.minor }}
      build_version: ${{ needs.build.outputs.build }}
      wg_enabled: true
    secrets:
      DEPLOY_SSH_PRIVATE_KEY: ${{ secrets.DEPLOY_SSH_PRIVATE_KEY }}
      DEPLOY_SSH_USER: ${{ secrets.DEPLOY_SSH_USER }}
      DEPLOY_SERVER: ${{ secrets.DEPLOY_SERVER }}
      DEPLOY_SSH_PORT: ${{ secrets.DEPLOY_SSH_PORT }}
      DEPLOY_DIR: ${{ secrets.DEPLOY_DIR }}
      DATA_DIR: ${{ secrets.DATA_DIR }}
      DOCKER_HUB_USER: ${{ secrets.DOCKER_HUB_USER }}
      DOCKER_IMAGE_NAME: ${{ secrets.DOCKER_IMAGE_NAME }}
      WG_CONFIG: ${{ secrets.WG_CONFIG }}
```

If all secrets have the same names in the calling repo, `secrets: inherit` works too.

The calling repo needs a `./deploy/` folder with at least an `up.sh`.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `major_version` | yes | | Major version, written to `.env` as `MAJOR_VERSION` |
| `minor_version` | yes | | Minor version, written to `.env` as `MINOR_VERSION` |
| `build_version` | yes | | Build number, written to `.env` as `BUILD_VERSION` |
| `ovpn_enabled` | no | `false` | Connect via OpenVPN before deploying |
| `wg_enabled` | no | `false` | Connect via WireGuard before deploying |

`VERSION` is set in `.env` as `<major>.<minor>.<build>`.

## Secrets

### Deployment (always required)

| Secret | Description |
|---|---|
| `DEPLOY_SSH_PRIVATE_KEY` | Private SSH key; its public key must be in `authorized_keys` of `DEPLOY_SSH_USER` on the target |
| `DEPLOY_SSH_USER` | SSH user on the target server |
| `DEPLOY_SERVER` | Hostname or IP of the target server (for VPN deployments: the internal LAN address) |
| `DEPLOY_SSH_PORT` | SSH port of the target server |
| `DEPLOY_DIR` | Directory on the target that `./deploy/` is copied into and where `up.sh` runs |
| `DATA_DIR` | Data directory on the target, created if missing |
| `DOCKER_HUB_USER` | Docker Hub user, written to `.env` as `DOCKER_HUB_USER` |
| `DOCKER_IMAGE_NAME` | Image name, written to `.env` as `DOCKER_IMAGE_NAME` |

### OpenVPN (only with `ovpn_enabled: true`)

| Secret | Description |
|---|---|
| `VPN_OVPN_FILE` | Full content of the client `.ovpn` file |
| `VPN_USERNAME` | VPN user, if the server requires user/password auth |
| `VPN_PASSWORD` | VPN password, if the server requires user/password auth |

### WireGuard (only with `wg_enabled: true`)

| Secret | Description |
|---|---|
| `WG_CONFIG` | Full content of the WireGuard client `.conf` file |

The config is written to `/etc/wireguard/wg-deploy.conf` and brought up as interface `wg-deploy`. It is always torn down and deleted at the end of the job, even if the deployment fails.

Notes for `WG_CONFIG`:

- **Restrict `AllowedIPs` to the target network** (e.g. `AllowedIPs = 10.10.196.0/22`). Configs exported from a UniFi gateway default to `0.0.0.0/0`, which would route all runner traffic through the tunnel while the job runs.
- `DNS = ...` lines are stripped automatically, so the runner's DNS is left unchanged. Use an IP for `DEPLOY_SERVER` or make sure the runner can resolve it.
- **Use one WireGuard peer (key) per runner.** Two runners using the same config at the same time will disconnect each other.
- The runner needs the WireGuard kernel module (included in current Ubuntu kernels). Containerized runners need `NET_ADMIN`.

Example:

```ini
[Interface]
PrivateKey = <client private key>
Address = 192.168.3.2/32

[Peer]
PublicKey = <server public key>
Endpoint = <public ip or hostname>:51820
AllowedIPs = 10.10.196.0/22
PersistentKeepalive = 25
```
