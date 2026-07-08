# SearXNG stack for Portainer

Create a new [SearXNG](https://github.com/searxng/searxng) instance in five minutes using [Portainer](https://www.portainer.io/).

> **Note:** This stack was migrated from the now-deprecated `searx` + `filtron` + `morty`
> setup to the maintained [SearXNG](https://github.com/searxng/searxng) engine.
> Bot protection is now handled by SearXNG's built-in limiter (backed by Valkey),
> and image results are proxied by SearXNG itself, so `filtron` and `morty` are no
> longer needed.

## What is included ?

| Name | Description | Docker image |
| -- | -- | -- |
| [Nginx](https://www.nginx.com/) | Reverse proxy HTTP server with SSL termination and static asset caching | [nginx:1.27-alpine](https://hub.docker.com/_/nginx) |
| [SearXNG](https://github.com/searxng/searxng) | Privacy-respecting metasearch engine | [searxng/searxng:latest](https://hub.docker.com/r/searxng/searxng) |
| [Valkey](https://valkey.io/) | In-memory data store backing SearXNG's rate limiter / bot protection | [valkey/valkey:8-alpine](https://hub.docker.com/r/valkey/valkey) |

## How to use it
- [Install docker](https://docs.docker.com/install/)
- [Install docker compose](https://docs.docker.com/compose/install/) (v2 or newer).
- [Install Portainer](https://docs.portainer.io/start/install)
- Use the stack GUI to spawn your SearXNG instance, providing the environment
  variables below.

## Environment variables
```
# Repository containing the nginx templates and other configurations.
# To deploy a specific branch, prepend the git flags, e.g.:
# CONFIG_REPO_URL=--branch my-branch https://github.com/oleduc/searx-docker-portainer.git
CONFIG_REPO_URL=https://github.com/oleduc/searx-docker-portainer.git

# Example search.example.com or localhost (default)
SEARXNG_HOSTNAME=<hostname>

# SearXNG secret key. Generate with: openssl rand -hex 32
SEARXNG_SECRET=<secret>

# Enable the built-in bot / rate limiter (requires the valkey service). true|false
SEARXNG_LIMITER=true
# Proxy image results through SearXNG for privacy. true|false
SEARXNG_IMAGE_PROXY=true
# Search submit method. POST keeps queries out of the URL/history (recommended).
# Set to GET to allow ?q= URL searches, the browser search bar and the JSON API.
SEARXNG_METHOD=POST

# NGINX HTTP / HTTPS ports
NGINX_HTTP_PORT=5080
NGINX_HTTPS_PORT=5443
# Path to your ssl certificates
SSL_CERT_PATH=<path to your fullchain certificate>
SSL_KEY_PATH=<path to your private key>

# Nginx static asset cache configuration
STATIC_CACHE_INDEX_SIZE=64m          # Size of the cached file index
STATIC_SERVER_CACHE_MAX_SIZE=1024m   # Maximum total size of the cached assets
STATIC_CACHE_EXPIRATION=30d          # How long until an asset is considered expired
STATIC_CACHE_INACTIVE=1d             # How long after expiration an asset is deleted
STATIC_CLIENT_CACHE_MAX_AGE=2592000  # How long the client should cache the response

# Nginx search result cache configuration (opt-in)
RESULTS_CACHE_INDEX_SIZE=256m        # Size of the cached search results index
RESULTS_SERVER_CACHE_MAX_SIZE=5120m  # Maximum total size of the cache
RESULTS_CACHE_EXPIRATION=10m         # How long until a search result is considered expired
RESULTS_CACHE_INACTIVE=5m            # How long after expiration a result is deleted
RESULTS_CLIENT_CACHE_MAX_AGE=600     # How long the client should cache the response
```

## Configuration

SearXNG is configured entirely through environment variables (`SEARXNG_*`), so no
config file needs to be edited for a basic deployment. On first start the
container writes a default `settings.yml` into the `searxng-config` volume; the
`SEARXNG_*` variables above override the relevant values. To customise engines or
UI, edit `/etc/searxng/settings.yml` inside the `searxng-config` volume — see the
[SearXNG settings documentation](https://docs.searxng.org/admin/settings/).

## Bot protection

Rate limiting and bot detection are provided by SearXNG's
[limiter](https://docs.searxng.org/admin/searx.limiter.html), enabled with
`SEARXNG_LIMITER=true` and backed by the bundled `valkey` service. The nginx
reverse proxy forwards the real client IP (`X-Forwarded-For` / `X-Real-IP`) so the
limiter sees the correct address.

## Search result formats

By default SearXNG only serves the `html` format and submits searches over `POST`
(so queries stay out of URLs and logs). To use the JSON API or plain `?q=` URLs
you need to:

1. Set `SEARXNG_METHOD=GET` so `?q=` GET requests are accepted.
2. Enable the extra formats in the `searxng-config` volume's `settings.yml`:
   ```yaml
   search:
     formats:
       - html
       - json
   ```

The limiter also guards the instance against non-browser clients, so add the
caller's address to `pass_ip` in `limiter.toml` (or lower the limits) for
unattended API access.
```
curl "https://<hostname>/search?q=test&format=json" | jq
```
