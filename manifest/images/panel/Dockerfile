FROM alpine:3.18 as base

RUN \
    apk --update add curl gettext mariadb-client nginx php82 php82 php82-bcmath php82-common php82-dom php82-fileinfo \
    php82-fpm php82-gd php82-mbstring php82-openssl php82-pdo php82-phar php82-json php82-sodium \
    php82-pdo_mysql php82-posix php82-session php82-simplexml php82-tokenizer php82-ctype php82-zlib php82-zip php82-xmlwriter \
    tini \
    && mkdir -p /var/www/html /run/nginx /etc/nginx/conf.d/

# Build phase of the container
# This is where composer is added and pterodactyl properly setup
FROM base as build

ARG VERSION
ARG GIT_BRANCH=release/${VERSION}

ENV VERSION=${VERSION}

WORKDIR /var/www/html

RUN \
    apk add yarn git;
  
RUN \
    git clone https://github.com/pterodactyl/panel ./ --depth 1 --branch ${GIT_BRANCH} \
    && rm .git -rf \
    && chmod -R 755 storage/* bootstrap/cache \
    && find storage -type d > .storage.tmpl \
    && curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer \
    && cp .env.example .env \
    && composer install --ansi --no-dev --optimize-autoloader \
    && chown -R nginx:nginx *;

RUN \
    yarn install --production \
    && yarn add cross-env \
    && yarn run build:production \
    && rm -rf node_modules

# Remove persistent data folders as they will be symlinked further down
RUN rm .env ./storage -rf

# Final Production phase of the controller
# All build requirements get scrapped as to maintain a small image
FROM base as production

ARG VERSION

ENV VERSION=${VERSION}

WORKDIR /var/www/html

COPY --from=build --chown=nginx:nginx /var/www /var/www
COPY ./root/ /

# Symlink storage, config and cache to /data
RUN \
    ln -s /data/storage storage; \
    ln -s /data/pterodactyl.conf .env;

EXPOSE 80 443
VOLUME [ "/data" ]

RUN dos2unix /entrypoint.sh
RUN dos2unix /usr/local/bin/wait-for
RUN dos2unix /etc/crontabs/nginx

ENTRYPOINT [ "/sbin/tini", "--", "/entrypoint.sh" ]
