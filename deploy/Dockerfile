FROM caddy:2.8.4-alpine AS mattermost-caddy
COPY caddy/Caddyfile /etc/caddy/Caddyfile
EXPOSE 80 443 443/udp
STOPSIGNAL SIGTERM