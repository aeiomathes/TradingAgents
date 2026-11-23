# Shared host notes (read before changing the server or its deploy files)

The Linode behind focusario.com no longer runs only TradingAgents. It also
serves **https://nventorium.com** (a separate personal inventory app). Both
apps sit behind the **one Caddy that lives in this stack**, because only one
process can own ports 80/443. If you remove or bypass that arrangement,
nventorium.com goes down.

## 1. How the sharing works

Two small edits to this repo's deploy files make it possible:

1. `server/deploy/Caddyfile` ends with `import /etc/caddy/sites/*.caddy`
   (an empty glob is valid, so the stack starts fine with the folder empty).
2. `server/docker-compose.prod.yml` mounts `/opt/caddy-sites` from the host
   into the `caddy` container at `/etc/caddy/sites` (read-only).

Each other app drops one `*.caddy` file into `/opt/caddy-sites` on the host.
Nventorium's file holds its site block: `nventorium.com` sends `/api/*` to its
backend container (prefix stripped), everything else to its web container,
and `www` redirects. Its containers join this stack's default network
(`tradingagents_default`, declared `external` in their compose file) so Caddy
can reach them by name. focusario.com, its Basic Auth and the certificate
volume are unaffected.

Create the snippets folder explicitly on a fresh host (Docker would create it
on its own, but as root-owned and without anyone having decided that):

```bash
sudo mkdir -p /opt/caddy-sites
```

## 2. Things that must NOT be undone

- **Do not redeploy from a version of these two files that lacks the two edits
  above.** The deploy command (`git pull`, then `docker compose pull && up -d`)
  recreates the Caddy container from whatever the compose file says. Without
  the mount, nventorium.com silently disappears.
- **If the server's checkout was edited by hand** (this host's was, before the
  edits landed in this branch), reconcile once before pulling:
  `git checkout -- server/deploy/Caddyfile server/docker-compose.prod.yml`,
  then `git pull`. Git refuses to pull over local modifications otherwise.
- **Keep the compose project name `tradingagents`.** Other stacks join this
  stack's network by its generated name, `tradingagents_default`. Renaming the
  project or the network disconnects them.
- **Keep the Caddy container name `tradingagents-caddy-1`.** Other runbooks
  reload Caddy with `docker exec tradingagents-caddy-1 caddy reload ...`.
- **Never `docker compose down -v` here.** The `caddy-data` volume holds the
  Let's Encrypt certificates for **every** site on the host. A plain `down`
  fails safely on the network while another stack is attached; `-v` does not.
- **Don't edit or delete other apps' files in `/opt/caddy-sites/`.**
- Ports 80/443 stay with this Caddy. Anything new goes behind it as another
  `*.caddy` snippet, not as a second web server.

## 3. Host facts

- Linode **Nanode: 1 GB RAM, 1 core, 25 GB disk**. Debian 12 (bookworm),
  kernel 6.1, cgroup v2, Docker 29.x with Compose 5.x (`docker compose` syntax
  is unchanged from earlier versions).
- The Docker "kernel does not support swap limit capabilities" warning that
  older notes describe no longer appears on this host; the `swapaccount=1`
  boot-parameter advice is obsolete.

## 4. Resources are shared and tight

1 GB total. The other stack's three containers use roughly 80 MB at idle and
are individually capped. The `3500M` limit on `api` in
`docker-compose.prod.yml` is larger than the machine's RAM, so it cannot
protect anything: a heavy analysis run, or several at once, can push the host
into OOM-killing, and the victim may be the other app's database. The server
does not cap concurrent analysis runs. Measure with
`sudo docker stats --no-stream` during a real run before raising concurrency,
and consider lowering that limit to something that fits.

## 5. Quick health check

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Status}}'
curl -sI https://focusario.com | head -1      # expect 401 (Basic Auth prompt)
curl -sI https://nventorium.com | head -1     # expect 200
```
