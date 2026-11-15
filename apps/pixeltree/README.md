# pixeltree (try-it page)

- URL: <https://pixeltree.mainstayx.com>
- Namespace: `pixeltree`
- ArgoCD Application: `app-pixeltree` (auto-sync from `apps/pixeltree`)
- Node: `mtl0` (arm64); measured there: model load ~21 s, one 1920x1080 parse p50 10.6 s / p95 11.3 s with 2 threads, peak RSS 2.5 GB; no volumes (uploads are processed in memory only; `/tmp` is an emptyDir)
- Health: `/healthz` (503 until the models are loaded); API: `POST /api/parse` (raw PNG / JPEG / WebP body)
- Source and Dockerfile: <https://github.com/SentimentalK/pixeltree> (`demo/`, `demo/Dockerfile`), AGPL-3.0

## Image

Not from a registry. Cross-built for arm64 on an amd64 workstation (buildx + qemu-aarch64) and imported
into the k3s containerd store on `mtl0`; the tag is the pixeltree commit:

```sh
cd /path/to/pixeltree
TAG=$(git rev-parse --short HEAD)
docker buildx build --platform linux/arm64 --load -f demo/Dockerfile -t pixeltree-demo:$TAG .
docker save pixeltree-demo:$TAG | ssh mtl0 'sudo k3s ctr images import -'
# then set `image: pixeltree-demo:$TAG` in deployment.yaml and push; ArgoCD rolls it out.
```

Cross-building is used because the node has no running Docker daemon; it takes a while under qemu
(torch and the OCR warm-up), the result is the same arm64 image.

## Limits

| what | value | where |
|---|---|---|
| request body | 8 MiB | Traefik `body-limit` middleware (413) and the app |
| formats | PNG / JPEG / WebP, decided by decoding | app (415) |
| pixels | long side <= 4096, <= 12 MP; > 1920 px long side is scaled down for inference | app (413) |
| concurrency | 1 inference at a time, queue of 1 (`PIXELTREE_DEMO_QUEUE_MAX`), then 429 | app |
| timeout | 30 s including the queue (504) | app |
| per-IP rate | whole site: 60/min, burst 40 (Traefik `ratelimit`, keyed on `CF-Connecting-IP`); `/api/parse`: 6/min, burst 3 (app, same key) | both |
| headers | CSP `default-src 'self'` + `frame-ancestors 'none'`, `nosniff`, `X-Frame-Options: DENY` | app |
| logs | time, method, path, status, duration, image size; never image bytes, tree text, IPs | app |

`CF-Connecting-IP` is only trustworthy for requests that come through Cloudflare; a client that reaches the
origin directly can forge it. The single inference slot and the short queue still bound the load.

## Resources and speed

Current values (deliberately conservative: mtl0 has 4 cores and runs other live apps):
requests `cpu: 500m`, `memory: 2600Mi`; limits `cpu: "2"`, `memory: 3584Mi`; `PIXELTREE_THREADS=2`,
`PIXELTREE_DEMO_QUEUE_MAX=1`. One parse takes ~10.6 s.

To make it faster: set the cpu limit to `"3"`, `PIXELTREE_THREADS` to `"3"` (~8.1 s per parse) and
`PIXELTREE_DEMO_QUEUE_MAX` to `"2"` (1 running + 2 queued still finish inside the 30 s timeout). Keep
`PIXELTREE_THREADS` equal to the cpu limit, and keep (queue + 1) x parse time under 30 s.

## Take it offline (rollback)

1. Remove `- pixeltree-app.yaml` from `apps/kustomization.yaml`, delete `apps/pixeltree-app.yaml` and
   `apps/pixeltree/`, push to `master`.
2. The root app prunes the `app-pixeltree` Application. Like the other apps here it has no
   resources finalizer, so its objects stay; delete them with
   `ssh mtl0 'sudo k3s kubectl delete namespace pixeltree'`.
3. Optionally free the image: `ssh mtl0 'sudo k3s ctr images rm docker.io/library/pixeltree-demo:<tag>'`.

Faster emergency stop without a commit (ArgoCD self-heal will undo it on its next sync, so follow with step 1):
`ssh mtl0 'sudo k3s kubectl -n pixeltree scale deploy/pixeltree --replicas=0'`.
