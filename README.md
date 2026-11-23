# local-npm

Nginx Proxy Manager - the fleet's reverse proxy, on `ore-dock-110-1`.

## Why this repo exists

This container used to be a bare `docker run` with no compose file and no
repository. Nothing in the fleet's update machinery could see it:

- `discover_version_drift.py` resolves images from Portainer stack records
  and compose files on disk. A bare container is in neither.
- `fleet_float_autoupdate.py` advances floating tags by pushing to a git
  repo. With no repo there was nothing to push to.
- The 15-minute Portainer poll deploys committed repository state. It never
  consults a registry.

So `local-npm` sat on **v2.16.0** while upstream shipped **v2.17.0**
(Nov 12) and **v2.17.1** (Nov 19), and NPM's own UI badge showed
"Update Available: v2.17.1" indefinitely. Bringing it under git means
the drift audit and the floating-tag updater both see it from now on.

## Why the image is digest-pinned

A floating `:latest` can never advance on its own: the poll redeploys the
tag exactly as written and does not re-pull it. The pin names the
artifact, and `fleet_float_autoupdate.py` advances the pin when upstream
moves.

## The Route53 plugin - read this before changing the entrypoint

Nginx Proxy Manager does **not** bundle any certbot DNS plugins in its
image. On a fresh `2.17.1` container:

```
$ docker run --rm --entrypoint sh jc21/nginx-proxy-manager:2.17.1 \
    -c 'certbot plugins 2>&1 | grep -c "dns-"'
0
```

The plugin is installed on demand by `/app/scripts/install-certbot-plugins`,
which pip-installs `certbot-dns-route53` at the version matching the
bundled certbot. In `/app/certbot/dns-plugins.json` the route53 entry is
templated:

```json
"route53": {
  "package_name": "certbot-dns-route53",
  "version": "=={{certbot-version}}",
  "dependencies": "acme=={{certbot-version}}"
}
```

**The renewal path does not install the plugin.** Only the *initial
issuance* path calls `installPlugin()` (`certificate.js:1097`). Renewal
runs `certbot renew`, which loads the authenticator named in
`/etc/letsencrypt/renewal/npm-<id>.conf`:

```
authenticator = dns-route53
```

and that plugin has to already be importable.

The container that ran before this migration had the plugin only because
somebody pip-installed it into the container layer on 2026-10-29 - it was
not part of the image. Recreating the container from the image alone would
have dropped it, and **all 105 Let's Encrypt certificates** - every one
of them using `dns_challenge` with `dns_provider=route53` - would have
failed to renew. 31 of them expire within 60 days.

So the compose entrypoint installs the plugin on every boot, using NPM's
own installer. Two details matter:

1. **`/usr/bin/common.sh` must be sourced first.** It derives and exports
   `CERTBOT_VERSION` from the installed certbot. Without it the installer
   tries to install `certbot-dns-route53==undefined` and fails.
2. It must run **before** `exec /init`, because `certbot renew` runs
   inside the container and needs the plugin on disk.

The install is idempotent and cheap when already present. If it fails,
NPM still starts and serves traffic - only renewal is affected - so the
failure is logged loudly rather than blocking startup.

## State

State lives in two existing named volumes, declared `external: true`:

| volume | mount | holds |
|---|---|---|
| `local-npm-data` | `/data` | sqlite database, generated nginx config, `keys.json` |
| `local-npm-letsencrypt` | `/etc/letsencrypt` | certificates and renewal config |

`keys.json` holds the JWT signing key. Back it up before any change.

## Deploy

Git-backed Portainer stack, standalone (Type=2) on endpoint 3 (`local`).
Poll interval 15m.
