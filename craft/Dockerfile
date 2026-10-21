# Drop-in replacement for the archived ghcr.io/craftcms/image.
# Same base, packages, paths, user, env vars and entrypoint, so projects
# only need to change their FROM line.
FROM ubuntu:24.04

ARG php_version=8.4
# the build fails if the PPA delivers an older PHP than this
ARG php_min_version=8.4.27
ARG userid=3000
ARG groupid=3000

# setup general options for environment variables
ARG PHP_MEMORY_LIMIT_ARG="256M"
ENV PHP_MEMORY_LIMIT=${PHP_MEMORY_LIMIT_ARG}
ARG PHP_MAX_EXECUTION_TIME_ARG="120"
ENV PHP_MAX_EXECUTION_TIME=${PHP_MAX_EXECUTION_TIME_ARG}
ARG PHP_UPLOAD_MAX_FILESIZE_ARG="20M"
ENV PHP_UPLOAD_MAX_FILESIZE=${PHP_UPLOAD_MAX_FILESIZE_ARG}
ARG PHP_MAX_INPUT_VARS_ARG="1000"
ENV PHP_MAX_INPUT_VARS=${PHP_MAX_INPUT_VARS_ARG}
ARG PHP_POST_MAX_SIZE_ARG="8M"
ENV PHP_POST_MAX_SIZE=${PHP_POST_MAX_SIZE_ARG}
ARG PHP_REGISTER_ARGC_ARGV="0"
ENV PHP_REGISTER_ARGC_ARGV=${PHP_REGISTER_ARGC_ARGV}

# setup opcache for environment variables
ARG PHP_OPCACHE_ENABLE_ARG="1"
ARG PHP_OPCACHE_REVALIDATE_FREQ_ARG="0"
ARG PHP_OPCACHE_VALIDATE_TIMESTAMPS_ARG="0"
ARG PHP_OPCACHE_MAX_ACCELERATED_FILES_ARG="10000"
ARG PHP_OPCACHE_MEMORY_CONSUMPTION_ARG="128"
ARG PHP_OPCACHE_MAX_WASTED_PERCENTAGE_ARG="10"
ARG PHP_OPCACHE_INTERNED_STRINGS_BUFFER_ARG="16"
ARG PHP_OPCACHE_FAST_SHUTDOWN_ARG="1"
ENV PHP_OPCACHE_ENABLE=$PHP_OPCACHE_ENABLE_ARG
ENV PHP_OPCACHE_REVALIDATE_FREQ=$PHP_OPCACHE_REVALIDATE_FREQ_ARG
ENV PHP_OPCACHE_VALIDATE_TIMESTAMPS=$PHP_OPCACHE_VALIDATE_TIMESTAMPS_ARG
ENV PHP_OPCACHE_MAX_ACCELERATED_FILES=$PHP_OPCACHE_MAX_ACCELERATED_FILES_ARG
ENV PHP_OPCACHE_MEMORY_CONSUMPTION=$PHP_OPCACHE_MEMORY_CONSUMPTION_ARG
ENV PHP_OPCACHE_MAX_WASTED_PERCENTAGE=$PHP_OPCACHE_MAX_WASTED_PERCENTAGE_ARG
ENV PHP_OPCACHE_INTERNED_STRINGS_BUFFER=$PHP_OPCACHE_INTERNED_STRINGS_BUFFER_ARG
ENV PHP_OPCACHE_FAST_SHUTDOWN=$PHP_OPCACHE_FAST_SHUTDOWN_ARG

# add the application user
RUN groupadd -r -g ${groupid} appgroup \
    && useradd --no-create-home --no-log-init --system --home-dir=/app --uid ${userid} --gid ${groupid} appuser

RUN mkdir -p /app && chown -R appuser:appgroup /app

# the apt lists stay in the image on purpose: project Dockerfiles run
# `apt install` without `apt update`, like with the craftcms image
RUN export DEBIAN_FRONTEND=noninteractive \
    && echo 'Acquire::Retries "10";' > /etc/apt/apt.conf.d/80-retries \
    && apt update -y \
    && apt install -y -q software-properties-common ca-certificates curl \
    && curl -fsSLo /tmp/debsuryorg-archive-keyring.deb https://packages.sury.org/debsuryorg-archive-keyring.deb \
    && dpkg -i /tmp/debsuryorg-archive-keyring.deb \
    && rm /tmp/debsuryorg-archive-keyring.deb \
    && echo "deb [signed-by=/usr/share/keyrings/debsuryorg-archive-keyring.gpg] https://packages.sury.org/php/ noble main" \
        > /etc/apt/sources.list.d/php.list \
    && apt update -y  \
    && apt install -y \
        curl \
        unzip \
        nginx \
        supervisor \
        libheif-plugin-libde265 \
        php${php_version}-bcmath \
        php${php_version}-cli \
        php${php_version}-common \
        php${php_version}-curl \
        php${php_version}-fpm \
        php${php_version}-gd \
        php${php_version}-iconv \
        php${php_version}-intl \
        php${php_version}-mbstring \
        php${php_version}-mysqlnd \
        php${php_version}-opcache \
        php${php_version}-pgsql \
        php${php_version}-redis \
        php${php_version}-soap \
        php${php_version}-xml \
        php${php_version}-zip \
    && apt upgrade -y \
    && apt autoremove -y \
    && apt clean -y

RUN php -r 'exit(version_compare(PHP_VERSION, $argv[1], ">=") ? 0 : 1);' "${php_min_version}" \
    || { echo "PHP $(php -r 'echo PHP_VERSION;') is older than ${php_min_version}" >&2; exit 1; }

# copy the files from the host to the container that we need
COPY etc/supervisord.conf /etc/supervisord.conf
COPY etc/supervisord.d /etc/supervisord.d
COPY etc/php-fpm/php-fpm.conf /etc/php-fpm.conf
COPY etc/php.d/60-craftcms.ini /etc/php/${php_version}/fpm/conf.d/60-craftcms.ini
COPY etc/php.d/60-craftcms.ini /etc/php/${php_version}/cli/conf.d/60-craftcms.ini

# set a friendly path for php-fpm that does not have the version
RUN update-alternatives --install /usr/sbin/php-fpm php-fpm /usr/sbin/php-fpm${php_version} 1

# set the sockets and pid files to be writable by the appuser
RUN mkdir -p /var/run/php && touch /var/run/php/php-fpm.sock && chown -R appuser:appgroup /var/run/php
RUN touch /run/php-fpm.pid && chown -R appuser:appgroup /run/php-fpm.pid
RUN touch /run/supervisord.pid && chown -R appuser:appgroup /run/supervisord.pid

WORKDIR /app

USER appuser

# check the server configuration to make sure it can run Craft CMS
# (a pinned release instead of piping HEAD/check.sh into bash; 5.x is the
# check for Craft 5, 6.x requires PHP 8.5)
ARG server_check_version=5.1.0
ARG server_check_sha256=0eaa1ff47c10c12a98f45d5c53f684a5c40162a004225013289cb519d5ebb3d6
RUN mkdir /tmp/server-check \
    && curl -fsSL -o /tmp/server-check/check.tar.gz \
        https://github.com/craftcms/server-check/releases/download/${server_check_version}/craftcms-server-check.tar.gz \
    && echo "${server_check_sha256}  /tmp/server-check/check.tar.gz" | sha256sum -c - \
    && tar -xzf /tmp/server-check/check.tar.gz -C /tmp/server-check \
    && php /tmp/server-check/checkit.php \
    && rm -rf /tmp/server-check

ENTRYPOINT ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
