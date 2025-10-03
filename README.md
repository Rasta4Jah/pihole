# Pi‑hole Docker Compose Stack (.env‑driven)

This repository runs **Pi‑hole** in Docker with a simple, portable `docker-compose.yml` that reads settings from a `.env` file. It’s designed to be friendly for Synology NAS _and_ general Linux hosts.

> Default values are baked in so it works out of the box, but you can override anything in `.env`.

---

## Files
- `docker-compose.yml` — the stack definition (variables come from `.env`)
- `.env.example` — copy to `.env` and edit your values
- `.gitignore` — keeps `.env` out of version control
- `README.md` — you’re reading it

---

## Quick start
```bash
# 1) (Optional) copy the example template and edit values
cp .env.example .env
# 2) Launch
docker compose pull
docker compose up -d
# Optionally specify the env file:
# docker compose --env-file ./.env up -d
```

> Networking: this stack uses `network_mode: host`. Pi‑hole’s DNS will bind the host’s port **53**, and the Web UI will use port **8080** by default (see `FTLCONF_webserver_port`). If you already have services listening on those ports, change the settings or switch to a user‑defined bridge network.

---

## Environment variables

| Variable | Default | Required? | What it does |
|---|---|---|---|
| `PIHOLE_API_PASSWORD` | `Password!` | Recommended to change | Sets the Pi‑hole web UI password. **Change this before exposing the UI.** |
| `TZ` | `America/New_York` | Optional | Container timezone. |
| `DNSMASQ_USER` | `pihole` | Optional | Linux user inside the container that `dnsmasq` runs as. Leave as default unless you know you need to change it. |
| `PIHOLE_UID` | `1026` | Optional | Numeric **user ID on the host** that should own the bind‑mounted Pi‑hole data. |
| `PIHOLE_GID` | `100` | Optional | Numeric **group ID on the host** that should own the bind‑mounted Pi‑hole data. |

These map into the compose file like so:

```yaml
environment:
  FTLCONF_webserver_api_password: ${PIHOLE_API_PASSWORD:-Password!}
  FTLCONF_webserver_port: 8080
  FTLCONF_dns_listeningMode: all
  TZ: ${TZ:-America/New_York}
  DNSMASQ_USER: ${DNSMASQ_USER:-pihole}
  PIHOLE_UID: ${PIHOLE_UID:-1026}
  PIHOLE_GID: ${PIHOLE_GID:-100}
```

> Note: Not all upstream images honor `PIHOLE_UID`/`PIHOLE_GID`. If the image ignores them, you can still avoid permission issues by setting ownership on the **host** directories (see below).

---

## Host data directories

By default, the compose file mounts these host paths (Synology‑style) into the container:

```yaml
volumes:
  - /volume1/docker/pihole/dnsmasq.d:/etc/dnsmasq.d:rw
  - /volume1/docker/pihole/pihole:/etc/pihole:rw
```

- On **Synology DSM**, these paths are fine.  
- On a **generic Linux host**, you might prefer something like `/srv/pihole/{dnsmasq.d,pihole}` or `/opt/pihole/...`. If you change the host paths, update the two `volumes:` lines accordingly.

Create and set ownership on your chosen host directories to match the UID/GID you will use:

```bash
# Example for a generic Linux host (adjust the base path as desired)
sudo mkdir -p /srv/pihole/dnsmasq.d /srv/pihole/pihole
sudo chown -R <UID>:<GID> /srv/pihole
```

---

## Choosing the right `PIHOLE_UID` and `PIHOLE_GID`

The goal is simple: the **host** user that owns your bind‑mounted directories should match the UID/GID used by the container for writing. This prevents “permission denied” issues and keeps files manageable from the host.

### How to find your UID/GID on different platforms

**Synology DSM**  
Use your DSM login user (or a service user you created). Find its numbers:
```bash
id <your-dsm-username>
# Example output: uid=1026(username) gid=100(users) ...
```
- Put those numbers into `.env` as `PIHOLE_UID=1026` and `PIHOLE_GID=100`.
- Ensure your bind‑mount directories are owned by that UID/GID (see chown example above).

**Generic Linux (Ubuntu, Debian, etc.)**  
Pick a host user (often your normal account or a dedicated `pihole` user):
```bash
id <host-username>
# Example: uid=1001(user) gid=1001(user) ...
```
- Set `PIHOLE_UID=1001`, `PIHOLE_GID=1001` to match.
- `chown` the bind‑mount folders to `1001:1001`.

**Unraid**  
Many stacks map to the default `nobody:users` which is typically `uid=99`, `gid=100`. You can either:
- Use `PIHOLE_UID=99` and `PIHOLE_GID=100`, and `chown -R 99:100` your data folders, **or**
- Create a dedicated user and use its UID/GID instead.

**TrueNAS SCALE**  
Create/select a user for the application data set and look up its IDs in **Accounts → Users** or via shell:
```bash
id <truenas-user>
```
Use those values for `PIHOLE_UID`/`PIHOLE_GID` and ensure dataset permissions are aligned.

**Docker Desktop (macOS/Windows)**  
Your bind mounts are presented from a Linux VM. The VM handles permissions; in most cases you **don’t need** to set `PIHOLE_UID`/`PIHOLE_GID`. If you do encounter “permission denied,” try switching the mount path to a shared folder managed by Docker Desktop, or use a Linux host path via WSL2 (Windows).

---

## Security and networking notes

- **Change the default password.** `Password!` is for demos. Set a strong `PIHOLE_API_PASSWORD` before exposing the UI beyond your LAN.  
- **Host networking:** With `network_mode: host`, port mappings in `ports:` are ignored. Pi‑hole will listen directly on the host’s ports (DNS on 53/UDP+TCP and web on 8080/TCP by default).  
- **Port conflicts:** If something already uses port 53 or 8080, either stop/relocate that service or change Pi‑hole’s web port (`FTLCONF_webserver_port`) and/or switch to a user‑defined bridge network and add explicit `ports:`.  
- **no-new-privileges:** The compose sets `security_opt: [ "no-new-privileges:false" ]`. For extra hardening, change to `true` unless you have a known need for additional privileges.

---

## Troubleshooting

- **Permission denied writing to `/etc/pihole` or `/etc/dnsmasq.d`:** Verify the host directories exist and are owned by the UID/GID you configured (or adjust `PIHOLE_UID`/`PIHOLE_GID` accordingly).  
- **Web UI doesn’t load:** Check for port conflicts on 8080 or change the port in the compose env var.  
- **DNS not responding:** Confirm nothing else is bound to port 53 on the host; on Synology ensure built‑in DNS Server or other services aren’t occupying port 53.

---

## Editing `.env`

Copy the template and edit:
```bash
cp .env.example .env
# then edit .env
```

Example `.env` values (safe defaults shown—remember to change the password):
```dotenv
PIHOLE_API_PASSWORD=Password!
TZ=America/New_York
DNSMASQ_USER=pihole
PIHOLE_UID=1026      # Synology example; replace with your host UID
PIHOLE_GID=100       # Synology example; replace with your host GID
```

That’s it. Tweak the bind‑mount paths and UID/GID for your platform, and you’re off to the races.


---

## Alternative compose for generic Linux (bridge networking)

If you don’t want host networking, use `docker-compose.bridge.yml` and standard bind mounts under `/srv/pihole/...`:

```yaml
# docker-compose.bridge.yml (excerpt)
services:
  pihole:
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8080:8080/tcp"
    networks:
      - pihole_net
networks:
  pihole_net:
    driver: bridge
```

Launch with:
```bash
sudo mkdir -p /srv/pihole/dnsmasq.d /srv/pihole/pihole
sudo chown -R ${PIHOLE_UID:-1026}:${PIHOLE_GID:-100} /srv/pihole

docker compose -f docker-compose.bridge.yml pull
docker compose -f docker-compose.bridge.yml up -d
```

> DHCP server note: Pi‑hole’s DHCP requires `network_mode: host` to broadcast properly. On bridge, keep DHCP **disabled** and continue using your router’s DHCP.
