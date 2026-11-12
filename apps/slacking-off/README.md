# slacking-off (Company Time)

- URL: <https://slacking-off.mainstayx.com>
- Namespace: `slacking-off`
- ArgoCD Application: `app-slacking-off` (auto-sync from `apps/slacking-off`)
- Node: `mtl0` (arm64); `local-path` PVC `slacking-off-data` (1Gi)
- Health: `/healthz`; public projection: `/api/projection`; SSE: `/api/watch`

## Image

The image is **not** pulled from a registry. It is built on/for arm64 and imported
into the k3s containerd store on `mtl0`:

```sh
# on an amd64 workstation, with buildx + qemu-aarch64:
docker buildx build --platform linux/arm64 --load \
  -f /path/to/k3s-homelab/apps/slacking-off/Dockerfile \
  -t slacking-off:arm64 /path/to/slacking-off

docker save slacking-off:arm64 |
  ssh mtl0 'sudo k3s ctr images import -'

ssh mtl0 'sudo k3s kubectl -n slacking-off rollout restart deploy/slacking-off'
```

`Dockerfile` in this directory is the native-arm64 variant for building directly
on `mtl0`; the deployed image also needs `loo-linux-x64` in the build context so
the account page can serve both Linux CLIs.

## Persistence / lifecycle

- `world.db` lives on the `local-path` PVC; `strategy: Recreate` and 1 replica
  prevent two SQLite writers.
- `--official` mode is fixed in the image CMD; the server writes a snapshot on
  SIGTERM (`terminationGracePeriodSeconds: 30`).
- Pin to `mtl0`: the image is imported only there and the PVC is node-local.

## TLS / DNS

`slacking-off.mainstayx.com` is covered by the existing Cloudflare wildcard and
cert-manager `letsencrypt-prod` / Traefik ingress. The server honours
`X-Forwarded-Proto` so install commands and cookies use HTTPS.
