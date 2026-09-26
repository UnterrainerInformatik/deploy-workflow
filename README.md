# deploy-workflow

Reusable GitHub Actions workflow that deploys a docker-compose based service to a server via SSH.

It copies the repo's `./deploy/` folder to the target server, writes a `.env` file with version and image info, and runs `up.sh` there. If VPN credentials are passed, it first connects to the target network: via WireGuard if `WG_CONFIG` is passed, otherwise via OpenVPN if `VPN_OVPN_FILE` is passed. Without either it deploys via SSH directly.

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
| `ovpn_enabled` | no | `false` | Deprecated, ignored. The VPN is picked from the passed secrets |
| `wg_enabled` | no | `false` | Deprecated, ignored. The VPN is picked from the passed secrets |

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

### VPN (optional)

The VPN is picked automatically from the secrets that are passed (non-empty):

1. `WG_CONFIG` → WireGuard
2. otherwise `VPN_OVPN_FILE` → OpenVPN
3. otherwise no VPN, SSH goes directly to `DEPLOY_SERVER`

| Secret | Description |
|---|---|
| `WG_CONFIG` | Full content of the WireGuard client `.conf` file |
| `VPN_OVPN_FILE` | Full content of the OpenVPN client `.ovpn` file (ignored if `WG_CONFIG` is passed) |
| `VPN_USERNAME` | OpenVPN user, if the server requires user/password auth |
| `VPN_PASSWORD` | OpenVPN password, if the server requires user/password auth |

The config is written to `/etc/wireguard/wg-deploy.conf` and brought up as interface `wg-deploy`. It is always torn down and deleted at the end of the job, even if the deployment fails.

Notes for `WG_CONFIG`:

- **Restrict `AllowedIPs` to the target network** (e.g. `AllowedIPs = 10.10.196.0/22`). Configs exported from a UniFi gateway default to `0.0.0.0/0`, which would route all runner traffic through the tunnel while the job runs.
- `DNS = ...` lines are stripped automatically, so the runner's DNS is left unchanged. Use an IP for `DEPLOY_SERVER` or make sure the runner can resolve it.
- **Use one WireGuard peer (key) per runner.** Two runners using the same config at the same time will disconnect each other.
- The runner needs the WireGuard kernel module (included in current Ubuntu kernels). Containerized runners need `NET_ADMIN` (or run privileged). Missing `wireguard-tools` and `iproute2` (`ip`, used by `wg-quick`) are installed via apt.

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
