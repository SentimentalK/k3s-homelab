# cats-toilet (俺家的猫厕所 Cat's Toilet)

- URL: <https://cats.mainstayx.com>
- Namespace: `cats-toilet`
- ArgoCD Application: `app-cats-toilet` (auto-sync from `apps/cats-toilet`)
- Node: `mtl0` (arm64); `local-path` PVC `cats-toilet-data` (1Gi)
- Health: `/healthz`; API under `/api` (contract: `server/API.md` in the
  cats-toilet repo); live events: `/api/events` (SSE)

One Node 24 process, no npm dependencies, serves the static game and the JSON
API (accounts, cloud saves, plaza presence, visits, gifts) on port 8080.

## Image

The image is **not** pulled from a registry. It is built for arm64 on the
amd64 workstation and imported into the k3s containerd store on `mtl0`. The
Dockerfile has no `RUN` step, so the cross-build needs no emulation:

```sh
docker buildx build --platform linux/arm64 --load \
  -f /path/to/k3s-homelab/apps/cats-toilet/Dockerfile \
  -t cats-toilet:arm64 /path/to/cats-toilet

docker save cats-toilet:arm64 |
  ssh mtl0 'sudo k3s ctr images import -'

ssh mtl0 'sudo k3s kubectl -n cats-toilet rollout restart deploy/cats-toilet'
```

`Dockerfile.dockerignore` next to the Dockerfile keeps the raw art
(`assets/raw/`), design notes, tools, tests and any local database out of the
build context.

## Persistence / lifecycle

- `cats.db` (SQLite, WAL) lives on the `local-path` PVC; `strategy: Recreate`
  and 1 replica prevent two writers.
- Who is online, who is squatting and the event streams live in the
  process's memory: a restart puts everyone offline until their client's next
  heartbeat (10 s) and EventSource reconnect.
- Pinned to `mtl0`: the image is imported only there and the PVC is node-local.
- The root filesystem is read-only; the process runs as uid 1000 (`node`).

## TLS / DNS

`cats.mainstayx.com` is covered by the existing Cloudflare wildcard and
cert-manager `letsencrypt-prod` / Traefik ingress. Sign-up rate limiting
reads the client address from `CF-Connecting-IP`.

## Known limits of this prototype

- Saves are client-authoritative: the server stores what the client sends and
  does not validate points, levels or items.
- A gift is not debited from the sender on the server.
- Login tokens do not expire; only logout revokes one.
- The game's URL cheats (`?coins=`, `?level=`, `?items=all`, `?reset=1`) work
  here too.
