# syntax=docker/dockerfile:1
#
# Build with the cats-toilet project root as the context (this directory also
# holds Dockerfile.dockerignore, which BuildKit picks up next to the Dockerfile):
#
#   docker buildx build --platform linux/arm64 --load \
#     -f /path/to/k3s-homelab/apps/cats-toilet/Dockerfile \
#     -t cats-toilet:arm64 /path/to/cats-toilet
#
# The app has no build step and no npm dependencies: one Node process serves
# the static game and the JSON API (node:http + node:sqlite). There is no RUN
# step, so a cross-build needs no emulation.

FROM node:24-bookworm-slim

ENV NODE_ENV=production
WORKDIR /app

COPY index.html manifest.webmanifest sw.js ./
COPY css ./css
COPY js ./js
COPY assets ./assets
COPY server/server.mjs server/app.mjs server/db.mjs server/static.mjs ./server/

VOLUME ["/data"]
EXPOSE 8080
STOPSIGNAL SIGTERM

# the image's built-in unprivileged user
USER 1000:1000
ENTRYPOINT ["node", "server/server.mjs"]
CMD ["--host", "0.0.0.0", "--port", "8080", "--db", "/data/cats.db"]
