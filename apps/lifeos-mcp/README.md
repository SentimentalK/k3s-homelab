# LifeOS MCP deployment

This directory is intentionally dormant until its external deployment values exist. `../lifeos-mcp-app.yaml` is not listed in `apps/kustomization.yaml`, so ArgoCD will not apply these resources yet.

## Activate

1. On the selected ARM64 K3s node, create the persistent workspace and grant it to UID/GID 10001:

   ```bash
   sudo install -d -o 10001 -g 10001 -m 0700 \
     /home/ubuntu/codes/lifeos-mcp/repo \
     /home/ubuntu/codes/lifeos-mcp/txns \
     /home/ubuntu/codes/lifeos-mcp/state
   ```

2. Label that exact node:

   ```bash
   kubectl label node <node-name> lifeos.sentimentalk.com/workspace=true
   ```

3. Create a read/write deploy key scoped only to `SentimentalK/LifeOS`. Use `secret.example.yaml` as input to `kubeseal`; commit the resulting `lifeos-git` SealedSecret and add it to this directory's `kustomization.yaml`.

4. Create a Secure MCP Tunnel and a runtime API key with Tunnels Read + Use. Seal `tunnel-id` and `api-key` into the `lifeos-tunnel` SealedSecret and add it to `kustomization.yaml`.

5. Wait for the LifeOS `MCP image` workflow, replace the placeholder MCP image with the emitted immutable digest, and optionally replace the tunnel-client version tag with its multi-arch digest.

6. Render locally before activation:

   ```bash
   kubectl kustomize apps/lifeos-mcp
   ```

7. Add `lifeos-mcp-app.yaml` to the root `apps/kustomization.yaml` and push. ArgoCD will then create the namespace and the single-replica `Recreate` Deployment.

The MCP server listens only on Pod loopback. There is no Service or Ingress. The tunnel sidecar is the sole remote path and receives no Git credential; the MCP container receives no OpenAI runtime key.
