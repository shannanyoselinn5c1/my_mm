FROM nginx:1.27.3-bookworm AS mattermost-nginx
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*
COPY nginx/nginx.conf /etc/nginx/nginx.conf
EXPOSE 8064
STOPSIGNAL SIGQUIT
