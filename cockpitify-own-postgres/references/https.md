# HTTPS for the ops-runner

`OPS_PUBLIC_URL` must be a public `https://` name that reaches the container's port 8080.

## A domain with Caddy (the server has ports 80 and 443 open)

1. Point a DNS record (A / AAAA, or CNAME) for e.g. `ops.example.com` at the server's public IP.
2. Run Caddy next to the runner; it gets and renews the certificate by itself.

```yaml
services:
  cockpitify-ops:
    image: ghcr.io/cockpitify/ops-runner:0.81.0
    restart: unless-stopped
    environment:
      DATABASE_URL: ${DATABASE_URL}
      OPS_HOST_KEY: ${OPS_HOST_KEY}
      OPS_PUBLIC_URL: https://ops.example.com
      OPS_PAIRING_CODE: K7QM4TXP
  caddy:
    image: caddy:2
    restart: unless-stopped
    ports: ["80:80", "443:443"]
    command: caddy reverse-proxy --from ops.example.com --to cockpitify-ops:8080
    volumes: ["caddy-data:/data"]
volumes:
  caddy-data:
```

The runner no longer needs `ports:` of its own: only Caddy is published.

Already running nginx or another proxy? Add a server block for the name that proxies to
`http://127.0.0.1:8080` and keeps the `Host` header; the certificate comes from wherever that proxy
gets its others.

## Cloudflare Tunnel (no open port, behind a firewall, no certificate to manage)

Needs a domain on Cloudflare.

1. In the Cloudflare dashboard: Zero Trust → Networks → Tunnels → Create a tunnel (Cloudflared).
   Copy its token.
2. Add a public hostname: `ops.example.com` → service `http://cockpitify-ops:8080`.
3. Run the connector beside the runner:

```yaml
services:
  cockpitify-ops:
    image: ghcr.io/cockpitify/ops-runner:0.81.0
    restart: unless-stopped
    environment:
      DATABASE_URL: ${DATABASE_URL}
      OPS_HOST_KEY: ${OPS_HOST_KEY}
      OPS_PUBLIC_URL: https://ops.example.com
      OPS_PAIRING_CODE: K7QM4TXP
  cloudflared:
    image: cloudflare/cloudflared:latest
    restart: unless-stopped
    command: tunnel run
    environment:
      TUNNEL_TOKEN: ${CLOUDFLARE_TUNNEL_TOKEN}
```

Keep `CLOUDFLARE_TUNNEL_TOKEN` in the same secrets file as `OPS_HOST_KEY`.

## Platforms that give HTTPS themselves

Fly.io, Render, Railway, Cloud Run, ECS behind a load balancer: deploy the image there, set the four
settings as secrets, and use the address the platform gives (or your own name on it) as
`OPS_PUBLIC_URL`. The container must be able to reach the database, and only one instance should run
(scale to 1).

## Check

```sh
curl -s https://ops.example.com/healthz
```

An answer like `{"ok":true,"paired":true,…}` from outside the server (not from the server itself)
means Cockpitify can reach it too.
