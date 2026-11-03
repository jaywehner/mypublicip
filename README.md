# mypublicip

A minimal "what's my IP" service, like ifconfig.me. It returns only the caller's IP address as plain text.

## Run with Docker Compose

```sh
docker compose up -d --build
```

This builds the image and starts it on port `8080`. To use a different port, change the `ports` line in `docker-compose.yml`, e.g. `"80:80"`.

Other commands:

```sh
docker compose logs -f              # view logs
docker compose down                 # stop and remove
docker compose up -d --build        # rebuild after changing nginx.conf
```

## Run with Docker

```sh
docker build -t mypublicip .
docker run -d --name mypublicip -p 8080:80 mypublicip
```

## Use

```sh
curl http://localhost:8080
```

Example response:

```
203.0.113.42
```

## Files

- `docker-compose.yml`: builds and runs the container on port 8080.
- `Dockerfile`: builds on `nginx:alpine`.
- `nginx.conf`: answers every path with `$remote_addr`.

## Running behind SWAG

Behind a reverse proxy the response is the proxy's IP. These steps set it up behind [SWAG](https://docs.linuxserver.io/general/swag) so it returns the real client IP.

### 1. Trust SWAG in `nginx.conf`

Add the `real_ip` lines, then rebuild the image:

```nginx
server {
    listen 80;
    default_type text/plain;

    set_real_ip_from 172.16.0.0/12;
    real_ip_header X-Real-IP;

    location / {
        return 200 "$remote_addr\n";
    }
}
```

`172.16.0.0/12` covers Docker's default bridge networks. If your SWAG network uses another subnet, change it to match. You can look it up with `docker network inspect <network>`. SWAG's `proxy.conf` already sends `X-Real-IP`. If SWAG is set up for Cloudflare real IPs, that header carries the visitor's IP, not Cloudflare's.

### 2. Run on the same network as SWAG

Edit `docker-compose.yml`: remove the `ports` section and join SWAG's network. SWAG reaches the container by its name.

```yaml
services:
  mypublicip:
    build: .
    image: mypublicip
    container_name: mypublicip
    restart: unless-stopped
    networks:
      - swag

networks:
  swag:
    external: true   # use your SWAG network's name
```

Then run `docker compose up -d --build`.

### 3. Add the SWAG proxy conf

Create `/config/nginx/proxy-confs/mypublicip.subdomain.conf` in SWAG:

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name mypublicip.*;

    include /config/nginx/ssl.conf;

    location / {
        include /config/nginx/proxy.conf;
        include /config/nginx/resolver.conf;
        set $upstream_app mypublicip;
        set $upstream_port 80;
        set $upstream_proto http;
        proxy_pass $upstream_proto://$upstream_app:$upstream_port;
    }
}
```

Add `mypublicip` to SWAG's `SUBDOMAINS` (or use a wildcard cert), then run `docker restart swag`.
