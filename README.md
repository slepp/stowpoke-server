# Stowpoke server

Run your own [Stowpoke](https://stowpoke.com): the inventory engine and web app behind one access token, in one SQLite
file on your machine. No account, no subscription, no one else's cloud. **v0.1 beta.**

This repository is the home of the published image: the user guide, `compose.yaml` and `kubernetes.yaml`. The
server's source is not published yet.

## What you need

- Docker, on `linux/amd64` or `linux/arm64` (a NAS, a small server, a Raspberry Pi 4/5 with a 64-bit OS).
- A folder or volume to keep the inventory in. Everything lives in `/data`.
- A token: a long secret you choose. It is the password to the inventory, shared by every phone and browser you
  trust.

## Start

Make a token and keep it somewhere safe:

```sh
openssl rand -base64 32
```

Then start the server, pasting the token:

```sh
docker run -d --name stowpoke --restart unless-stopped \
  -p 8787:8787 -v stowpoke:/data \
  -e API_TOKEN=YOUR_TOKEN \
  ghcr.io/slepp/stowpoke-server:latest
```

Check it:

```sh
curl http://localhost:8787/api/health
# {"status":"ok","protocol":1,"accounts":false,"apple_web_client_id":null}
```

`accounts` is `false` on your own server: there is no sign-in, only the token.

## Connect

- **Web:** open `http://your-address:8787` and paste the token. The browser keeps it.
- **iPhone:** Settings → **Your own server** → enter the same address and token.
- **T-Embed reader:** set it up from the app (Settings → Your devices → Set up a reader), or give it the address and
  token in `config.h`.

On your home network use the machine's address, for example `192.168.1.20:8787`. Numbers and names ending in
`.local` use plain HTTP automatically; for any other name without HTTPS, include the scheme
(`http://nas.example:8787`).

## Configuration

| Setting | Default | Required | What |
| --- | --- | --- | --- |
| `API_TOKEN` | — | yes | The shared secret. At least 16 characters; generate 32 random bytes. Changing it signs every device out. |
| `PORT` | `8787` | no | Port inside the container. |
| `DATA_DIR` | `/data` | no | Where `stowpoke.db` lives. Mount a volume here. |
| `WEB_DIR` | `/app/web` | no | The built web app. Leave it alone in the published image. |

The container runs as the unprivileged `bun` user (uid 1000). A named volume such as `stowpoke` is easiest. If you
bind-mount a host folder instead, it must be writable by uid 1000:

```sh
mkdir -p /srv/stowpoke && sudo chown 1000:1000 /srv/stowpoke
```

## Docker Compose

```sh
printf 'STOWPOKE_TOKEN=%s\n' "$(openssl rand -base64 32)" > .env
chmod 600 .env
docker compose up -d
```

The included [compose.yaml](compose.yaml) uses `STOWPOKE_TOKEN` from `.env`, the named volume `stowpoke`, and
`restart: unless-stopped`.

## Kubernetes

```sh
kubectl create secret generic stowpoke --from-literal=token="$(openssl rand -base64 32)"
kubectl apply -f kubernetes.yaml
```

[kubernetes.yaml](kubernetes.yaml) creates a single-replica Deployment (`Recreate`, one copy per volume), a 1 Gi
PersistentVolumeClaim and a Service on port 80 → 8787. Expose it over HTTPS with your usual Ingress or Gateway.
Set the token in the Secret first; the manifest reads `stowpoke`/`token`. The pod runs with `fsGroup: 1000` so the
`/data` volume is writable.

## Reach it away from home

Keep it on the LAN if that is enough. To use it from anywhere, put HTTPS in front and do not expose port 8787
directly. With Caddy:

```sh
caddy reverse-proxy --from stowpoke.example.com --to localhost:8787
```

A Cloudflare Tunnel, Tailscale or a Kubernetes Ingress works too. Your proxy must forward WebSocket upgrades for
`/api/live`; otherwise the web app still loads, but changes from other devices do not arrive live. Caddy does this
automatically. For nginx, set `proxy_http_version 1.1` and pass the `Upgrade`/`Connection` headers.

Use an `https://` address in the app when the server is not on your LAN. A plain `http://` address only makes sense
inside your own network.

## Operate it

### Logs and health

```sh
docker logs -f stowpoke
curl -fsS http://localhost:8787/api/health
```

The health endpoint answers `200` when the server is up. `/api/catalog` and every other API route need
`Authorization: Bearer YOUR_TOKEN`; a `401` without it is expected.

### Back up

The easy way is **Settings → Export** in the web app or the iPhone app: one JSON file with places, items and tags.
It is never gated.

For a file-level backup, stop the container first. SQLite keeps a write-ahead log, so copying `/data` while the
server runs can produce an inconsistent copy.

```sh
docker stop stowpoke
docker cp stowpoke:/data ./stowpoke-data-backup
docker start stowpoke
```

If you use the named volume without a container, run a temporary one:

```sh
docker run --rm -v stowpoke:/data -v "$PWD":/backup alpine \
  tar czf /backup/stowpoke-data.tgz -C /data .
```

Keep the backup off the machine. The token is not in the database; keep it with your other secrets.

### Restore

- **From an export:** use **Import** in the web app or the app. Import loads into an **empty** inventory and refuses
  otherwise (`409`), so test it before you need it.
- **From a file copy:** stop the container, replace the contents of the volume, and start it again. Restore into the
  same image version that made the backup if you can.

### Update

```sh
docker pull ghcr.io/slepp/stowpoke-server:latest
docker stop stowpoke
docker rm stowpoke
# run the same docker run command again
```

With Compose:

```sh
docker compose pull
docker compose up -d
```

The volume is untouched. Every build is tagged `sha-<short>`; releases also get a version tag. Pin one of those
instead of `latest` if you want to update deliberately. To roll back, start the previous tag with the same volume.
Back up before a major update.

### Change the token

Change `API_TOKEN` and restart the container. Every phone and browser must enter the new token; the old one stops
working immediately. There is no per-device token and no way to revoke one device while leaving the others.

### One copy per volume

Run a single container against a volume. Two copies sharing `stowpoke.db` will fight over SQLite locks. For
Kubernetes, keep `replicas: 1` and the `Recreate` strategy.

### Disk and resources

The image is small and the server is idle when nothing is happening. The database grows with items, tags and the
movement history; a household inventory is normally well under a few hundred megabytes. `/data` also holds the
SQLite write-ahead log (`stowpoke.db-wal`) and shared-memory file (`stowpoke.db-shm`) while the server runs; all
three belong together.

### API

Every route under `/api` except `/api/health` needs the bearer token. The catalog, items, places, movements,
reader events, export and import used by the apps are there. `/api/live` is a WebSocket that announces changes;
it authenticates with the `stowpoke` subprotocol plus `bearer.<base64url token>`. If you write your own client,
read the app's network traffic or the server source; there is no separate API reference yet.

## Troubleshooting

| What you see | What to do |
| --- | --- |
| Container exits at once with "Set API_TOKEN…" | `API_TOKEN` is missing or shorter than 16 characters. |
| `401` from the web app or phone | The token does not match. Watch for a trailing newline or a token left over from before a change. |
| Phone cannot connect, browser can | Use the machine's LAN address, not `localhost`; check the phone is on the same network and the port is open. |
| Web page loads but data does not | The reverse proxy is not forwarding `/api` (and WebSocket upgrades for `/api/live`). |
| `database is locked` | A second container is using the same volume. Stop it. |
| `permission denied` on `/data` | The bind mount is not writable by uid 1000: `sudo chown -R 1000:1000 /srv/stowpoke`. |
| Image will not pull | The package must be public; if it is not yet, authenticate with `docker login ghcr.io`. |

## What is not in the beta

Separate people and households, Sign in with Apple, and linking a screen by QR code belong to the hosted
Stowpoke service. On your own server one token is shared by every device, and anyone with it can read and change
everything.

The T-Embed reader **is** supported: set it up from the app or with the address and token in `config.h`.

## Help

- Guide, with the same steps and a FAQ: <https://stowpoke.com/self-host>
- Email: <support@stowpoke.com>
- This repository: guide, [compose.yaml](compose.yaml) and [kubernetes.yaml](kubernetes.yaml). The image is built
  from the private Stowpoke product repository; its source is not published yet.

## Security

The server sends nothing to Stowpoke: no telemetry, no update check, no error reports. When a request fails
unexpectedly it writes one line to its own log (a code, the operation and a random reference, also shown in the
error message); nothing else about the request leaves the machine.

Treat `API_TOKEN` like a password, use HTTPS outside your LAN, and keep the image updated. The server has no
built-in user accounts, rate limiting or audit log; if you need those, put it behind your own proxy.
