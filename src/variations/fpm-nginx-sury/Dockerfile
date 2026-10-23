# check=skip=SecretsUsedInArgOrEnv
##########
# Neuromobile: PHP 7.2 FPM + NGINX built from Debian + deb.sury.org
##########
# The official "php" images stopped receiving security fixes for 7.2 years ago.
# This variation keeps the serversideup tooling (s6, nginx, entrypoint scripts)
# but installs PHP from deb.sury.org, which still backports security patches.
# It mimics the official image layout (/usr/local/etc/php, /usr/local/sbin/php-fpm)
# so all scripts under src/ work unchanged.
ARG BASE_OS_VERSION='bookworm'
ARG PHP_VERSION='7.2'
ARG BASE_IMAGE="debian:${BASE_OS_VERSION}-slim"

##########
# S6 Build
##########
FROM ${BASE_IMAGE} AS s6-build

ARG S6_DIR='/opt/s6/'
ARG S6_SRC_URL="https://github.com/just-containers/s6-overlay/releases/download"

# copy our scripts
COPY --chmod=755 src/common/ /
COPY --chmod=755 src/s6/ /

RUN apt-get update && \
    apt-get install -y --no-install-recommends ca-certificates curl xz-utils && \
    docker-php-serversideup-s6-install

####################
# APT repositories (NGINX + sury)
####################
FROM ${BASE_IMAGE} AS repo-config

ARG NGINX_SIGNING_KEY_URL="https://nginx.org/keys/nginx_signing.key"
ARG NGINX_SIGNING_FINGERPRINT="573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62"
ARG SURY_KEYRING_URL="https://packages.sury.org/debsuryorg-archive-keyring.deb"
ARG SURY_SIGNING_FINGERPRINT="15058500A0235D97F5D10063B188E2B695BD4743"

RUN apt-get update && \
    apt-get install -y --no-install-recommends curl gnupg2 ca-certificates && \
    . /etc/os-release && \
    mkdir -p /root/.gnupg && \
    \
    # Import and verify NGINX signing key
    curl -sSLf "$NGINX_SIGNING_KEY_URL" | gpg --dearmor > /usr/share/keyrings/nginx-archive-keyring.gpg && \
    if ! gpg --show-keys --with-colons /usr/share/keyrings/nginx-archive-keyring.gpg | grep -q "$NGINX_SIGNING_FINGERPRINT"; then \
        echo "ERROR: NGINX key did not match signing signature!" && \
        exit 1; \
    fi && \
    echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/debian ${VERSION_CODENAME} nginx" > /etc/apt/sources.list.d/nginx.list && \
    printf "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" > /etc/apt/preferences.d/99nginx && \
    \
    # Import and verify sury signing key
    curl -sSLf -o /tmp/debsuryorg-archive-keyring.deb "$SURY_KEYRING_URL" && \
    dpkg -i /tmp/debsuryorg-archive-keyring.deb && \
    if ! gpg --show-keys --with-colons /usr/share/keyrings/debsuryorg-archive-keyring.gpg | grep -q "$SURY_SIGNING_FINGERPRINT"; then \
        echo "ERROR: sury key did not match signing signature!" && \
        exit 1; \
    fi && \
    echo "deb [signed-by=/usr/share/keyrings/debsuryorg-archive-keyring.gpg] https://packages.sury.org/php/ ${VERSION_CODENAME} main" > /etc/apt/sources.list.d/php.list

##########
# FPM-NGINX-SURY: Main Image
##########
FROM ${BASE_IMAGE}
ARG PHP_VERSION
# Pinned on purpose: bump these to pick up new security patches from sury
ARG PHP_SURY_VERSION='7.2.34-66+0~20261021.122+debian12~1.gbpe0bcd4'
ARG PHP_REDIS_SURY_VERSION='1:6.0.2-3+0~20260711.8+debian12~1.gbpa7c7d9'
ARG DEPENDENCY_PACKAGES_DEBIAN='ca-certificates curl libfcgi-bin gettext-base procps zip'
ARG DEPENDENCY_PHP_PACKAGES='fpm cli common json opcache readline curl mbstring xml mysql pgsql sqlite3 zip intl gd'
ARG REPOSITORY_BUILD_VERSION='dev'
ARG NGINX_VERSION='1.30.1-1~bookworm'

LABEL org.opencontainers.image.title="neuromobile/docker-php7.2 (fpm-nginx-sury)" \
    org.opencontainers.image.description="PHP ${PHP_VERSION} from deb.sury.org on Debian, with the serversideup/php fpm-nginx tooling." \
    org.opencontainers.image.source="https://github.com/serversideup/docker-php" \
    org.opencontainers.image.vendor="Neuromobile" \
    org.opencontainers.image.version="${REPOSITORY_BUILD_VERSION}" \
    org.opencontainers.image.licenses="GPL-3.0-or-later"

ENV APP_BASE_DIR=/var/www/html \
    COMPOSER_ALLOW_SUPERUSER=1 \
    COMPOSER_HOME=/composer \
    COMPOSER_MAX_PARALLEL_HTTP=24 \
    DISABLE_DEFAULT_CONFIG=false \
    LOG_OUTPUT_LEVEL=warn \
    HEALTHCHECK_PATH="/healthcheck" \
    NGINX_ACCESS_LOG="/dev/stdout" \
    NGINX_ERROR_LOG="/dev/stderr" \
    NGINX_FASTCGI_BUFFERS="8 8k" \
    NGINX_FASTCGI_BUFFER_SIZE="8k" \
    NGINX_HTTP_PORT="8080" \
    NGINX_HTTPS_PORT="8443" \
    NGINX_LISTEN_IP_PROTOCOL="all" \
    NGINX_SERVER_TOKENS=off \
    NGINX_WEBROOT=/var/www/html/public \
    NGINX_CLIENT_MAX_BODY_SIZE="100M" \
    PHP_DATE_TIMEZONE="UTC" \
    PHP_DISPLAY_ERRORS=Off \
    PHP_DISPLAY_STARTUP_ERRORS=Off \
    PHP_ERROR_LOG="/dev/stderr" \
    PHP_ERROR_REPORTING="22527" \
    PHP_FPM_PM_CONTROL=ondemand \
    PHP_FPM_PM_MAX_CHILDREN="20" \
    PHP_FPM_PM_MAX_REQUESTS="0" \
    PHP_FPM_PM_MAX_SPARE_SERVERS="3" \
    PHP_FPM_PM_MIN_SPARE_SERVERS="1" \
    PHP_FPM_PM_START_SERVERS="2" \
    PHP_FPM_PM_STATUS_PATH="/status" \
    PHP_FPM_POOL_NAME="www" \
    PHP_FPM_PROCESS_CONTROL_TIMEOUT="10s" \
    PHP_MAX_EXECUTION_TIME="99" \
    PHP_MAX_INPUT_TIME="-1" \
    PHP_MAX_INPUT_VARS="1000" \
    PHP_MEMORY_LIMIT="256M" \
    PHP_OPCACHE_ENABLE="0" \
    PHP_OPCACHE_ENABLE_FILE_OVERRIDE="0" \
    PHP_OPCACHE_FORCE_RESTART_TIMEOUT="180" \
    PHP_OPCACHE_INTERNED_STRINGS_BUFFER="8" \
    PHP_OPCACHE_JIT="off" \
    PHP_OPCACHE_JIT_BUFFER_SIZE="0" \
    PHP_OPCACHE_MAX_ACCELERATED_FILES="10000" \
    PHP_OPCACHE_MEMORY_CONSUMPTION="128" \
    PHP_OPCACHE_REVALIDATE_FREQ="2" \
    PHP_OPCACHE_SAVE_COMMENTS="1" \
    PHP_OPCACHE_VALIDATE_TIMESTAMPS="1" \
    PHP_OPEN_BASEDIR="" \
    PHP_POST_MAX_SIZE="100M" \
    PHP_REALPATH_CACHE_TTL="120" \
    PHP_SESSION_COOKIE_SECURE=false \
    PHP_UPLOAD_MAX_FILE_SIZE="100M" \
    PHP_ZEND_DETECT_UNICODE="" \
    PHP_ZEND_MULTIBYTE="Off" \
    S6_BEHAVIOUR_IF_STAGE2_FAILS=2 \
    S6_CMD_WAIT_FOR_SERVICES_MAXTIME=0 \
    S6_KEEP_ENV=1 \
    S6_VERBOSITY=1 \
    SHOW_WELCOME_MESSAGE=true \
    SSL_MODE=off \
    SSL_CERTIFICATE_FILE=/etc/ssl/private/self-signed-web.crt \
    SSL_PRIVATE_KEY_FILE=/etc/ssl/private/self-signed-web.key \
    # Same layout as the official images. The leading ":" keeps sury's own
    # conf.d (extensions) and adds ours on top, for both CLI and FPM.
    PHP_INI_DIR=/usr/local/etc/php \
    PHP_INI_SCAN_DIR=":/usr/local/etc/php/conf.d"

# copy our scripts
COPY --chmod=755 src/common/ /
COPY --chmod=755 src/s6/ /
COPY --chmod=755 src/utilities-webservers/ /

# copy s6-overlay from s6-build
COPY --from=s6-build /opt/s6/ /

# copy php-fpm-healthcheck from s6-build
COPY --from=s6-build /usr/local/bin/php-fpm-healthcheck /usr/local/bin/php-fpm-healthcheck

# Copy NGINX and sury repository setup from the repo-config stage
COPY --from=repo-config /usr/share/keyrings/nginx-archive-keyring.gpg /usr/share/keyrings/
COPY --from=repo-config /usr/share/keyrings/debsuryorg-archive-keyring.gpg /usr/share/keyrings/
COPY --from=repo-config /etc/apt/sources.list.d/nginx.list /etc/apt/sources.list.d/
COPY --from=repo-config /etc/apt/sources.list.d/php.list /etc/apt/php.list.pending
COPY --from=repo-config /etc/apt/preferences.d/99nginx /etc/apt/preferences.d/

# install PHP, dependencies, and clean up
RUN \
    # sury is served over HTTPS, so ca-certificates must exist before enabling it
    docker-php-serversideup-dep-install-debian "ca-certificates" && \
    mv /etc/apt/php.list.pending /etc/apt/sources.list.d/php.list && \
    \
    PHP_PACKAGES="php${PHP_VERSION}-redis=${PHP_REDIS_SURY_VERSION}" && \
    for ext in ${DEPENDENCY_PHP_PACKAGES}; do \
        PHP_PACKAGES="${PHP_PACKAGES} php${PHP_VERSION}-${ext}=${PHP_SURY_VERSION}"; \
    done && \
    docker-php-serversideup-dep-install-debian "${DEPENDENCY_PACKAGES_DEBIAN} nginx=${NGINX_VERSION} ${PHP_PACKAGES}" && \
    \
    # Mimic the official image layout so the serversideup scripts work unchanged
    ln -s "/usr/sbin/php-fpm${PHP_VERSION}" /usr/local/sbin/php-fpm && \
    printf "[global]\ninclude=/usr/local/etc/php-fpm.d/*.conf\n" > /usr/local/etc/php-fpm.conf && \
    ln -sf /usr/local/etc/php-fpm.conf "/etc/php/${PHP_VERSION}/fpm/php-fpm.conf" && \
    \
    # Like the official images, run without a php.ini (PHP defaults + our conf.d)
    rm -f "/etc/php/${PHP_VERSION}/cli/php.ini" "/etc/php/${PHP_VERSION}/fpm/php.ini" && \
    \
    # Ensure /var/www/ has the correct permissions
    mkdir -p /var/www/html && \
    chown -R www-data:www-data /var/www && \
    chmod -R 755 /var/www && \
    \
    # Set the image version
    echo "${REPOSITORY_BUILD_VERSION}" > /etc/serversideup-php-version && \
    \
    # Make composer cache directory
    mkdir -p "${COMPOSER_HOME}" && \
    chown -R www-data:www-data "${COMPOSER_HOME}" && \
    \
    # Ensure /var/cache/nginx/ exists
    mkdir -p /var/cache/nginx/ && \
    chown -R www-data:www-data /var/cache/nginx/ && \
    \
    # clear all nginx provided configurations
    rm -rf /etc/nginx/conf.d/*.conf && \
    rm /etc/nginx/nginx.conf

# Copy our nginx configurations
COPY --chmod=755 src/variations/fpm-nginx/etc/ /etc/

# copy our fpm configurations
COPY --chmod=755 src/php-fpm.d/ /

# install composer from Composer's official Docker image
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

RUN docker-php-serversideup-set-file-permissions --owner www-data:www-data --service nginx

# Fix S6 Overlay issues with Big Cloud PaaS (https://github.com/serversideup/docker-php/pull/376#issuecomment-2179262427)
RUN chown -R www-data:www-data /run

USER www-data

EXPOSE 8080 8443

ENTRYPOINT ["docker-php-serversideup-entrypoint"]

# Set stop signal to SIGQUIT for a graceful shutdown instead of S6's preferred SIGTERM (https://github.com/just-containers/s6-overlay/issues/586)
STOPSIGNAL SIGQUIT

WORKDIR ${APP_BASE_DIR}

CMD ["/init"]

HEALTHCHECK --start-period=60s --start-interval=3s --interval=10s --timeout=3s --retries=3 \
    CMD [ "sh", "-c", "curl --insecure --silent --location --show-error --fail http://localhost:${NGINX_HTTP_PORT}${HEALTHCHECK_PATH} || exit 1" ]
